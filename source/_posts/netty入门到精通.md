---
title: netty入门到精通
abbrlink: d38fb5d0
date: 2018-04-03 22:39:09
tags:
categories:
---

# netty3.x

**入门**

``` java
public void run() {
    // Configure the server.
    ServerBootstrap bootstrap = new ServerBootstrap(
            new NioServerSocketChannelFactory(
                    Executors.newCachedThreadPool(),
                    Executors.newCachedThreadPool()));

    // Set up the pipeline factory.
    bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
        public ChannelPipeline getPipeline() throws Exception {
            return Channels.pipeline(new EchoServerHandler());
        }
    });
         bootstrap.setOption("child.tcpNoDelay", true);
        bootstrap.setOption("child.keepAlive", true);
....
    // Bind and start to accept incoming connections.
    bootstrap.bind(new InetSocketAddress(port));
}


```

业务代码

``` java
public class EchoServerHandler extends SimpleChannelUpstreamHandler {

    @Override
    public void messageReceived(
            ChannelHandlerContext ctx, MessageEvent e) {
        // Send back the received message to the remote peer.
        e.getChannel().write(e.getMessage());
    }
}
```

**精通**
手册：http://netty.io/3.7/guide/
# netty4.x

**入门**
服务端

``` java
     /**
     * 服务端监听的端口地址
     */
    private static final int portNumber = 7878;

  EventLoopGroup bossGroup = new NioEventLoopGroup();
    EventLoopGroup workerGroup = new NioEventLoopGroup();
    try {
        ServerBootstrap b = new ServerBootstrap();
        b.group(bossGroup, workerGroup);
        b.channel(NioServerSocketChannel.class);
        b.childHandler(new HelloServerInitializer());

        // 服务器绑定端口监听
        ChannelFuture f = b.bind(portNumber).sync();

        System.out.println("init server");

        // 监听服务器关闭监听
        f.channel().closeFuture().sync();

        // 可以简写为
        /* b.bind(portNumber).sync().channel().closeFuture().sync(); */
    } finally {
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }
```

``` java
public class HelloServerInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();

        // 以("\n")为结尾分割的 解码器
        pipeline.addLast("framer", new DelimiterBasedFrameDecoder(8192, Delimiters.lineDelimiter()));

        // 字符串解码 和 编码
        pipeline.addLast("decoder", new StringDecoder());
        pipeline.addLast("encoder", new StringEncoder());

        // 自己的逻辑Handler
        pipeline.addLast("handler", new HelloServerHandler());
    }
}
```

``` java
public class HelloServerHandler extends SimpleChannelInboundHandler<String> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
        // 收到消息直接打印输出
        System.out.println(ctx.channel().remoteAddress() + " Say : " + msg);

        // 返回客户端消息 - 我已经接收到了你的消息
        ctx.writeAndFlush("我Received your message "+msg+"!\n");
    }

    /*
     * 
     * 覆盖 channelActive 方法 在channel被启用的时候触发 (在建立连接的时候)
     * 
     * channelActive 和 channelInActive 在后面的内容中讲述，这里先不做详细的描述
     * */
/*    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {

        System.out.println("RamoteAddress : " + ctx.channel().remoteAddress() + " active !");

        ctx.writeAndFlush( "Welcome to " + InetAddress.getLocalHost().getHostName() + " service!\n");

        super.channelActive(ctx);
    }*/
}
```

客户端

``` java
    public static String host = "127.0.0.1";
    public static int port = 7878;

        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
            .channel(NioSocketChannel.class)
            .handler(new HelloClientInitializer());

            // 连接服务端
            Channel ch = b.connect(host, port).sync().channel();
            System.out.println("init client");
            // 控制台输入
            BufferedReader in = new BufferedReader(new InputStreamReader(System.in));
            for (;;) {
                String line = in.readLine();
                if (line == null) {
                    continue;
                }
                /*
                 * 向服务端发送在控制台输入的文本 并用"\r\n"结尾
                 * 之所以用\r\n结尾 是因为我们在handler中添加了 DelimiterBasedFrameDecoder 帧解码。
                 * 这个解码器是一个根据\n符号位分隔符的解码器。所以每条消息的最后必须加上\n否则无法识别和解码
                 * */
                ch.writeAndFlush(line + "\r\n");
            }
        } finally {
            // The connection is closed automatically on shutdown.
            group.shutdownGracefully();
        }

```

