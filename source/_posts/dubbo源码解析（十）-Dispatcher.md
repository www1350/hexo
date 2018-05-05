---
title: dubbo源码解析（十） Dispatcher
tags:
  - dubbo
  - rpc
categories:
  - 中间件
  - 源码
abbrlink: f64ed43e
date: 2018-04-23 19:32:39
---

# 线程模型

如果事件处理的逻辑能迅速完成，并且不会发起新的 IO 请求，比如只是在内存中记个标识，则直接在 IO 线程上处理更快，因为减少了线程池调度。

但如果事件处理逻辑较慢，或者需要发起新的 IO 请求，比如需要查询数据库，则必须派发到线程池，否则 IO 线程阻塞，将导致不能接收其它请求。

如果用 IO 线程处理事件，又在事件处理过程中发起新的 IO 请求，比如在连接事件中发起登录请求，会报“可能引发死锁”异常，但不会真死锁。

![image](https://user-images.githubusercontent.com/7789698/39110477-d4f17800-4703-11e8-909b-5a5983a1dc48.png)

因此，需要通过不同的派发策略和不同的线程池配置的组合来应对不同的场景:

```xml
<dubbo:protocol name="dubbo" dispatcher="all" threadpool="fixed" threads="100" />
```

Dispatcher

- `all` 所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等（如果线程池不可用了，就使用共享线程池）。
- `direct` 所有消息都不派发到线程池，全部在 IO 线程上直接执行。
- `message` 只有请求响应消息派发到线程池，其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
- `execution` 连接断开事件请求响应消息派发到线程池，其它心跳等消息，直接在 IO 线程上执行。
- `connection` 在 IO 线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池(如果请求响应消息的线程池不可用了，就使用共享线程池)。

ThreadPool

- `fixed` 固定大小线程池，启动时建立线程，不关闭，一直持有。(缺省)
- `cached` 缓存线程池，空闲一分钟自动删除，需要时重建。
- `limited` 可伸缩线程池，但池中的线程数只会增长不会收缩。只增长不收缩的目的是为了避免收缩时突然来了大流量引起的性能问题。
- `eager` 优先创建`Worker`线程池。在任务数量大于`corePoolSize`但是小于`maximumPoolSize`时，优先创建`Worker`来处理任务。当任务数量大于`maximumPoolSize`时，将任务放入阻塞队列中。阻塞队列充满时抛出`RejectedExecutionException`。(相比于`cached`:`cached`在任务数量超过`maximumPoolSize`时直接抛出异常而不是将任务放入阻塞队列)

上面这个eager是小伙伴时无两最近提交的，可以忽略。

<!-- more -->

### 源码

Dispatcher接口

```java
@SPI(AllDispatcher.NAME)
public interface Dispatcher {

    /**
     * dispatch the message to threadpool.
     *
     * @param handler
     * @param url
     * @return channel handler
     */
    @Adaptive({Constants.DISPATCHER_KEY, "dispather", "channel.handler"})
    // The last two parameters are reserved for compatibility with the old configuration
    ChannelHandler dispatch(ChannelHandler handler, URL url);

}
```



```properties
all=com.alibaba.dubbo.remoting.transport.dispatcher.all.AllDispatcher
direct=com.alibaba.dubbo.remoting.transport.dispatcher.direct.DirectDispatcher
message=com.alibaba.dubbo.remoting.transport.dispatcher.message.MessageOnlyDispatcher
execution=com.alibaba.dubbo.remoting.transport.dispatcher.execution.ExecutionDispatcher
connection=com.alibaba.dubbo.remoting.transport.dispatcher.connection.ConnectionOrderedDispatcher
```

#### DirectDispatcher

```java
public class DirectDispatcher implements Dispatcher {
    public static final String NAME = "direct";
    public ChannelHandler dispatch(ChannelHandler handler, URL url) {
        return handler;
    }
}
```



![image](https://user-images.githubusercontent.com/7789698/39111930-75d21db0-4709-11e8-8dfd-78b21a3534e2.png)

#### WrappedChannelHandler

```java
//共享线程池
protected static final ExecutorService SHARED_EXECUTOR = Executors.newCachedThreadPool(new NamedThreadFactory("DubboSharedHandler", true));

protected final ExecutorService executor;

protected final ChannelHandler handler;

protected final URL url;
```

```java
public WrappedChannelHandler(ChannelHandler handler, URL url) {
    this.handler = handler;
    this.url = url;
    executor = (ExecutorService) ExtensionLoader.getExtensionLoader(ThreadPool.class).getAdaptiveExtension().getExecutor(url);

    String componentKey = Constants.EXECUTOR_SERVICE_COMPONENT_KEY;
    if (Constants.CONSUMER_SIDE.equalsIgnoreCase(url.getParameter(Constants.SIDE_KEY))) {
        componentKey = Constants.CONSUMER_SIDE;
    }
    DataStore dataStore = ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension();
    dataStore.put(componentKey, Integer.toString(url.getPort()), executor);
}
```



#### AllDispatcher

所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等。

- 如果线程池不可用了，就使用共享线程池

```java
public class AllDispatcher implements Dispatcher {
    public static final String NAME = "all";
    public ChannelHandler dispatch(ChannelHandler handler, URL url) {
        return new AllChannelHandler(handler, url);
    }
}
```

```java
public class AllChannelHandler extends WrappedChannelHandler {
    public AllChannelHandler(ChannelHandler handler, URL url) {
        super(handler, url);
    }
    
    public void connected(Channel channel) throws RemotingException {
        //连接进线程池
        ExecutorService cexecutor = getExecutorService();
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.CONNECTED));
        } catch (Throwable t) {
            throw new ExecutionException("connect event", channel, getClass() + " error when process connected event .", t);
        }
    }

    public void disconnected(Channel channel) throws RemotingException {
        //断开进线程池
        ExecutorService cexecutor = getExecutorService();
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.DISCONNECTED));
        } catch (Throwable t) {
            throw new ExecutionException("disconnect event", channel, getClass() + " error when process disconnected event .", t);
        }
    }
//请求响应进线程池
    public void received(Channel channel, Object message) throws RemotingException {
        ExecutorService cexecutor = getExecutorService();
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
            //TODO A temporary solution to the problem that the exception information can not be sent to the opposite end after the thread pool is full. Need a refactoring
            //fix The thread pool is full, refuses to call, does not return, and causes the consumer to wait for time out
           if(message instanceof Request && t instanceof RejectedExecutionException){
              Request request = (Request)message;
              if(request.isTwoWay()){
                 String msg = "Server side(" + url.getIp() + "," + url.getPort() + ") threadpool is exhausted ,detail msg:" + t.getMessage();
                 Response response = new Response(request.getId(), request.getVersion());
                 response.setStatus(Response.SERVER_THREADPOOL_EXHAUSTED_ERROR);
                 response.setErrorMessage(msg);
                 channel.send(response);
                 return;
              }
           }
            throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
        }
    }

    public void caught(Channel channel, Throwable exception) throws RemotingException {
        ExecutorService cexecutor = getExecutorService();
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.CAUGHT, exception));
        } catch (Throwable t) {
            throw new ExecutionException("caught event", channel, getClass() + " error when process caught event .", t);
        }
    }
//如果线程池是空的或者已经被shutdown了
    private ExecutorService getExecutorService() {
        ExecutorService cexecutor = executor;
        if (cexecutor == null || cexecutor.isShutdown()) {
            cexecutor = SHARED_EXECUTOR;
        }
        return cexecutor;
    }
}
```
#### MessageOnlyChannelHandler

只有请求响应消息派发到线程池，其它连接断开事件，心跳等消息，直接在 IO 线程上执行。

```java
public class MessageOnlyChannelHandler extends WrappedChannelHandler {
    public MessageOnlyChannelHandler(ChannelHandler handler, URL url) {
        super(handler, url);
    }
//请求响应消息进线程池
    public void received(Channel channel, Object message) throws RemotingException {
        ExecutorService cexecutor = executor;
        if (cexecutor == null || cexecutor.isShutdown()) {
            cexecutor = SHARED_EXECUTOR;
        }
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
            throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
        }
    }

}
```

#### ExecutionChannelHandler

只请求消息派发到线程池，不含响应，响应和其它连接断开事件，心跳等消息，直接在 IO 线程上执行。

与AllChannelHandler不同之处在于，若创建的线程池ExecutorService不可用（null或者被shutdown了），AllChannelHandler将使用共享线程池，而ExecutionChannelHandler只有报错了。

- 都是用线程池解决

```java
public class ExecutionChannelHandler extends WrappedChannelHandler {
    public ExecutionChannelHandler(ChannelHandler handler, URL url) {
        super(handler, url);
    }
//连接进线程池
    public void connected(Channel channel) throws RemotingException {
        executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.CONNECTED));
    }
//断开进线程池
    public void disconnected(Channel channel) throws RemotingException {
        executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.DISCONNECTED));
    }
//请求响应
    public void received(Channel channel, Object message) throws RemotingException {
       try {
            executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
            //TODO A temporary solution to the problem that the exception information can not be sent to the opposite end after the thread pool is full. Need a refactoring
            //fix The thread pool is full, refuses to call, does not return, and causes the consumer to wait for time out
           if(message instanceof Request &&
                 t instanceof RejectedExecutionException){
              Request request = (Request)message;
              if(request.isTwoWay()){
                 String msg = "Server side("+url.getIp()+","+url.getPort()+") threadpool is exhausted ,detail msg:"+t.getMessage();
                 Response response = new Response(request.getId(), request.getVersion());
                 response.setStatus(Response.SERVER_THREADPOOL_EXHAUSTED_ERROR);
                 response.setErrorMessage(msg);
                 channel.send(response);
                 return;
              }
           }
            throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
        }
    }
//异常放入线程池
    public void caught(Channel channel, Throwable exception) throws RemotingException {
        executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.CAUGHT, exception));
    }
}
```

#### ConnectionOrderedChannelHandler

在 IO 线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池。

- 如果请求响应消息的线程池不可用了，就使用共享线程池。
- 连接断开事件放入队列有序逐个执行

```java
public class ConnectionOrderedChannelHandler extends WrappedChannelHandler {
    protected final ThreadPoolExecutor connectionExecutor;
    private final int queuewarninglimit;

    public ConnectionOrderedChannelHandler(ChannelHandler handler, URL url) {
        super(handler, url);
        String threadName = url.getParameter(Constants.THREAD_NAME_KEY, Constants.DEFAULT_THREAD_NAME);
        connectionExecutor = new ThreadPoolExecutor(1, 1,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<Runnable>(url.getPositiveParameter(Constants.CONNECT_QUEUE_CAPACITY, Integer.MAX_VALUE)),
                new NamedThreadFactory(threadName, true),
                new AbortPolicyWithReport(threadName, url)
        );  // FIXME There's no place to release connectionExecutor!
        queuewarninglimit = url.getParameter(Constants.CONNECT_QUEUE_WARNING_SIZE, Constants.DEFAULT_CONNECT_QUEUE_WARNING_SIZE);
    }
//连接放入队列
    public void connected(Channel channel) throws RemotingException {
        try {
            checkQueueLength();
            connectionExecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.CONNECTED));
        } catch (Throwable t) {
            throw new ExecutionException("connect event", channel, getClass() + " error when process connected event .", t);
        }
    }
//断开放入队列
    public void disconnected(Channel channel) throws RemotingException {
        try {
            checkQueueLength();
            connectionExecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.DISCONNECTED));
        } catch (Throwable t) {
            throw new ExecutionException("disconnected event", channel, getClass() + " error when process disconnected event .", t);
        }
    }
//请求响应消息派发线程池
//若创建的线程池ExecutorService不可用（null或者被shutdown了），将使用共享线程池
    public void received(Channel channel, Object message) throws RemotingException {
        ExecutorService cexecutor = executor;
        if (cexecutor == null || cexecutor.isShutdown()) {
            cexecutor = SHARED_EXECUTOR;
        }
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
            //fix, reject exception can not be sent to consumer because thread pool is full, resulting in consumers waiting till timeout.
            if (message instanceof Request && t instanceof RejectedExecutionException) {
                Request request = (Request) message;
                if (request.isTwoWay()) {
                    String msg = "Server side(" + url.getIp() + "," + url.getPort() + ") threadpool is exhausted ,detail msg:" + t.getMessage();
                    Response response = new Response(request.getId(), request.getVersion());
                    response.setStatus(Response.SERVER_THREADPOOL_EXHAUSTED_ERROR);
                    response.setErrorMessage(msg);
                    channel.send(response);
                    return;
                }
            }
            throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
        }
    }
//异常放入线程池
    public void caught(Channel channel, Throwable exception) throws RemotingException {
        ExecutorService cexecutor = executor;
        if (cexecutor == null || cexecutor.isShutdown()) {
            cexecutor = SHARED_EXECUTOR;
        }
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.CAUGHT, exception));
        } catch (Throwable t) {
            throw new ExecutionException("caught event", channel, getClass() + " error when process caught event .", t);
        }
    }

    private void checkQueueLength() {
        if (connectionExecutor.getQueue().size() > queuewarninglimit) {
            logger.warn(new IllegalThreadStateException("connectionordered channel handler `queue size: " + connectionExecutor.getQueue().size() + " exceed the warning limit number :" + queuewarninglimit));
        }
    }
}
```