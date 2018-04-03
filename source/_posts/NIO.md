---
title: NIO
abbrlink: 429dd195
date: 2018-04-03 22:40:38
tags:
categories:
---

## 阻塞
假设有一个管道，进程A为管道的写入方，Ｂ为管道的读出方。

假设一开始内核缓冲区是空的，B作为读出方，被阻塞着。然后首先A往管道写入，这时候内核缓冲区由空的状态变到非空状态，内核就会产生一个事件告诉Ｂ该醒来了，这个事件姑且称之为“**缓冲区非空**”。

但是“缓冲区非空”事件通知B后，B却还没有读出数据；且内核许诺了不能把写入管道中的数据丢掉这个时候，Ａ写入的数据会滞留在内核缓冲区中，如果内核也缓冲区满了，B仍未开始读数据，最终内核缓冲区会被填满，这个时候会产生一个I/O事件，告诉进程A，你该等等（阻塞）了，我们把这个事件定义为“**缓冲区满**”。

假设后来Ｂ终于开始读数据了，于是内核的缓冲区空了出来，这时候内核会告诉A，内核缓冲区有空位了，你可以从长眠中醒来了，继续写数据了，我们把这个事件叫做“**缓冲区非满**”

也许事件Y1已经通知了A，但是A也没有数据写入了，而Ｂ继续读出数据，知道内核缓冲区空了。这个时候内核就告诉B，你需要阻塞了！，我们把这个时间定为“**缓冲区空**”。

阻塞I/O模式下，一个线程只能处理一个流的I/O事件。如果想要同时处理多个流，要么多进程(fork)，要么多线程(pthread_create)

## 非阻塞忙轮询的I/O方式
```
while true {
    for i in stream[]; {
        if i has data
            read until unavailable
    }
}
```
如果所有的流都没有数据，那么只会白白浪费CPU


## 非阻塞无差别轮询的I/O方式
```
while true {
    select(streams[])
    for i in streams[] {
        if i has data
            read until unavailable
    }
}
```
为了避免CPU空转，可以引进了一个代理（一开始有一位叫做select的代理，后来又有一位叫做poll的代理，不过两者的本质是一样的）。这个代理可以同时观察许多流的I/O事件，在空闲的时候，会把当前线程阻塞掉，当有一个或多个流有I/O事件时，就从阻塞态中醒来，于是我们的程序就会轮询一遍所有的流。

使用select，我们有O(n)的无差别轮询复杂度，同时处理的流越多，每一次无差别轮询时间就越长。

## epoll
epoll会把哪个流发生了怎样的I/O事件通知我们。（复杂度降低到了O(1)）

- epoll_create 创建一个epoll对象，一般epollfd = epoll_create()
- epoll_ctl （epoll_add/epoll_del的合体），往epoll对象中增加/删除某一个流的某一个事件
- epoll_ctl(epollfd, EPOLL_CTL_ADD, socket, EPOLLIN);//注册缓冲区非空事件，即有数据流入
- epoll_ctl(epollfd, EPOLL_CTL_DEL, socket, EPOLLOUT);//注册缓冲区非满事件，即流可以被写入
- epoll_wait(epollfd,…)等待直到注册的事件发生
  （注：当对一个非阻塞流的读写发生缓冲区满或缓冲区空，write/read会返回-1，并设置errno=EAGAIN。而epoll只关心缓冲区非满和缓冲区非空事件）。

```
while true {
    active_stream[] = epoll_wait(epollfd)
    for i in active_stream[] {
        read or write till
    }
}
```

在内核的最底层是中断，类似系统回调的机制。网卡设备对应一个中断号, 当网卡收到网络端的消息的时候会向CPU发起中断请求, 然后CPU处理该请求. 通过驱动程序 进而操作系统得到通知, 系统然后通知epoll, epoll通知用户代码。epoll在被内核初始化时（操作系统启动），同时会开辟出epoll自己的内核高速cache区，用于安置每一个我们想监控的socket，这些socket会以红黑树的形式保存在内核cache里，以支持快速的查找、插入、删除。这个内核高速cache区，就是建立连续的物理内存页，然后在之上建立slab层，简单的说，就是物理上分配好你想要的size的内存对象，每次使用时都是使用空闲的已分配好的对象。

## epoll和select的区别
进程通过将一个或多个fd传递给select或poll系统调用，阻塞在select;这样select/poll可以帮我们侦测许多fd是否就绪；但是select/poll是顺序扫描fd是否就绪，而且支持的fd数量有限。linux还提供了一个epoll系统调用，epoll是基于事件驱动方式，而不是顺序扫描,当有fd就绪时，立即回调函数rollback


传统的BIO里面socket.read()，如果TCP RecvBuffer里没有数据，函数会一直阻塞，直到收到数据，返回读到的数据。

对于NIO，如果TCP RecvBuffer有数据，就把数据从网卡读到内存，并且返回给用户；反之则直接返回0，永远不会阻塞。

最新的AIO(Async I/O)里面会更进一步：不但等待就绪是非阻塞的，就连数据从网卡到内存的过程也是异步的。

换句话说，BIO里用户最关心“我要读”，NIO里用户最关心"我可以读了"，在AIO模型里用户更需要关注的是“读完了”。