``` java
public class HelloClientHandler extends SimpleChannelInboundHandler<String>{

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
        // 收到消息直接打印输出
        System.out.println(ctx.channel().remoteAddress() + " Say : " + msg);

    }
}
```

``` java
public class HelloClientInitializer extends ChannelInitializer<SocketChannel>{

    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
          ChannelPipeline pipeline = ch.pipeline();

            // 以("\n")为结尾分割的 解码器
            pipeline.addLast("framer", new DelimiterBasedFrameDecoder(8192, Delimiters.lineDelimiter()));

            // 字符串解码 和 编码
            pipeline.addLast("decoder", new StringDecoder());
            pipeline.addLast("encoder", new StringEncoder());

            // 自己的逻辑Handler
            pipeline.addLast("handler", new HelloClientHandler());
    }

}
```

- 一个EventLoopGroup包含一个或多个EventLoop
- 一个EventLoop在它生命周期只和一个Thread绑定
- 所有EventLoop处理的I/O事件都将在专有的Thread上处理
- 一个channel在生命周期只注册于一个EventLoop
- 一个EventLoop可能会分配给一个或多个channel

# Channel
- 线程安全
- write:数据写到远程结点。数据传给channelpipeline排队直到被冲刷
- flush：已写数据冲刷到传输底层socket
- 内置的传输：
1. NIO／io.netty.channel.socket.nio／基于选择器
2. Epoll／io.netty.channel.epoll／JNI驱动的epoll和非阻塞IO
3. OIO／io.netty.channel.socket.oio／java.net基础／阻塞流
4. Local／io.netty.channel.local／VM内部通过管道进行通信／本地传输
5. Embedded／io.netty.channel.embedded／Embedded，ChannelHandler不需经过网络

