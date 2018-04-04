---
title: dubbo源码解析（六） DubboInvoker
abbrlink: a645d07f
date: 2018-04-03 22:44:43
tags: [dubbo,rpc]
categories: [中间件,源码]
---

当service被调用时，通过代理最后调用的是FailoverClusterInvoker的invoke

![sequencediagram111](https://user-images.githubusercontent.com/7789698/38227884-f84568c4-3732-11e8-991b-26baf6d6f4d9.png)

## proxy 服务代理层

```java
public class JavassistProxyFactory extends AbstractProxyFactory {
    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }
}
```

真正的service通过Javassist代理被代理成代理子类

InvokerInvocationHandler

```java
public class InvokerInvocationHandler implements InvocationHandler {
//MockClusterInvoker
    private final Invoker<?> invoker;

    public InvokerInvocationHandler(Invoker<?> handler) {
        this.invoker = handler;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        if ("toString".equals(methodName) && parameterTypes.length == 0) {
            return invoker.toString();
        }
        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
            return invoker.hashCode();
        }
        if ("equals".equals(methodName) && parameterTypes.length == 1) {
            return invoker.equals(args[0]);
        }
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }

}
```

FailoverClusterInvoker失败自动切换

```java
public Result invoke(final Invocation invocation) throws RpcException {
    checkWhetherDestroyed();
    LoadBalance loadbalance = null;
    //列出所有的invoker
    List<Invoker<T>> invokers = list(invocation);
    if (invokers != null && !invokers.isEmpty()) {
        //获取相应的LoadBalance
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                .getMethodParameter(invocation.getMethodName(), Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
    }
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    return doInvoke(invocation, invokers, loadbalance);
}

    protected List<Invoker<T>> list(Invocation invocation) throws RpcException {
        //最后通过methodInvokerMap取到所有RegistryDirectory$InvokerDelegate(DubboInvoker)
        List<Invoker<T>> invokers = directory.list(invocation);
        return invokers;
    }
```



```java
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    List<Invoker<T>> copyinvokers = invokers;
    checkInvokers(copyinvokers, invocation);
    //获取retries次数
    int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
    if (len <= 0) {
        len = 1;
    }
    // retry loop.
    RpcException le = null; // last exception.
    List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
    Set<String> providers = new HashSet<String>(len);
    //重试
    for (int i = 0; i < len; i++) {
        //Reselect before retry to avoid a change of candidate `invokers`.
        //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
        if (i > 0) {
            checkWhetherDestroyed();
            copyinvokers = list(invocation);
            // check again
            checkInvokers(copyinvokers, invocation);
        }
        //根据均衡算法选取Invoker，调用FailoverClusterInvoker的doSelect
        //RegistryDirectory$InvokerDelegate(DubboInvoker)
        Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
        invoked.add(invoker);
        RpcContext.getContext().setInvokers((List) invoked);
        try {
            //调用FailoverClusterInvoker 的invoke
             //
            Result result = invoker.invoke(invocation);
            if (le != null && logger.isWarnEnabled()) {
                logger.warn("Although retry the method " + invocation.getMethodName()
                        + " in the service " + getInterface().getName()
                        + " was successful by the provider " + invoker.getUrl().getAddress()
                        + ", but there have been failed providers " + providers
                        + " (" + providers.size() + "/" + copyinvokers.size()
                        + ") from the registry " + directory.getUrl().getAddress()
                        + " on the consumer " + NetUtils.getLocalHost()
                        + " using the dubbo version " + Version.getVersion() + ". Last error is: "
                        + le.getMessage(), le);
            }
            //成功或者抛出异常就返回
            //RpcException异常则重试
            return result;
        } catch (RpcException e) {
            if (e.isBiz()) { // biz exception.
                throw e;
            }
            le = e;
        } catch (Throwable e) {
            le = new RpcException(e.getMessage(), e);
        } finally {
            providers.add(invoker.getUrl().getAddress());
        }
    }
    throw new RpcException(le != null ? le.getCode() : 0, "Failed to invoke the method "
            + invocation.getMethodName() + " in the service " + getInterface().getName()
            + ". Tried " + len + " times of the providers " + providers
            + " (" + providers.size() + "/" + copyinvokers.size()
            + ") from the registry " + directory.getUrl().getAddress()
            + " on the consumer " + NetUtils.getLocalHost() + " using the dubbo version "
            + Version.getVersion() + ". Last error is: "
            + (le != null ? le.getMessage() : ""), le != null && le.getCause() != null ? le.getCause() : le);
}
```

RegistryDirectory$InvokerDelegate

```java
private static class InvokerDelegate<T> extends InvokerWrapper<T> {
    private URL providerUrl;

    //这里其实是DubboProtocol#refer的结果
    //DubboInvoker
    public InvokerDelegate(Invoker<T> invoker, URL url, URL providerUrl) {
        super(invoker, url);
        this.providerUrl = providerUrl;
    }

    public URL getProviderUrl() {
        return providerUrl;
    }
}
```



DubboInvoker无疑是最核心的所在，其他的不过是负载、失效、mock等代理封装

![sequencediagram12333](https://user-images.githubusercontent.com/7789698/38234393-5b63e418-3750-11e8-9890-b353220bd4eb.png)

### DubboInvoker

```java
protected Result doInvoke(final Invocation invocation) throws Throwable {
    RpcInvocation inv = (RpcInvocation) invocation;
    final String methodName = RpcUtils.getMethodName(invocation);
    inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
    inv.setAttachment(Constants.VERSION_KEY, version);

    ExchangeClient currentClient;
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        currentClient = clients[index.getAndIncrement() % clients.length];
    }
    try {
        boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        if (isOneway) {
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            currentClient.send(inv, isSent);
            RpcContext.getContext().setFuture(null);
            return new RpcResult();
        } else if (isAsync) {
            ResponseFuture future = currentClient.request(inv, timeout);
            RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
            return new RpcResult();
        } else {
            RpcContext.getContext().setFuture(null);
            return (Result) currentClient.request(inv, timeout).get();
        }
    } catch (TimeoutException e) {
        throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    } catch (RemotingException e) {
        throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

HeaderExchangeChannel

```java
public void send(Object message, boolean sent) throws RemotingException {
    if (closed) {
        throw new RemotingException(this.getLocalAddress(), null, "Failed to send message " + message + ", cause: The channel " + this + " is closed!");
    }
    if (message instanceof Request
            || message instanceof Response
            || message instanceof String) {
        channel.send(message, sent);
    } else {
        Request request = new Request();
        request.setVersion("2.0.0");
        request.setTwoWay(false);
        request.setData(message);
        channel.send(request, sent);
    }
}
```

NettyChannel

```java
public void send(Object message, boolean sent) throws RemotingException {
    super.send(message, sent);

    boolean success = true;
    int timeout = 0;
    try {
        ChannelFuture future = channel.write(message);
        if (sent) {
            timeout = getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
            success = future.await(timeout);
        }
        Throwable cause = future.getCause();
        if (cause != null) {
            throw cause;
        }
    } catch (Throwable e) {
        throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
    }

    if (!success) {
        throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress()
                + "in timeout(" + timeout + "ms) limit");
    }
}
```

FailoverClusterInvoker如何负载的呢？

```java
private Invoker<T> doSelect(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
    if (invokers == null || invokers.isEmpty())
        return null;
    //只有一个直接返回
    if (invokers.size() == 1)
        return invokers.get(0);
    // If we only have two invokers, use round-robin instead.
    if (invokers.size() == 2 && selected != null && !selected.isEmpty()) {
        return selected.get(0) == invokers.get(0) ? invokers.get(1) : invokers.get(0);
    }
    if (loadbalance == null) {
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
    }
    Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);

    //If the `invoker` is in the  `selected` or invoker is unavailable && availablecheck is true, reselect.
    //失效了重新选取
    if ((selected != null && selected.contains(invoker))
            || (!invoker.isAvailable() && getUrl() != null && availablecheck)) {
        try {
            Invoker<T> rinvoker = reselect(loadbalance, invocation, invokers, selected, availablecheck);
            if (rinvoker != null) {
                invoker = rinvoker;
            } else {
                //Check the index of current selected invoker, if it's not the last one, choose the one at index+1.
                int index = invokers.indexOf(invoker);
                try {
                    //Avoid collision
                    invoker = index < invokers.size() - 1 ? invokers.get(index + 1) : invoker;
                } catch (Exception e) {
                    logger.warn(e.getMessage() + " may because invokers list dynamic change, ignore.", e);
                }
            }
        } catch (Throwable t) {
            logger.error("cluster reselect fail reason is :" + t.getMessage() + " if can not solve, you can set cluster.availablecheck=false in url", t);
        }
    }
    return invoker;
}
```



RandomLoadBalance

```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    int length = invokers.size(); // Number of invokers
    int totalWeight = 0; // The sum of weights
    boolean sameWeight = true; // Every invoker has the same weight?
    for (int i = 0; i < length; i++) {
        int weight = getWeight(invokers.get(i), invocation);
        totalWeight += weight; // Sum
        if (sameWeight && i > 0
                && weight != getWeight(invokers.get(i - 1), invocation)) {
            sameWeight = false;
        }
    }
    if (totalWeight > 0 && !sameWeight) {
        // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
        int offset = random.nextInt(totalWeight);
        // Return a invoker based on the random value.
        for (int i = 0; i < length; i++) {
            offset -= getWeight(invokers.get(i), invocation);
            if (offset < 0) {
                return invokers.get(i);
            }
        }
    }
    // If all invokers have the same weight value or totalWeight=0, return evenly.
    return invokers.get(random.nextInt(length));
}
```