NIO一个重要的特点是：socket主要的读、写、注册和接收函数，在等待就绪阶段都是非阻塞的，真正的I/O操作是同步阻塞的（消耗CPU但性能非常高）。

NIO的读写函数可以立刻返回，这就给了我们不开线程利用CPU的最好机会：如果一个连接不能读写（socket.read()返回0或者socket.write()返回0），我们可以把这件事记下来，记录的方式通常是在Selector上注册标记位，然后切换到其它就绪的连接（channel）继续进行读写。

下面具体看下如何利用事件模型单线程处理所有I/O请求：

NIO的主要事件有几个：读就绪、写就绪、有新连接到来。

我们首先需要注册当这几个事件到来的时候所对应的处理器。然后在合适的时机告诉事件选择器：我对这个事件感兴趣。对于写操作，就是写不出去的时候对写事件感兴趣；对于读操作，就是完成连接和系统没有办法承载新读入的数据的时；对于accept，一般是服务器刚启动的时候；而对于connect，一般是connect失败需要重连或者直接异步调用connect的时候。

其次，用一个死循环选择就绪的事件，会执行系统调用（Linux 2.6之前是select、poll，2.6之后是epoll，Windows是IOCP），还会阻塞的等待新事件的到来。新事件到来的时候，会在selector上注册标记位，标示可读、可写或者有连接到来。

注意，select是阻塞的，无论是通过操作系统的通知（epoll）还是不停的轮询(select，poll)，这个函数是阻塞的。所以你可以放心大胆地在一个while(true)里面调用这个函数而不用担心CPU空转。



Java NIO 由以下几个核心部分组成：

- Channels
- Buffers
- Selectors



# Buffer
当我们需要与 NIO Channel 进行交互时, 我们就需要使用到 NIO Buffer, 即数据从 Buffer读取到 Channel 中, 并且从 Channel 中写入到 Buffer 中.
实际上, 一个 Buffer 其实就是一块内存区域, 我们可以在这个内存区域中进行数据的读写. NIO Buffer 其实是这样的内存块的一个封装, 并提供了一些操作方法让我们能够方便地进行数据的读写.

Buffer 类型有:

1. ByteBuffer 包括HeapByteBuffer和DirectByteBuffer两种。
  ByteBuffer
```
    public static ByteBuffer allocate(int capacity) {
        if (capacity < 0)
            throw new IllegalArgumentException();
        return new HeapByteBuffer(capacity, capacity);
    }
```

HeapByteBuffer 通过初始化字节数组hd，在虚拟机堆上申请内存空间。
 ```
   HeapByteBuffer(int cap, int lim) {            // package-private

        super(-1, 0, lim, cap, new byte[cap], 0);
        /*
        hb = new byte[cap];
        offset = 0;
        */
    }


    ByteBuffer(int mark, int pos, int lim, int cap,   // package-private
                 byte[] hb, int offset)
    {
        super(mark, pos, lim, cap);
        this.hb = hb;
        this.offset = offset;
    }

final byte[] hb;
 ```

```
    public static ByteBuffer allocateDirect(int capacity) {
        return new DirectByteBuffer(capacity);
    }
```


DirectByteBuffer 通过unsafe.allocateMemory在物理内存中申请地址空间（非jvm堆内存），并在ByteBuffer的address变量中维护指向该内存的地址。
unsafe.setMemory(base, size, (byte) 0)方法把新申请的内存数据清零。
 ```
   DirectByteBuffer(int cap) {                   // package-private
        super(-1, 0, cap, cap);
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);

        long base = 0;
        try {
            base = unsafe.allocateMemory(size);
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        unsafe.setMemory(base, size, (byte) 0);
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;
    }
 ```