## Channel生命周期
- ChannelUnregistered  channel已创建，但还没注册到EventLoop
- ChannelRegistered 注册到EventLoop
- ChannelActive channel处于活动状态（已连接到远程结点），可以接收和发送数据
- ChannelInactive 没有连接到远程结点
  ![image](https://user-images.githubusercontent.com/7789698/33238160-3fe36bf2-d2c2-11e7-8ecb-cd339369a28e.png)


## ChannelHandler生命周期
- handlerAdded   当ChannelHandler被添加到一个ChannelPipeline时被调用
- handlerRemoved   当ChannelHandler从一个ChannelPipeline中移除时被调用
- exceptionCaught  处理过程中ChannelPipeline中发生错误时被调用

### ChannelInboundHandler——处理输入数据和所有类型的状态变化
方法：
![image](https://user-images.githubusercontent.com/7789698/33238166-663b09d6-d2c2-11e7-83a9-ad54630d7866.png)


| 类型                      | 描述                                                         |
| ------------------------- | ------------------------------------------------------------ |
| channelRegistered         | 当一个Channel注册到EventLoop上，可以处理I/O时被调用          |
| channelUnregistered       | 当一个Channel从它的EventLoop上解除注册，不再处理I/O时被调用  |
| channelActive             | 当Channel变成活跃状态时被调用；Channel是连接/绑定、就绪的    |
| channelInactive           | 当Channel离开活跃状态，不再连接到某个远端时被调用            |
| channelReadComplete       | 当Channel上的某个读操作完成时被调用                          |
| channelRead               | 当从Channel中读数据时被调用                                  |
| channelWritabilityChanged | 当Channel的可写状态改变时被调用。通过这个方法，用户可以确保写操作不会进行地太快（避免OutOfMemoryError）或者当Channel又变成可写时继续写操作。Channel类的isWritable()方法可以用来检查Channel的可写状态。可写性的阈值可以通过Channel.config().setWriteHighWaterMark()和Channel.config().setWriteLowWaterMark()来设定。 |
| userEventTriggered        | 因某个POJO穿过ChannelPipeline引发ChannelnboundHandler.fireUserEventTriggered()时被调用 |


当一个ChannelInboundHandler实现类重写channelRead()方法时，它要负责释放ByteBuf相关的内存
```
public class DiscardHandler extends ChannelInboundHandlerAdapter {  
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        //手动释放消息  
        ReferenceCountUtil.release(msg);  
    }  
}  
```
一个更简单的替代方法就是用SimpleChannelInboundHandler
```
public class SimpleDiscardHandler extends SimpleChannelInboundHandler<Object> {  
    @Override  
    protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {  
        //不需要手动释放  
    }  
}  
```
```
public abstract class SimpleChannelInboundHandler<I> extends ChannelInboundHandlerAdapter {
...
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        boolean release = true;
        try {
            if (acceptInboundMessage(msg)) {
                @SuppressWarnings("unchecked")
                I imsg = (I) msg;
                channelRead0(ctx, imsg);
            } else {
                release = false;
                ctx.fireChannelRead(msg);
            }
        } finally {
            if (autoRelease && release) {
                ReferenceCountUtil.release(msg);
            }
        }
    }

 protected abstract void channelRead0(ChannelHandlerContext ctx, I msg) throws Exception;
}
...

```

### ChannelOutboundHandler——处理输出数据，可以拦截所有操作

| 类型                                                         | 描述                                 |
| ------------------------------------------------------------ | ------------------------------------ |
| bind(ChannelHandlerContext,SocketAddress,ChannelPromise)     | 请求绑定Channel到一个本地地址        |
| connect(ChannelHandlerContext, SocketAddress,SocketAddress,ChannelPromise) | 请求连接Channel到远端                |
| disconnect(ChannelHandlerContext, ChannelPromise)            | 请求从远端断开Channel                |
| close(ChannelHandlerContext,ChannelPromise)                  | 请求关闭Channel                      |
| deregister(ChannelHandlerContext, ChannelPromise)            | 请求Channel从它的EventLoop上解除注册 |
| read(ChannelHandlerContext)                                  | 请求从Channel中读更多的数据          |
| flush(ChannelHandlerContext)                                 | 请求通过Channel刷队列数据到远端      |
| write(ChannelHandlerContext,Object, ChannelPromise)          | 请求通过Channel写数据到远端          |


>  CHANNELPROMISE VS. CHANNELFUTURE
>  ChannelOutboundHandler的大部分方法都用了一个ChannelPromise输入参数，用于当操作完成时收到通知。ChannelPromise是ChannelFuture的子接口，定义了可写的方法，比如setSuccess()，或者setFailure()，而ChannelFuture则是不可变对象。

### ChannelHandler适配器类
![image](https://user-images.githubusercontent.com/7789698/33238280-80455306-d2c5-11e7-9b35-5970d907283b.png)

## 资源管理
无论何时你对数据操作ChannelInboundHandler.channelRead()或者ChannelOutboundHandler.write()，你需要确保没有资源泄露。也许你还记得上一章我们提到过，Netty采用引用计数来处理ByteBuf池。所以，在你用完一个ByteBuf后，调整引用计数的值是很重要的。

为了帮助你诊断潜在的问题， Netty提供了ResourceLeakDetector类，它通过采样应用程序1%的buffer分配来检查是否有内存泄露。这个过程的开销是很小的。

如果泄露被检测到，会产生类似下面这样的日志消息：

> LEAK: ByteBuf.release() was not called before it's garbage-collected. Enable
> advanced leak reporting to find out where the leak occurred. To enable
> advanced leak reporting, specify the JVM option
> '-Dio.netty.leakDetectionLevel=ADVANCED' or call
> ResourceLeakDetector.setLevel().

| 级别     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| DISABLED | 关闭内存泄露检测。 只有在大量测试后，才能用这个级别          |
| SIMPLE   | 报告默认的1%采样率中发现的任何泄露。这是默认的级别，在大部分情况下适用 |
| ADVANCED | 报告发现的泄露和消息的位置。使用默认的采样率。               |
| PARANOID | 类似ADVANCED级别，但是每个消息的获取都被检测采样。这对性能有很大影响，只能在调试阶段使用。 |

用上表中的某个值来配置下面这个Java系统属性，就可以设定内存泄露检测级别：

`java -Dio.netty.leakDetectionLevel=ADVANCED`

如果你设定这个JVM选项然后重启你的应用，你会看到应用中泄露buffer的最新位置。下面是一个单元测试产生的典型的内存泄露报告：

> Running io.netty.handler.codec.xml.XmlFrameDecoderTest
> 15:03:36.886 [main] ERROR io.netty.util.ResourceLeakDetector - LEAK:
> ByteBuf.release() was not called before it's garbage-collected.
> Recent access records: 1
> #1: io.netty.buffer.AdvancedLeakAwareByteBuf.toString(
> AdvancedLeakAwareByteBuf.java:697)
> io.netty.handler.codec.xml.XmlFrameDecoderTest.testDecodeWithXml(
> XmlFrameDecoderTest.java:157)
> io.netty.handler.codec.xml.XmlFrameDecoderTest.testDecodeWithTwoMessages(
> XmlFrameDecoderTest.java:133)


在你实现ChannelInboundHandler.channelRead()或者ChannelOutboundHandler.write()时，你怎样用这个诊断工具来防止内存泄露呢？让我们来看下ChannelRead()操作“消费(consume)”输入数据这个情况：就是说，当前handler没有通过ChannelContext.fireChannelRead()把消息传递到下一个ChannelInboundHandler。下面的代码说明了如何释放这条消息占用的内存。
```
public class DiscardInboundHandler extends ChannelInboundHandlerAdapter {  
    @Override  
   public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        ReferenceCountUtil.release(msg);  
    }  
}  

```



```
public class DiscardOutboundHandler extends ChannelOutboundHandlerAdapter {  
    @Override  
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {  
        ReferenceCountUtil.release(msg);  
        promise.setSuccess();  
    }  
}  
```
重要的是，不仅要释放资源，而且要通知ChannelPromise，否则会出现某个ChannelFutureListener没有被通知到消息已经被处理的情况。

总之，如果一个消息被“消费”或者丢弃，没有送到ChannelPipeline中的下一个ChannelOutboundHandler，用户就要负责调用ReferenceCountUtil.release()。如果消息到达了真正的传输层，在它被写到socket中或者Channel关闭时，会被自动释放（这种情况下用户就不用管了）。

## ChannelPipeline接口
https://segmentfault.com/a/1190000007308934
如果你把一个ChannelPipeline看成是一串ChannelHandler实例，拦截穿过Channel的输入输出event，那么就很容易明白这些ChannelHandler的交互是如何构成了一个应用程序数据和事件处理逻辑的核心。

每个新创建的Channel都会分配一个新的ChannelPipeline。这个关系是恒定的；Channel不可以换别ChannelPipeline，也不可以解除掉当前分配的ChannelPipeline。在Netty组件的整个生命周期中这个关系是固定的，不需要开发者采取什么操作。

根据来源，一个event可以被一个ChannelInboundHandler或者ChannelOutboundHandler处理。接下来，通过调用ChannelHandlerContext的方法，它会被转发到下一个同类型的handler。

![image](https://user-images.githubusercontent.com/7789698/33238425-90b989a2-d2c8-11e7-8b81-ae8eb80ed1ae.png)



| 方法名                  | 描述                                                         |
| ----------------------- | ------------------------------------------------------------ |
| fireChannelRegistered   | 调用ChannelPipeline中下一个ChannelInboundHandler的channelRegistered(ChannelHandlerContext) |
| fireChannelUnregistered | 调用ChannelPipeline中下一个ChannelInboundHandler的channelUnRegistered(ChannelHandlerContext) |
| fireChannelActive       | 调用ChannelPipeline中下一个ChannelInboundHandler的channelActive(ChannelHandlerContext) |
| fireChannelInactive     | 调用ChannelPipeline中下一个ChannelInboundHandler的channelInactive(ChannelHandlerContext) |
| fireExceptionCaught     | 调用ChannelPipeline中下一个ChanneHandler的exceptionCaught(ChannelHandlerContext,Throwable) |
| fireUserEventTriggered  | 调用ChannelPipeline中下一个ChannelInboundHandler的userEventTriggered(ChannelHandlerContext, Object) |
| fireChannelRead         | 调用ChannelPipeline中下一个ChannelInboundHandler的channelRead(ChannelHandlerContext, Object msg) |
| fireChannelReadComplete | 调用ChannelPipeline中下一个ChannelStateHandler的channelReadComplete(ChannelHandlerContext) |


| 方法名        | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| bind          | 绑定Channel到一个本地地址。这会调用ChannelPipeline中下一个ChannelOutboundHandler的bind(ChannelHandlerContext, SocketAddress, ChannelPromise) |
| connect       | 连接Channel到一个远端地址。这会调用ChannelPipeline中下一个ChannelOutboundHandler的connect(ChannelHandlerContext, SocketAddress, ChannelPromise) |
| disconnect    | 断开Channel。这会调用ChannelPipeline中下一个ChannelOutboundHandler的disconnect(ChannelHandlerContext, ChannelPromise) |
| close         | 关闭Channel。这会调用ChannelPipeline中下一个ChannelOutboundHandler的close(ChannelHandlerContext,ChannelPromise) |
| deregister    | Channel从它之前分配的EventLoop上解除注册。这会调用ChannelPipeline中下一个ChannelOutboundHandler的deregister(ChannelHandlerContext, ChannelPromise) |
| flush         | 刷所有Channel待写的数据。这会调用ChannelPipeline中下一个ChannelOutboundHandler的flush(ChannelHandlerContext) |
| write         | 往Channel写一条消息。这会调用ChannelPipeline中下一个ChannelOutboundHandler的write(ChannelHandlerContext, Object msg, ChannelPromise)   注意：不会写消息到底层的Socket，只是排队等候。如果要写到Socket中，调用flush()或者writeAndFlush() |
| writeAndFlush | 这是先后调用write()和flush()的便捷方法。                     |
| read          | 请求从Channel中读更多的数据。这会调用ChannelPipeline中下一个ChannelOutboundHandler的read(ChannelHandlerContext) |

## ChannelHandlerContext接口
ChannelHandlerContext代表了一个ChannelHandler和一个ChannelPipeline之间的关系，它在ChannelHandler被添加到ChannelPipeline时被创建。ChannelHandlerContext的主要功能是管理它对应的ChannelHandler和属于同一个ChannelPipeline的其他ChannelHandler之间的交互。

ChannelHandlerContext有很多方法，其中一些方法Channel和ChannelPipeline也有，但是有些区别。如果你在Channel或者ChannelPipeline实例上调用这些方法，它们的调用会穿过整个pipeline。而在ChannelHandlerContext上调用的同样的方法，仅仅从当前ChannelHandler开始，走到pipeline中下一个可以处理这个event的ChannelHandler。


| 方法名                  | 描述                                                         |
| ----------------------- | ------------------------------------------------------------ |
| bind                    | 绑定到给定的SocketAddress，返回一个ChannelFuture             |
| channel                 | 返回绑定的Channel                                            |
| close                   | 关闭Channel，返回一个ChannelFuture                           |
| connect                 | 连接到给定的SocketAddress，返回一个ChannelFuture             |
| deregister              | 从先前分配的EventExecutor上解除注册，返回一个ChannelFuture   |
| disconnect              | 从远端断开，返回一个ChannelFuture                            |
| executor                | 返回分发event的EventExecutor                                 |
| fireChannelActive       | 触发调用下一个ChannelInboundHandler的channelActive()（已连接） |
| fireChannelInactive     | 触发调用下一个ChannelInboundHandler的channelInactive()（断开连接） |
| fireChannelRead         | 触发调用下一个ChannelInboundHandler的channelRead()（收到消息） |
| fireChannelReadComplete | 触发channelWritabilityChanged event到下一个ChannelInboundHandler |
| handler                 | 返回绑定的ChannelHandler                                     |
| isRemoved               | 如果绑定的ChannelHandler已从ChannelPipeline中删除，返回true  |
| name                    | 返回本ChannelHandlerContext 实例唯一的名字                   |
| Pipeline                | 返回绑定的ChannelPipeline                                    |
| read                    | 从Channel读数据到第一个输入buffer；如果成功，触发一条channelRead event，通知handler channelReadComplete |
| write                   | 通过本ChannelHandlerContext写消息穿过pipeline                |
在使用ChannelHandlerContext API时，请牢记下面几点：
- 一个ChannelHandler绑定的ChannelHandlerContext 永远不会改变，所以把它的引用缓存起来是安全的。
- 像我们在这节刚开始解释过的，ChannelHandlerContext的一些方法和其他类（Channel和ChannelPipeline）的方法名字相似，但是ChannelHandlerContext的方法采用了更短的event传递路程。我们应该尽可能利用这一点来实现最好的性能。


## 异常出站
1.添加ChannelFutureListener就是为了在ChannelFuture实例上调用addListener(ChannelFutureListener)方法，有两种方法可以做到这个。最常用的方法是在输出操作（比如write()）返回的ChannelFuture上调用addListener()。
```
ChannelFuture future = channel.write(...);
 future.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture f) throws Exception {
                    if (!f.isSuccess()) {
                        f.cause().printStackTrace();
                        r.channel().close();
                    } 
                }
            });
```
2.添加一个ChannelFutureListener到ChannelPromise，然后将这个ChannelPromise作为参数传入ChannelOutboundHandler方法。下面的代码和前一段代码有相同的效果。
```
public class OutboundExceptionHandler extends ChannelOutboundHandlerAdapter {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        promise.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture f) {
                if (!f.isSuccess()) {
                    f.cause().printStackTrace();
                    f.channel().close();
                }
            }
        });
    }
}
```


# ByteBuf
![image](https://user-images.githubusercontent.com/7789698/33237900-9d46f7ac-d2bb-11e7-9c72-0cbb05445aa6.png)
1. reader index前面的数据是已经读过的数据，这些数据可以扔掉
2. 从reader index开始，到writer index之前的数据是可读数据
3. 从writer index开始，为可写区域
  正是因为这样的设计，ByteBuf可以同时读写数据（只要可读区域和可写区域都还有空闲空间），而java.nio.ByteBuffer则必须调用flip()方法才能从写状态切换到读状态。


# ByteBufAllocator
```
ByteBufAllocator byteBufAllocator = channel.alloc();
        //        byteBufAllocator.compositeBuffer();
        //        byteBufAllocator.buffer();
ByteBuf byteBuf = byteBufAllocator.directBuffer();
```
![image](https://user-images.githubusercontent.com/7789698/33237903-c687db40-d2bb-11e7-99b5-6e232728da4d.png)
![image](https://user-images.githubusercontent.com/7789698/33237904-ca2bdbc0-d2bb-11e7-922a-04325053362e.png)

UnpooledByteBufAllocator:池化了ByteBuf并最大限度减少内存碎片。使用jemalloc(https://www.cnblogs.com/gaoxing/p/4253833.html)
PooledByteBufAllocator:不池化，每次调用返回新实例


# Unpooled
创建未池化ByteBuf

# ByteBufUtil类
- hexdump 十六进制形式打印ByteBuf内容
- equals 判断两个ByteBuf相等

[Netty系列之Netty高性能之道](http://www.infoq.com/cn/articles/netty-high-performance/)

[Netty系列之Netty线程模型
](http://www.infoq.com/cn/articles/netty-threading-model#mainLogin)