2. CharBuffer
3. DoubleBuffer
4. FloatBuffer
5. IntBuffer
6. LongBuffer
7. ShortBuffer
8. [MappedByteBuffer](http://www.jianshu.com/p/f90866dcbffc)

使用Buffer读写数据一般遵循以下四个步骤：
1. 写入数据到Buffer
2. 调用flip()方法
3. 从Buffer中读取数据
4. 调用clear()方法或者compact()方法

当我们将数据写入到 Buffer 中时, Buffer 会记录我们已经写了多少的数据, 当我们需要从 Buffer 中读取数据时, 必须调用 Buffer.flip()将 Buffer 切换为读模式.
一旦读取了所有的 Buffer 数据, 那么我们必须清理 Buffer, 让其从新可写, 清理 Buffer 可以调用 Buffer.clear() 或 Buffer.compact().
例如:

```
        IntBuffer intBuffer = IntBuffer.allocate(2);
        intBuffer.put(12345678);
        intBuffer.put(2);
        intBuffer.flip();
        System.err.println(intBuffer.get());
        System.err.println(intBuffer.get());
```



- mark：初始值为-1，用于备份当前的position
- position：初始值为0。position表示当前可以写入或读取数据的位置。当写入或读取一个数据后， position向前移动到下一个位置。
- limit：
  写模式下，limit表示最多能往Buffer里写多少数据，等于capacity值。
  读模式下，limit表示最多可以读取多少数据。
- capacity：缓存数组大小


![image](https://user-images.githubusercontent.com/7789698/30013264-2a2093e8-9178-11e7-8b97-027d224908d3.png)



mark()：把当前的position赋值给mark
```
    public final Buffer mark() {
        mark = position;
        return this;
    }
```

reset()：把mark值还原给position
```
    public final Buffer reset() {
        int m = mark;
        if (m < 0)
            throw new InvalidMarkException();
        position = m;
        return this;
    }
```

clear()：一旦读完Buffer中的数据，需要让Buffer准备好再次被写入，clear会恢复状态值，但不会擦除数据。
```
    public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }
```

flip()：Buffer有两种模式，写模式和读模式，flip后Buffer从写模式变成读模式。
```
    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
```


rewind()：重置position为0，从头读写数据。
```
    public final Buffer rewind() {
        position = 0;
        mark = -1;
        return this;
    }
```



# Channel

- 流是单向的，通道是双向的，可读可写。
- 流读写是阻塞的，通道可以异步读写。
- 流中的数据可以选择性的先读到缓存中，通道的数据总是要先读到一个缓存中，或从缓存中写入，如下所示：
  ![image](https://user-images.githubusercontent.com/7789698/30013733-c37fffc2-917a-11e7-843f-2bab57034d57.png)

目前已知Channel的实现类有：
1. FileChannel  从文件中读写数据。
2. DatagramChannel  能通过UDP读写网络中的数据。
3. SocketChannel  能通过TCP读写网络中的数据。
4. ServerSocketChannel  可以监听新进来的TCP连接，像Web服务器那样。对每一个新进来的连接都会创建一个SocketChannel。

从Channel写到Buffer的例子

`int bytesRead = inChannel.read(buf); //read into buffer.`

从Buffer读取数据到Channel的例子：

`int bytesWritten = inChannel.write(buf);`



FileChannel的read、write和map通过其实现类FileChannelImpl实现。
```
File file = new RandomAccessFile("data.txt", "rw");
FileChannel channel = file.getChannel();
ByteBuffer buffer = ByteBuffer.allocate(48);

int bytesRead = channel.read(buffer);
while (bytesRead != -1) {
    System.out.println("Read " + bytesRead);
    buffer.flip();
    while(buffer.hasRemaining()){
        System.out.print((char) buffer.get());
    }
    buffer.clear();
    bytesRead = channel.read(buffer);
}
file.close();
```


```
    public int read(ByteBuffer dst) throws IOException {
        ensureOpen();
        if(!readable) {
            throw new NonReadableChannelException();
        } else {
            synchronized(positionLock) {
                int n = 0;
                int ti = -1;
                try {
                   begin();
                    ti = threads.add();
                    if(!isOpen()) {
                        return 0;
                    } else {
                        do {
                            n = IOUtil.read(this.fd, dst, -1L, this.nd);
                        } while(n == IOStatus.INTERRUPTED && this.isOpen());

                        return IOStatus.normalize(n);
                    }
                } finally {
                   threads.remove(ti);
                   end(n > 0);
                    assert IOStatus.check(n);
                }
            }
        }
    }
```

IOUtil
```
    static int read(FileDescriptor fd, ByteBuffer dst, long position, NativeDispatcher nd) throws IOException {
        if(dst.isReadOnly()) {
            throw new IllegalArgumentException("Read-only buffer");
        } else if(dst instanceof DirectBuffer) {
            return readIntoNativeBuffer(fd, dst, position, ));
        } else {
            ByteBuffer bb = Util.getTemporaryDirectBuffer(dst.remaining());
            try {
                int n = readIntoNativeBuffer(fd, bb, position, ));
                bb.flip();
                if(n > 0) {
                    dst.put(bb);
                }
                return n;
            } finally {
                Util.offerFirstTemporaryDirectBuffer(var5);
            }
        }
    }
```

通过上述实现可以看出，基于channel的文件数据读取步骤如下：
1、申请一块和缓存同大小的DirectByteBuffer bb。
2、读取数据到缓存bb，底层由NativeDispatcher的read实现。
3、把bb的数据读取到dst（用户定义的缓存，在jvm中分配内存）。

read方法导致数据复制了两次。


 ```
   public int write(ByteBuffer src) throws IOException {
       ensureOpen();
        if(!writable) {
            throw new NonWritableChannelException();
        } else {
            synchronized(positionLock) {
                int n = 0;
                int ti = -1;

                try {
                   begin();
                    ti =threads.add();
                    if(isOpen()) {
                        do {
                            n = IOUtil.write(this.fd, src, -1L, this.nd);
                        } while(n == IOStatus.INTERRUPTED && this.isOpen());
                        return IOStatus.normalize(n);
                    }
                    return 0;
                } finally {
                   threads.remove(ti);
                   end(n > 0);
                    assert IOStatus.check(n);
                }
            }
        }
    }
 ```


```
    static int write(FileDescriptor fd, ByteBuffer src, long position, NativeDispatcher nd) throws IOException {
        if(src instanceof DirectBuffer) {
            return writeFromNativeBuffer(fd, src, position, nd);
        } else {
            int pos = src.position();
            int lim = src.limit();
            assert(pos <= lim);
            int rem = pos <= lim?lim - pos:0;
            ByteBuffer bb = Util.getTemporaryDirectBuffer(rem);
            try {
                bb.put(src);
                bb.flip();
                src.position(pos);
                int n = writeFromNativeBuffer(fd, bb, position, nd);
                if(n > 0) {
                    src.position(pos + n);
                }
                return n;
            } finally {
                Util.offerFirstTemporaryDirectBuffer(bb);
            }
        }
    }
```

通过上述实现可以看出，基于channel的文件数据写入步骤如下：
1、申请一块DirectByteBuffer，bb大小为byteBuffer中的limit - position。
2、复制byteBuffer中的数据到bb中。
3、把数据从bb中写入到文件，底层由NativeDispatcher的write实现，具体如下：
```
private static int writeFromNativeBuffer(FileDescriptor fd, 
      ByteBuffer bb, long position, NativeDispatcher nd)
  throws IOException {
  int pos = bb.position();
  int lim = bb.limit();
  assert (pos <= lim);
  int rem = (pos <= lim ? lim - pos : 0);

  int written = 0;
  if (rem == 0)
      return 0;
  if (position != -1) {
      written = nd.pwrite(fd,
                          ((DirectBuffer)bb).address() + pos,
                          rem, position);
  } else {
      written = nd.write(fd, ((DirectBuffer)bb).address() + pos, rem);
  }
  if (written > 0)
      bb.position(pos + written);
  return written;
}
```


transferFrom()

FileChannel的transferFrom()方法可以将数据从源通道传输到FileChannel中。在SoketChannel的实现中，SocketChannel只会传输此刻准备好的数据（可能不足count字节）。因此，SocketChannel可能不会将请求的所有数据(count个字节)全部传输到FileChannel中。

`toChannel.transferFrom(0, fromChannel.size(), fromChannel);`

transferTo()

transferTo()方法将数据从FileChannel传输到其他的channel中

`fromChannel.transferTo(position, count, toChannel);`


## Scattering Reads
Scattering Reads是指数据从一个channel读取到多个buffer中


```
ByteBuffer header = ByteBuffer.allocate(128);
ByteBuffer body   = ByteBuffer.allocate(1024);
ByteBuffer[] bufferArray = { header, body };
channel.read(bufferArray);
```

read()方法按照buffer在数组中的顺序将从channel中读取的数据写入到buffer，当一个buffer被写满后，channel紧接着向另一个buffer中写


## Gathering Writes

Gathering Writes是指数据从多个buffer写入到同一个channel

write()方法会按照buffer在数组中的顺序，将数据写入到channel，注意只有position和limit之间的数据才会被写入



# Selector
Selector（选择器）是Java NIO中能够检测一到多个NIO通道，并能够知晓通道是否为诸如读写事件做好准备的组件。这样，一个单独的线程可以管理多个channel，从而管理多个网络连接。仅用单个线程来处理多个Channels的好处是，只需要更少的线程来处理通道。事实上，可以只用一个线程处理所有的通道。对于操作系统来说，线程之间上下文切换的开销很大，而且每个线程都要占用系统的一些资源（如内存）。因此，使用的线程越少越好。

Selector的创建
`Selector selector = Selector.open();`

向Selector注册通道

```
channel.configureBlocking(false);
SelectionKey key = channel.register(selector,Selectionkey.OP_READ);
```

- Connect   SelectionKey.OP_CONNECT(8)
- Accept     SelectionKey.OP_ACCEPT(16)
- Read        SelectionKey.OP_READ(1)
- Write       SelectionKey.OP_WRITE(4)


可以用“位或”操作符将常量连接
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;


## interest集合
```
int interestSet = selectionKey.interestOps();
boolean isInterestedInAccept  = (interestSet & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT；
boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT;
boolean isInterestedInRead    = interestSet & SelectionKey.OP_READ;
boolean isInterestedInWrite   = interestSet & SelectionKey.OP_WRITE;
```

## ready集合
ready 集合是通道已经准备就绪的操作的集合。在一次选择(Selection)之后，你会首先访问这个ready set。可以这样访问ready集合：

```
int readySet = selectionKey.readyOps();

selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();

    public final boolean isReadable() {
        return (readyOps() & OP_READ) != 0;
    }

    public final boolean isWritable() {
        return (readyOps() & OP_WRITE) != 0;
    }

    public final boolean isConnectable() {
        return (readyOps() & OP_CONNECT) != 0;
    }

    public final boolean isAcceptable() {
        return (readyOps() & OP_ACCEPT) != 0;
    }

```


```
Set selectedKeys = selector.selectedKeys();
Iterator keyIterator = selectedKeys.iterator();
while(keyIterator.hasNext()) {
    SelectionKey key = keyIterator.next();
    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.
    } else if (key.isConnectable()) {
        // a connection was established with a remote server.
    } else if (key.isReadable()) {
        // a channel is ready for reading
    } else if (key.isWritable()) {
        // a channel is ready for writing
    }
    keyIterator.remove();
}
```
注意每次迭代末尾的keyIterator.remove()调用。Selector不会自己从已选择键集中移除SelectionKey实例。必须在处理完通道时自己移除。下次该通道变成就绪时，Selector会再次将其放入已选择键集中。

SelectionKey.channel()方法返回的通道需要转型成你要处理的类型，如ServerSocketChannel或SocketChannel等。



```
    selector = Selector.open();//创建多路复用器
    serverSocketChannel = ServerSocketChannel.open();//打开管道
    serverSocketChannel.socket()
          .bind(new InetSocketAddress(port),1024);//绑定端口
    serverSocketChannel.configureBlocking(false);//非阻塞
    serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);//管道注册到多路复用器上

    Set<SelectionKey> keySet = selector.selectedKeys();
    Iterator<SelectionKey> iterator =  keySet.iterator();
    while (iterator.hasNext()){
           SelectionKey key = iterator.next();
           iterator.remove();
           try {
                if (key.isAcceptable()){
                    SocketChannel socketChannel = ((ServerSocketChannel) key.channel()).accept();//完成tcp握手，建立物理链路
                    socketChannel.configureBlocking(false);
                    socketChannel.register(selector,SelectionKey.OP_READ,ByteBuffer.allocate(1024));//注册客户端到多路复用器上，监听读操作
            }
           }catch (Exception e){//关闭
                if (key!=null){
                     key.cancel();
                     if (key.channel()!=null){
                           key.channel().close();
                      }
                  }
           }
   }
```



# 非阻塞IO通道（Non-blocking IO Pipelines）
非阻塞的IO管道（Non-blocking IO Pipelines）可以看做是整个非阻塞IO处理过程的链条。包括在以非阻塞形式进行的读与写操作

![1](https://user-images.githubusercontent.com/7789698/30017848-4baa5b4c-918d-11e7-905d-5463469a08b7.png)

我们的组件（Component）通过Selector检查当前Channel是否有数据需要写入。此时component读入数据，并且根据输入的数据input对外提供数据输出output。这个对外的数据输出output被写到了另一个Channel中。
一个非阻塞的IO管道不必同时需要读和写数据，通常来说有些管道只需要读数据，而另一些管道则只需写数据。当然一个非阻塞的IO管道他也可以同时从多个Channel中读取数据，例如同时冲多个SocketChannel中读取数据；

非阻塞和阻塞通道比较（Non-blocking vs. Blocking IO Pipelines）

非阻塞IO管道和阻塞IO管道之间最大的区别是他们各自如何从Channel（套接字socket或文件file）读写数据。
IO管道通常直接从流中读取数据，然后把数据分割为连续的消息。这个处理与我们读取流信息，用tokenizer进行解析非常相似。不同的是我们在这里会把数据流分割为更大一些的消息块。我把这个过程叫做Message Reader.下面是一张说明的插图：
![1](https://user-images.githubusercontent.com/7789698/30018294-e9992e5e-918e-11e7-96cc-e2ab16045e19.png)
一个阻塞IO管道的使用可以和输入流一样调用，每次从Channel中读取一个字节的数据，阻塞自身直到有数据可读。这个流程就是一个阻塞的Messsage Reader实现。

使用阻塞IO大大简化了Message Reader的实现成本。阻塞的Message Reader无需关注没有数据返回的情形，无需关注返回部分数据或者数据解析需要被复用的问题。
相似的，一个阻塞的Message Writer也不需要关注写入部分数据，和数据复用的问题。


基础的非阻塞通道设计（Basic Non-blocking IO Pipeline Design）

一个非阻塞的IO通道可以用单线程读取多个数据流。这个前提是相关的流可以切换为非阻塞模式（并不是所有流都可以以非阻塞形式操作）。在非阻塞模式下，读取一个流可能返回0个或多个字节。如果流还没有可供读取的数据那么就会返回0，其他大于1的返回都表明这是实际读取到的数据；
为了避开没有数据可读的流，我们结合Java NIO中的Selector。一个Selector可以注册多个SelectableChannel实例。当我们调用select()或selectorNow()方法时Selector会返回一个有数据可读的SelectableChannel实例。这个设计可以如下插图：
![1](https://user-images.githubusercontent.com/7789698/30018352-288fb452-918f-11e7-8764-ea3b57895dd2.png)

读取部分信息(Reading Partial Messages)

当我们冲SelectableChannel中读取一段数据后，我们并不知道这段数据是否是完整的一个message。因为一个数据段可能包含部分message，也就是说即可能少于一个message，也可能多一个message，正如下面这张插图所示意的那样：
![1](https://user-images.githubusercontent.com/7789698/30018411-572a7b8a-918f-11e7-9482-2c5dbe929d15.png)

要处理这种截断的message，我们会遇到两个问题：
1. 检测数据段中是否包含一个完整的message
2. 在message剩余部分获取到之前，我们如何处理不完整的message

检测完整message要求Message Reader查看数据段中的数据是否至少包含一个完整的message。如果包含一个或多个完整message，这些message可以被下发到通道中处理。查找完整message的过程是个大量重复的操作，所以这个操作必须是越快越好的。

当数据段中有一个不完整的message时，无论不完整消息是整个数据段还是说在完整message前后，这个不完整的message数据都需要在剩余部分获得前存储起来。

检查message完整性和存储不完整message都是Message Reader的职责。为了避免混淆来自不同Channel的数据，我们为每一个Channel分配一个Message Reader。整个设计大概是这样的：

![1](https://user-images.githubusercontent.com/7789698/30018448-740a93e8-918f-11e7-9062-5bf2af8f82f7.png)


当我们通过Selector获取到一个有数据可以读取的Channel之后，改Channel关联的Message Reader会读取数据，并且把数据打断为Message块。得到完整的message后就可以通过通道下发到其他组件进行处理。
一个Message Reader自然是协议相关的。他需要知道message的格式以便读取。如果我们的服务器是跨协议复用的，那他必须实现Message Reader的协议-大致类似于接收一个Message Reader工厂作为配置参数。

存储不完整的Message（Storing Partial Messages）

现在我们已经明确了由Message Reader负责不完整消息的存储直到接收到完整的消息。闲杂我们还需要知道这个存储过程需要如何来实现。
在设计的时候我们需要考虑两个关键因素：
1. 我们希望在拷贝消息数据的时候数据量能尽可能的小，拷贝量越大则性能相对越低；
2. 我们希望完整的消息是以顺序的字节存储，这样方便进行数据的解析；

为每个Message Reade分配Buffer（A Buffer Per Message Reader）

显然不完整的消息数据需要存储在某种buffer中。比较直接的办法是我们为每个Message Reader都分配一个内部的buffer成员。但是，多大的buffer才合适呢？这个buffer必须能存储下一个message最大的大小。如果一个message最大是1MB，那每个Message Reader内部的buffer就至少有1MB大小。
在百万级别的并发链接数下，1MB的buffer基本没法正常工作。举例来说，1,000,000 x 1MB就是1TB的内存大小！如果消息的最大数据量是16MB又需要多少内存呢？128MB呢？

可伸缩Buffer（Resizable Buffers）

另一个方案是在每个Message Reader内部维护一个容量可变的buffer。一个可变的buffer在初始化时占用较少控件，在消息变得很大超出容量时自动扩容。这样每个链接就不需要都占用比如1MB的空间。每个链接只使用承载下一个消息所必须的内存大小。

拷贝扩容（Resize by Copy）

第一种实现可伸缩buffer的办法是初始化buffer的时候只申请较少的空间，比如4KB。如果消息超出了4KB的大小那么开赔一个更大的空间，比如8KB，然后把4KB中的数据拷贝纸8KB的内存块中。
以拷贝方式扩容的优点是一个消息的全部数据都被保存在了一个连续的字节数组中。这使得数据解析变得更加容易。
同时它的缺点是会增加大量的数据拷贝操作。
为了减少数据的拷贝操作，你可以分析整个消息流中的消息大小，一次来找到最适合当前机器的可以减少拷贝操作的buffer大小。例如，你可能会注意到觉大多数的消息都是小于4KB的，因为他们仅仅包含了一个非常请求和响应。这意味着消息的处所荣校应该设置为4KB。
同时，你可能会发现如果一个消息大于4KB，很可能是因为他包含了一个文件。你会可能注意到 大多数通过系统的数据都是小于128KB的。所以我们可以在第一次扩容设置为128KB。
最后你可能会发现当一个消息大于128KB后，没有什么规律可循来确定下次分配的空间大小，这意味着最后的buffer容量应该设置为消息最大的可能数据量。
结合这三次扩容时的大小设置，可以一定程度上减少数据拷贝。4KB以下的数据无需拷贝。在1百万的连接下需要的空间例如1,000,000x4KB=4GB，目前（2015）大多数服务器都扛得住。4KB到128KB会仅需拷贝一次，即拷贝4KB数据到128KB的里面。消息大小介于128KB和最大容量的时需要拷贝两次。首先4KB数据被拷贝第二次是拷贝128KB的数据，所以总共需要拷贝132KB数据。假设没有很多的消息会超过128KB，那么这个方案还是可以接受的。
当一个消息被完整的处理完毕后，它占用的内容应当即刻被释放。这样下一个来自东一个链接通道的消息可以从最小的buffer大小重新开始。这个操作是必须的如果我们需要尽可能高效地复用不同链接之间的内存。大多数情况下并不是所有的链接都会在同一时刻需要大容量的buffer。
笔者写了一个完整的教程阐述了如何实现一个内存buffer使其支持扩容：[Resizable Arrays](http://tutorials.jenkov.com/java-performance/resizable-array.html) 。这个教程也附带了一个指向GitHub上的源码仓地址，里面有实现方案的具体代码。

追加扩容（Resize by Append）

另一种实现buffer扩容的方案是让buffer包含几个数组。当需要扩容的时候只需要在开辟一个新的字节数组，然后把内容写到里面去。
这种扩容也有两个具体的办法。一中是开辟单独的字节数组，然后用一个列表把这些独立数组关联起来。另一种是开辟一些更大的，相互共享的字节数组切片，然后用列表把这些切片和buffer关联起来。个人而言，笔者认为第二种切片方案更好一点点，但是它们之前的差异比较小。
这种追加扩容的方案不管是用独立数组还是切片都有一个优点，那就是写数据的时候不需要二外的拷贝操作。所有的数据可以直接从socket（Channel）中拷贝至数组活切片当中。
这种方案的缺点也很明显，就是数据不是存储在一个连续的数组中。这会使得数据的解析变得更加复杂，因为解析器不得不同时查找每一个独立数组的结尾和所有数组的结尾。正因为我们需要在写数据时查找消息的结尾，这个模型在设计实现时会相对不那么容易。

TLV编码消息(TLV Encoded Messages)

有些协议的消息消失采用的是一种TLV格式（Type, Length, Value）。这意味着当消息到达时，消息的完整大小存储在了消息的开始部分。我们可以立刻判断为消息开辟多少内存空间。
TLV编码是的内存管理变得更加简单。我们可以立刻知道为消息分配多少内存。即便是不完整的消息，buffer结尾后面也不会有浪费的内存。
TLV编码的一个缺点是我们需要在消息的全部数据接收到之前就开辟好需要用的所有内存。因此少量链接慢，但发送了大块数据的链接会占用较多内存，导致服务器无响应。
解决上诉问题的一个变通办法是使用一种内部包含多个TLV的消息格式。这样我们为每个TLV段分配内存而不是为整个的消息分配，并且只在消息的片段到达时才分配内存。但是消息片段很大时，任然会出现一样的问题。
另一个办法是为消息设置超时，如果长时间未接收到的消息（比如10-15秒）。这可以让服务器从偶发的并发处理大块消息恢复过来，不过还是会让服务器有一段时间无响应。另外恶意的DoS攻击会导致服务器开辟大量内存。
TLV编码有不同的变种。有多少字节使用这样确切的类型和字段长度取决于每个独立的TLV编码。有的TLV编码吧字段长度放在前面，接着放类型，最后放值。尽管字段的顺序不同，但他任然是一个TLV的类型。
TLV编码使得内存管理更加简单，这也是HTTP1.1协议让人觉得是一个不太优良的的协议的原因。正因如此，HTTP2.0协议在设计中也利用TLV编码来传输数据帧。也是因为这个原因我们设计了自己的利用TLV编码的网络协议[VStack.co](http://vstack.co/)。


写不完整的消息（Writing Partial Messages）

在非阻塞IO管道中，写数据也是一个不小的挑战。当你调用一个非阻塞模式Channel的write()方法时，无法保证有多少机字节被写入了ByteBuffer中。write方法返回了实际写入的字节数，所以跟踪记录已被写入的字节数也是可行的。这就是我们遇到的问题：持续记录被写入的不完整的小树知道一个消息中所有的数据都发送完毕。
为了管理不完整消息的写操作，我们需要创建一个Message Writer。正如前面的Message Reader，我们也需要每个Channel配备一个Message Writer来写数据。在每个Message Writer中我们记录准确的已经写入的字节数。
为了避免多个消息传递到Message Writer超出他所能处理到Channel的量，我们需要让到达的消息进入队列。Message Writer则尽可能快的将数据写到Channel里。
下面是一个流程图，展示的是不完整消息被写入的过程：

![1](https://user-images.githubusercontent.com/7789698/30021081-b69d9198-9198-11e7-8baa-7aec0dd22307.png)

为了使Message Writer能够持续发送刚才已经发送了一部分的消息，Message Writer需要被移植调用，这样他就可以发送更多数据。

如果你有大量的链接，你会持有大量的Message Writer实例。检查比如1百万的Message Writer实例是来确定他们是否处于可写状态是很慢的操作。首先，许多Message Writer可能根本就没有数据需要发送。我们不想检查这些实例。其次，不是所有的Channel都处于可写状态。我们不想浪费时间在这些非写入状态的Channel。

为了检查一个Channel是否可写，可以把它注册到Selector上。但是我们不希望把所有的Channel实例都注册到Selector。试想一下，如果你有1百万的链接，这里面大部分是空闲的，把1百万链接都祖册到Selector上。然后调用select方法的时候就会有很多的Channel处于可写状态。你需要检查所有这些链接中的Message Writer以确认是否有数据可写。
为了避免检查所有的这些Message Writer，以及那些根本没有消息需要发送给他们的Channel实例，我么可以采用入校两步策略：
1. 当有消息写入到Message Writer忠厚，把它关联的Channel注册到Selector上（如果还未注册的话）。
2. 当服务器有空的时候，可以检查Selector看看注册在上面的Channel实例是否处于可写状态。每个可写的channel，使其Message Writer向Channel中写入数据。如果Message Writer已经把所有的消息都写入Channel，把Channel从Selector上解绑。

这两个小步骤确保只有有数据要写的Channel才会被注册到Selector。

集成（Putting it All Together）

正如你所知到的，一个被阻塞的服务器需要时刻检查当前是否有显得完整消息抵达。在一个消息被完整的收到前，服务器可能需要检查多次。检查一次是不够的。
类似的，服务器也需要时刻检查当前是否有任何可写的数据。如果有的话，服务器需要检查相应的链接看他们是否处于可写状态。仅仅在消息第一次进入队列时检查是不够的，因为一个消息可能被部分写入。
总而言之，一个非阻塞的服务器要三个管道，并且经常执行：
1. 读数据管道，用来检查打开的链接是否有新的数据到达；
2. 处理数据管道，负责处理接收到的完整消息；
3. 写数据管道，用于检查是否有数据可以写入打开的连接中；
  这三个管道在循环中重复执行。你可以尝试优化它的执行。比如，如果没有消息在队列中等候，那么可以跳过写数据管道。或者，如果没有收到新的完整消息，你甚至可以跳过处理数据管道。
  下面这张流程图阐述了这整个服务器循环过程：
  ![1](https://user-images.githubusercontent.com/7789698/30021182-11fc30d0-9199-11e7-8a7f-74fec4018028.png)

假如你还是感觉这比较复杂难懂，可以去clone我们的源码仓： https://github.com/jjenkov/java-nio-server 也许亲眼看到了代码会帮助你理解这一块是如何实现的。

服务器线程模型（Server Thread Model）

我们在GitHub上的源码中实现的非阻塞IO服务使用了一个包含两条线程的线程模型。第一个线程负责从ServerSocketChannel接收到达的链接。另一个线程负责处理这些链接，包括读消息，处理消息，把响应写回到链接。这个双线程模型如下：


![1](https://user-images.githubusercontent.com/7789698/30021219-2a808a02-9199-11e7-82a6-2c274afc2745.png)

DatagramChannel数据报通道

```
DatagramChannel channel = DatagramChannel.open();
channel.socket().bind(new InetSocketAddress(9999));
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();

**channel.receive(buf);**
```

receive()方法会把接收到的数据包中的数据拷贝至给定的Buffer中。如果数据包的内容超过了Buffer的大小，剩余的数据会被直接丢弃。



```
String newData = "New String to wrte to file..."+System.currentTimeMillis();
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
buf.put(newData.getBytes());
buf.flip();

**int byteSent = channel.send(buf, new InetSocketAddress("jenkov.com", 80));**
```

上述示例会把一个字符串发送到“jenkov.com”服务器的UDP端口80.目前这个端口没有被任何程序监听，所以什么都不会发生。当发送了数据后，我们不会收到数据包是否被接收的的通知，这是由于UDP本身不保证任何数据的发送问题。


链接特定机器地址（Connecting to a Specific Address）

DatagramChannel实际上是可以指定到网络中的特定地址的。由于UDP是面向无连接的，这种链接方式并不会创建实际的连接，这和TCP通道类似。确切的说，他会锁定DatagramChannel,这样我们就只能通过特定的地址来收发数据包。

```
channel.connect(new InetSocketAddress("jenkov.com"), 80));
int bytesRead = channel.read(buf);
int bytesWritten = channel.write(buf);
```


 NIO Pipe管道

一个Java NIO的管道是两个线程间单向传输数据的连接。一个管道（Pipe）有一个source channel和一个sink channel(没想到合适的中文名)。我们把数据写到sink channel中，这些数据可以同过source channel再读取出来

![1](https://user-images.githubusercontent.com/7789698/30043278-18437b70-9229-11e7-8043-8053bbc77817.png)


创建管道(Creating a Pipe)

打开一个管道通过调用Pipe.open()工厂方法，如下：
`Pipe pipe = Pipe.open();`

向管道写入数据（Writing to a Pipe）

向管道写入数据需要访问他的sink channel,接下来就是调用write()方法写入数据了：

```
Pipe.SinkChannel sinkChannel = pipe.sink();
String newData = "New String to write to file..." + System.currentTimeMillis();

ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
buf.put(newData.getBytes());

buf.flip();

while(buf.hasRemaining()) {
    sinkChannel.write(buf);
}
```

从管道读取数据（Reading from a Pipe）

类似的从管道中读取数据需要访问他的source channel,接下来调用read()方法读取数据：
```
Pipe.SourceChannel sourceChannel = pipe.source();
ByteBuffer buf = ByteBuffer.allocate(48);

int bytesRead = inChannel.read(buf);
```


NIO AsynchronousFileChannel异步文件通道

```
Path path = Paths.get("data/test.xml");

AsynchronousFileChannel fileChannel =
    AsynchronousFileChannel.open(path, StandardOpenOption.READ);

ByteBuffer buffer = ByteBuffer.allocate(1024);
long position = 0;
Future<Integer> operation = fileChannel.read(buffer, 0);

while(!operation.isDone());

buffer.flip();
byte[] data = new byte[buffer.limit()];
buffer.get(data);
System.out.println(new String(data));
buffer.clear();
```


通过CompletionHandler读取数据（Reading Data Via a CompletionHandler）

```
fileChannel.read(buffer, position, buffer, new CompletionHandler<Integer, ByteBuffer>() {
    @Override
    public void completed(Integer result, ByteBuffer attachment) {
        System.out.println("result = " + result);

        attachment.flip();
        byte[] data = new byte[attachment.limit()];
        attachment.get(data);
        System.out.println(new String(data));
        attachment.clear();
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment) {
    }
});
```
一旦读取完成，将会触发CompletionHandler的completed()方法，并传入一个Integer和ByteBuffer。前面的整形表示的是读取到的字节数大小。第二个ByteBuffer也可以换成其他合适的对象方便数据写入。 如果读取操作失败了，那么会触发failed()方法。



参考：http://www.jianshu.com/p/052035037297
http://ifeve.com/java-nio-scattergather/
https://java-nio.avenwu.net/java-nio-channel.html
http://www.importnew.com/24794.html