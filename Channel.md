# Channel和Unsafe
Netty提供了自己的Channel和其子类实现，用于异步I/O操作和其他相关操作。

Unsafe是一个内部接口，聚合在Channel中协助进行网络读写相关的操作。Unsafe是Channel的内部辅助类，不应该被Netty框架的上层使用者调用。

# Channel功能说明
Channel是Netty网络操作抽象类，聚合了一组功能，包括但不限于网络的读、写，客户端发起连接、主动关闭连接，链路关闭，获取通信双方的网络地址等。
它也包含了Netty框架相关的一些功能，包括获取该Channel的EventLoop，获取缓冲分配器ByteBufAllocator和pipeline等。

## Channel工作原理
JDK NIO Channel的缺点：
1. JDK的SocketChannel和ServerSocketChannel没有统一的Channel接口供业务开发者使用
2. JDK的SocketChannel和ServerSocketChannel主要职责就是网络I/O操作，由于他们的是SPI类接口，具体的实现由虚拟机厂家决定，所以扩展不方便。
3. Netty的Channel需要能够跟Netty的整体架构融合在一起，例如I/O模型、基于ChannelPipeline的定制模型。以及基于元数据描述配置化的TCP参数等，
上述JDK的SocketChannel和ServerSocketChannel都没有提供，需要重新封装

Netty Channel设计原理：
1. 在Channel接口层，采用Facade模式进行统一封装，将网络I/O操作、网络I/O相关联的其他操作封装起来，统一对外提供。
2. Channel接口的定义尽量大而全，为SocketChannel和ServerSocketChannel提供统一的视图，由不同子类实现不同的功能，公共功能在抽象父类实现，
最大程度上实现功能和接口的重用
3. 具体实现采用聚合而非包含的方式，将相关的功能类聚合在Channel中，由Channel统一负责分配和调度，功能实现更加灵活。

### Channel功能介绍
#### 网络I/O操作
1. Channel read()：从当前的Channel中读取数据到第一个inbound缓冲区，如果数据被成功读取，触发ChannelHandler.channelRead(ChannelHandler
Context context, ChannelPromise promise)事件，读取操作API调用完成之后，紧接着会触发ChannelHandler.channelReadComplete事件
2. ChannelFuture write(Object msg)：请求将当前的msg通过ChannelPipeline写入到目标Channel中。write操作只是将消息存入到消息发送环形数
组中，并没有真正被发送，只有调用flush操作才会被写入Channel中，发送给对方。
3. ChannelFuture write(Object msg, ChannelPromise promise)：功能与write(Object msg)相同，但是携带了ChannelPromise参数负责设置
写入操作的结果
4. ChannelFuture writeAndFlush(Object msg, ChannelPromise promise)：与方法3类似，不同之处在于它会将消息写入到Channel中发送，等价
于单独调用write和flush。
5. ChannelFuture writeAndFlush(Object msg)：功能等同于4，但是没有携带ChannelPromise参数
6. Channel flush()：将之前写入消息发送环形数组中的消息全部写入目标Channel中，发送给通信对方。
7. ChannelFuture close(ChannelPromise promise)：主动关闭当前连接，通过ChannelPromise设置操作结果并进行结果通知，无论操作是否成功，
都可以通过ChannelPromise获取操作结果。该操作会级联触发ChannelPipeline中所有ChannelHandler的ChannelHandler.close(ChannelHandler
Context context, ChannelPromise promise)事件
8. ChannelFuture disconnect(ChannelPromise promise)：请求断开与远程通信对端的连接并使用ChannelPromise来获取操作结果的通知消息。该
方法会级联触发ChannelHandler.disconnect(ChannelHandlerContext context, ChannelPromise promise)事件
9. ChannelFuture connect(SocketAddress remoteAddress)：客户端使用指定的服务端地址remoteAddress发起连接请求，如果连接因为应答超时
而失败，ChannelFuture中的操作结果就是ConnectTimeoutException，如果连接被拒绝，操作结果是ConnectException。该方法会级联触发
ChannelHandler.connect(ChannelHandlerContext context, SocketAddress remoteAddress, SocketAddress localAddress, 
ChannelPromise promise)事件。
10. ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress)：该功能与9相似，唯一不同的是先绑定指定的本地
地址localAddress，然后再连接服务端
11. ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise)：该功能与9相似，唯一不同的是携带了ChannelPromise
参数用来写入操作结果
12. ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise)：与11功能类似，
唯一不同就是绑定了本地地址
13. ChannelFuture bind(SocketAddress localAddress)：绑定指定的本地Socket地址localAddress，该方法会级联触发ChannelHandler.bind
(ChannelHandlerContext context, ChannelPromise promise)事件
14. ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise)：与13功能类似，携带了ChannelPromise参数来写入操作结果
15. ChannelConfig config()：获取当前Channel的配置信息，例如CONNECT_TIMEOUT_MILLIS
16. boolean isOpen()：判断当前Channel是否已经打开
17. boolean isRegistered()：判断当前Channel是否已经注册到EventLoop上
18. boolean isActive()：判断当前Channel是否已经处于激活状态
19. ChannelMetadata metadata()：获取当前Channel的元数据描述信息，包括TCP参数配置等。
20. SocketAddress localAddress()：获取当前Channel的本地绑定地址
21. SocketAddress remoteAddress()：获取当前Channel通信的远程Socket地址

#### 其他API
1. EventLoop eventLoop()：获取Channel注册的EventLoop。Channel需要注册到EventLoop的多路复用器上，用来处理I/O事件，EventLoop本质上
就是处理网络读写事件的Reactor线程。在Netty中，EventLoop不仅仅用来处理网络事件，也可以用来执行定时任务和用户自定义NioTask等任务。
2. Channel parent()：对于服务端Channel而言，父Channel为空；对于客户端Channel，它的父Channel就是创建它的ServerSocketChannel
3. ChannelId id()：返回ChannelId对象，ChannelId是Channel的唯一标识

ChannelId的可能生成策略如下：
1. 机器的MAC地址等可以代表全局唯一的信息
2. 当前的进程ID
3. 当前系统时间的毫秒——System.currentTimeMillis()
4. 当前系统时间纳秒数——System.nanoTime()
5. 32位的随机整形数
6. 32位自增的序列数

## Channel源码
### Channel继承关系类图
服务端NioServerSocketChannel继承关系图
![FF7QMQ.png](https://s1.ax1x.com/2018/11/24/FF7QMQ.png)

客户端NioSocketChannel继承关系图
![FF7URU.png](https://s1.ax1x.com/2018/11/24/FF7URU.png)

### AbstractChannel源码
#### 成员变量定义
![FF7ci6.png](https://s1.ax1x.com/2018/11/24/FF7ci6.png)
1. 首先定义了5个静态全局异常
2. Channel parent：代表父类Channel
3. ChannelId id：采用默认方式生成的全局唯一ID
4. Unsafe unsafe：Unsafe实例
5. DefaultChannelPipeline pipeline：当前Channel对应的DefaultChannelPipeline
6. EventLoop eventLoop：当前Channel注册的EventLoop等一系列

通过变量定义可以看出，AbstractChannel聚合了所有Channel使用到的能力对象，由AbstractChannel提供初始化和统一封装，如果功能和子类强相关，则
定义成抽象方法由子类具体实现。

#### 核心API
Netty基于事件驱动，当Channel进行I/O操作时会产生对应的I/O事件，然后驱动事件在ChannelPipeline中传播，由对应的ChannelHandler对事件进行拦截
和处理，不关心的事件可以直接忽略。

网络I/O操作直接调用DefaultChannelPipeline的相关方法，由DefaultChannelPipeline中对应的ChannelHandler进行具体的逻辑处理。

AbstractChannel提供了一些公共API的具体实现，例如localAddress()和remoteAddress()，下面看一下remoteAddress的源码：
```java
    @Override
    public SocketAddress remoteAddress() {
        SocketAddress remoteAddress = this.remoteAddress;
        if (remoteAddress == null) {
            try {
                this.remoteAddress = remoteAddress = unsafe().remoteAddress();
            } catch (Throwable t) {
                // Sometimes fails on a closed socket in Windows.
                return null;
            }
        }
        return remoteAddress;
    }
```
首先从缓存的成员变量中获取，如果第一次调用为空，需要通过unsafe的remoteAddress获取，它是个抽象方法，具体由对应的Channel子类实现。

### AbstractNioChannel源码
#### 成员变量定义
![FFLmwj.png](https://s1.ax1x.com/2018/11/24/FFLmwj.png)
1. 定义了一个DO_CLOSE_CLOSED_CHANNEL_EXCEPTION静态全局异常
2. SelectableChannel ch：由于NIO Channel、NioSocketChannel和NioServerSocketChannel需要公用，所以定义了一个SocketChannel和
ServerSocketChannel的公共父类SelectableChannel，用于设置SelectableChannel参数和进行I/O操作。
3. int readInterestOp：代表了JDK SelectionKey的OP_READ
4. volatile SelectionKey selectionKey：该SelectionKey是Channel注册到EventLoop后返回的选择键。由于Channel会面临多个业务线程的并发
写操作，当SelectionKey由SelectionKey修改以后，为了能让其他业务线程感知到变化，所以需要使用volatile保证修改的可见性。
5. ChannelPromise connectPromise：代表连接操作结果
6. ScheduledFuture<?> connectTimeoutFuture：连接超时定时器
7. SocketAddress requestedRemoteAddress：请求的远程通信地址信息

#### 核心源码
AbstractNioChannel注册源码
```java
    @Override
    protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    // Force the Selector to select now as the "canceled" SelectionKey may still be
                    // cached and not removed because no Select.select(..) operation was called yet.
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    // We forced a select operation on the selector before but the SelectionKey is still cached
                    // for whatever reason. JDK bug ?
                    throw e;
                }
            }
        }
    }
```
首先定义一个boolean类型的局部变量selected来标识注册操作是否成功，调用SelectableChannel的register方法，将当前的Channel注册到EventLoop
的多路复用器上，SelectableChannel的注册方法定义如下：
```java
public abstract SelectionKey register(Selector sel, int ops, Object att) throws ClosedChannelException;
```
注册Channel的时候需要指定监听的网络操作位来表示Channel对哪几类网络事件感兴趣，具体的定义如下：
1. public static final int OP_READ = 1 << 0： 读操作位
2. public static final int OP_WRITE = 1 << 2：写操作位
3. public static final int OP_CONNECT = 1 << 3：客户端连接服务端操作位
4. public static final int OP_ACCEPT = 1 << 4：服务端接受客户端连接操作位

AbstractNioChannel注册的是0，说明对任何事件不感兴趣，仅仅完成注册操作。注册的时候可以指定附件，后续Channel接收到网络事件通知时可以从
SelectionKey中重新获取之前的附件进行处理。如果注册Channel成功，则返回SelectionKey，通过SelectionKey可以从多路复用器中获取Channel对象。

如果当前注册返回的SelectionKey已经被取消，则抛出CancelledKeyException异常，捕获该异常进行处理。如果是第一次处理该异常，调用多路复用器的
selectNow()方法将已经取消的selectionKey从多路复用器中删除掉。操作成功之后，将selected置为true，说明之前失效的selectionKey已经被删除掉。
继续发起下一次注册操作，如果成功则退出，如果仍然发生CancelledKeyException异常，说明我们无法删除已经被取消的selectionKey，发生这种问题，
直接抛出CancelledKeyException异常到上层进行统一处理。

下面看一下准备处理读操作之前需要设置网络操作位为读的代码：
```java
    @Override
    protected void doBeginRead() throws Exception {
        // Channel.read() or ChannelHandlerContext.read() was called
        final SelectionKey selectionKey = this.selectionKey;
        if (!selectionKey.isValid()) {
            return;
        }

        readPending = true;

        final int interestOps = selectionKey.interestOps();
        if ((interestOps & readInterestOp) == 0) {
            selectionKey.interestOps(interestOps | readInterestOp);
        }
    }
```
获取当前的SelectionKey进行判断，如果可用说明Channel当前状态正常，则可以进行正常的操作位修改。先将等待读设置为true，将SelectionKey当前的
操作位与读操作位按位于操作，如果等于0，说明目前并没有设置读操作位，通过interestOps | readInterestOp设置读操作位，最后调用selectionKey的
interestOps方法重新设置通道的网络操作位，这样就可以监听网络的读事件。

### AbstractNioByteChannel
#### 成员变量定义
private final Runnable flushTask：负责继续写半包消息

#### API
看下doWrite(ChannelOutboundBuffer in)的源码：
```java
    @Override
    protected void doWrite(ChannelOutboundBuffer in) throws Exception {
        int writeSpinCount = config().getWriteSpinCount();
        do {
            Object msg = in.current();
            if (msg == null) {
                // Wrote all messages.
                clearOpWrite();
                // Directly return here so incompleteWrite(...) is not called.
                return;
            }
            writeSpinCount -= doWriteInternal(in, msg);
        } while (writeSpinCount > 0);

        incompleteWrite(writeSpinCount < 0);
    }
```
首先从消息发送环形数组中弹出一个消息，判断该消息是否为空，如果为空，说明所有的消息都已经发送完成，清除半包标识，退出循环（该循环的次数默认最
多16次），设置半包消息有最大处理次数的原因是当循环发送的时候，I/O线程会一直进行写操作，此时I/O线程无法处理其他的I/O操作，例如读新的消息或
执行定时任务和NioTask等，如果网络I/O阻塞或者对方接受消息太慢，可能会导致线程假死。看一下清除半包标识clearOpWrite()的逻辑代码：
```java
    protected final void clearOpWrite() {
        final SelectionKey key = selectionKey();
        // Check first if the key is still valid as it may be canceled as part of the deregistration
        // from the EventLoop
        // See https://github.com/netty/netty/issues/2104
        if (!key.isValid()) {
            return;
        }
        final int interestOps = key.interestOps();
        if ((interestOps & SelectionKey.OP_WRITE) != 0) {
            key.interestOps(interestOps & ~SelectionKey.OP_WRITE);
        }
    }
```
首先获取当前的SelectionKey，如果当前的SelectionKey已被取消或者无效，直接返回。如果有效，则获取当前的监控的网络操作位，判断当前的网络操作
位是否监听写事件，如果正在监听，则取消对写事件的监听。

如果发送的消息不为空，则继续对消息进行处理，源码如下：
```java
    private int doWriteInternal(ChannelOutboundBuffer in, Object msg) throws Exception {
        if (msg instanceof ByteBuf) {
            ByteBuf buf = (ByteBuf) msg;
            if (!buf.isReadable()) {
                in.remove();
                return 0;
            }

            final int localFlushedAmount = doWriteBytes(buf);
            if (localFlushedAmount > 0) {
                in.progress(localFlushedAmount);
                if (!buf.isReadable()) {
                    in.remove();
                }
                return 1;
            }
        } else if (msg instanceof FileRegion) {
            FileRegion region = (FileRegion) msg;
            if (region.transferred() >= region.count()) {
                in.remove();
                return 0;
            }

            long localFlushedAmount = doWriteFileRegion(region);
            if (localFlushedAmount > 0) {
                in.progress(localFlushedAmount);
                if (region.transferred() >= region.count()) {
                    in.remove();
                }
                return 1;
            }
        } else {
            // Should not reach here.
            throw new Error();
        }
        return WRITE_STATUS_SNDBUF_FULL;
    }
```
首先判断消息的类型是否是ByteBuf，如果是进行强制转换，判断ByteBuf是否可读，如果不可读，将该消息从消息循环数组中删除，继续循环处理其他消息。
如果可读，则由具体的实现子类完成将ByeBuf写入到Channel中，并返回写入的字节数。如果返回的字节数小于等于0，则返回整形的最大数值。如果返回的
写入字节数大于0，设置该消息的处理的进度，然后再判断该消息是否可读，如果不可读，就把该消息移除，返回1。

最后还有一块处理半包发送任务的代码incompleteWrite，源码如下：
```java
    protected final void incompleteWrite(boolean setOpWrite) {
        // Did not write completely.
        if (setOpWrite) {
            setOpWrite();
        } else {
            // It is possible that we have set the write OP, woken up by NIO because the socket is writable, and then
            // use our write quantum. In this case we no longer want to set the write OP because the socket is still
            // writable (as far as we know). We will find out next time we attempt to write if the socket is writable
            // and set the write OP if necessary.
            clearOpWrite();

            // Schedule flush again later so other tasks can be picked up in the meantime
            eventLoop().execute(flushTask);
        }
    }
```
首先判断是否需要设置半包标识，如果需要则调用setOpWrite()来设置半包标识。如果没有设置写操作位，需要启动单独的Runnable flushTask，将其加入
到EventLoop中执行，由Runnable负责半包消息的发送，它就是简单的调用flush方法来发送缓冲数组中的消息。

### AbstractNioMessageChannel
#### 成员变量定义
boolean inputShutdown

#### API
```java
    @Override
    protected void doWrite(ChannelOutboundBuffer in) throws Exception {
        final SelectionKey key = selectionKey();
        final int interestOps = key.interestOps();

        for (;;) {
            Object msg = in.current();
            if (msg == null) {
                // Wrote all messages.
                if ((interestOps & SelectionKey.OP_WRITE) != 0) {
                    key.interestOps(interestOps & ~SelectionKey.OP_WRITE);
                }
                break;
            }
            try {
                boolean done = false;
                for (int i = config().getWriteSpinCount() - 1; i >= 0; i--) {
                    if (doWriteMessage(msg, in)) {
                        done = true;
                        break;
                    }
                }

                if (done) {
                    in.remove();
                } else {
                    // Did not write all messages.
                    if ((interestOps & SelectionKey.OP_WRITE) == 0) {
                        key.interestOps(interestOps | SelectionKey.OP_WRITE);
                    }
                    break;
                }
            } catch (Exception e) {
                if (continueOnWriteError()) {
                    in.remove(e);
                } else {
                    throw e;
                }
            }
        }
    }
```
在循环体中对消息进行发送，首先从ChannelOutboundBuffer中弹出一条消息进行处理，如果消息为空，说明发送缓冲区为空，所有消息都被发送完成。此时
清除写半包标识，退出循环。
然后借用writeSpinCount对单条消息进行发送，调用doWriteMessage判断消息是否发送成功，如果成功，则将发送标识done设置为true，退出循环，否则
继续执行循环，知道执行writeSpinCount次。
发送完成后，判断发送结果，如果当前的消息被完全发送出去，则将该消息从缓冲数组中删除；否则设置半包标识，注册SelectionKey.OP_WRITE到多路复用
器上，由多路复用器轮询对应的Channel重新发送尚未发送完全的半包消息。

AbstractNioMessageChannel和AbstractNioByteChannel不同之处是前者发送的是POJO对象，后者发送的是ByteBuf或者FileRegion。

### NioServerSocketChannel
#### 成员变量和静态方法
![FkzPPJ.png](https://s1.ax1x.com/2018/11/25/FkzPPJ.png)
1. ChannelMetadata METADATA：Channel元数据信息
2. ServerSocketChannelConfig config：用于配置ServerSocketChannel的TCP参数
3. ServerSocketChannel newSocket(SelectorProvider provider)：借助SelectorProvider的openServerSocketChannel方法打开新的ServerSocketChannel

#### API
```java
    @Override
    public boolean isActive() {
        return javaChannel().socket().isBound();
    }

    @Override
    public InetSocketAddress remoteAddress() {
        return null;
    }

    @Override
    protected ServerSocketChannel javaChannel() {
        return (ServerSocketChannel) super.javaChannel();
    }

    @Override
    protected SocketAddress localAddress0() {
        return SocketUtils.localSocketAddress(javaChannel().socket());
    }

    @Override
    protected void doBind(SocketAddress localAddress) throws Exception {
        if (PlatformDependent.javaVersion() >= 7) {
            javaChannel().bind(localAddress, config.getBacklog());
        } else {
            javaChannel().socket().bind(localAddress, config.getBacklog());
        }
    }
```
doBind方法一来运行时JAVA的版本，如果大于7就调用ServerSocketChannel的bind方法，否则调用ServerSocket的bind方法。
```java
    @Override
    protected int doReadMessages(List<Object> buf) throws Exception {
        SocketChannel ch = SocketUtils.accept(javaChannel());

        try {
            if (ch != null) {
                buf.add(new NioSocketChannel(this, ch));
                return 1;
            }
        } catch (Throwable t) {
            logger.warn("Failed to create a new channel from an accepted socket.", t);

            try {
                ch.close();
            } catch (Throwable t2) {
                logger.warn("Failed to close a socket.", t2);
            }
        }

        return 0;
    }
```
首先通过SocketUtils.accept来接受新的连接，如果新的连接不为空，则借助ServerSocketChannel和新接受的SocketChannel来创建一个NioSocketChannel，
并将NioSocketChannel添加到List<Object> buf，然后返回1，表示服务端接受消息成功。

对于NioServerSocketChannel来说，它的读取操作就是接收客户端的连接，创建NioSocketChannel对象。

#### 无关API
一些方法是与客户端Channel相关的，因此，对于服务端Channel无须实现，如果这些方法被误调，则返回UnsupportedOperationException异常。
```java
    @Override
    protected boolean doConnect(
            SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
        throw new UnsupportedOperationException();
    }

    @Override
    protected void doFinishConnect() throws Exception {
        throw new UnsupportedOperationException();
    }

    @Override
    protected SocketAddress remoteAddress0() {
        return null;
    }

    @Override
    protected void doDisconnect() throws Exception {
        throw new UnsupportedOperationException();
    }

    @Override
    protected boolean doWriteMessage(Object msg, ChannelOutboundBuffer in) throws Exception {
        throw new UnsupportedOperationException();
    }
```

### NioSocketChannel
#### 连接操作
```java
    @Override
    protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
        if (localAddress != null) {
            doBind0(localAddress);
        }

        boolean success = false;
        try {
            boolean connected = SocketUtils.connect(javaChannel(), remoteAddress);
            if (!connected) {
                selectionKey().interestOps(SelectionKey.OP_CONNECT);
            }
            success = true;
            return connected;
        } finally {
            if (!success) {
                doClose();
            }
        }
    }
```
首先判断本地Socket地址是否为空，如果不为空，先绑定本地Socket地址。绑定成功之后，SocketUtils.connect通过向远程Socket地址发起TCP连接。
对连接结果进行判断，连接结果有以下三种可能：
1. 连接成功，返回true
2. 暂时没有连接上，服务端没有返回ACK应答，连接结果不确定，返回false
3. 连接失败，直接抛出I/O异常

如果是结果2，需要将NioSocketChannel的SelectionKey设置为OP_CONNECT，监听连接网络操作位。如果抛出了I/O异常，说明客户端的TCP握手请求直接
被RESET或者被拒绝，此时需要调用doClose()关闭客户端连接
```java
    protected void doClose() throws Exception {
        super.doClose();
        javaChannel().close();
    }
```

#### 写半包
```java
    @Override
    protected void doWrite(ChannelOutboundBuffer in) throws Exception {
        SocketChannel ch = javaChannel();
        int writeSpinCount = config().getWriteSpinCount();
        do {
            if (in.isEmpty()) {
                // All written so clear OP_WRITE
                clearOpWrite();
                // Directly return here so incompleteWrite(...) is not called.
                return;
            }

            // Ensure the pending writes are made of ByteBufs only.
            int maxBytesPerGatheringWrite = ((NioSocketChannelConfig) config).getMaxBytesPerGatheringWrite();
            ByteBuffer[] nioBuffers = in.nioBuffers(1024, maxBytesPerGatheringWrite);
            int nioBufferCnt = in.nioBufferCount();

            // Always us nioBuffers() to workaround data-corruption.
            // See https://github.com/netty/netty/issues/2761
            switch (nioBufferCnt) {
                case 0:
                    // We have something else beside ByteBuffers to write so fallback to normal writes.
                    writeSpinCount -= doWrite0(in);
                    break;
                case 1: {
                    // Only one ByteBuf so use non-gathering write
                    // Zero length buffers are not added to nioBuffers by ChannelOutboundBuffer, so there is no need
                    // to check if the total size of all the buffers is non-zero.
                    ByteBuffer buffer = nioBuffers[0];
                    int attemptedBytes = buffer.remaining();
                    final int localWrittenBytes = ch.write(buffer);
                    if (localWrittenBytes <= 0) {
                        incompleteWrite(true);
                        return;
                    }
                    adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite);
                    in.removeBytes(localWrittenBytes);
                    --writeSpinCount;
                    break;
                }
                default: {
                    // Zero length buffers are not added to nioBuffers by ChannelOutboundBuffer, so there is no need
                    // to check if the total size of all the buffers is non-zero.
                    // We limit the max amount to int above so cast is safe
                    long attemptedBytes = in.nioBufferSize();
                    final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                    if (localWrittenBytes <= 0) {
                        incompleteWrite(true);
                        return;
                    }
                    // Casting to int is safe because we limit the total amount of data in the nioBuffers to int above.
                    adjustMaxBytesPerGatheringWrite((int) attemptedBytes, (int) localWrittenBytes,
                            maxBytesPerGatheringWrite);
                    in.removeBytes(localWrittenBytes);
                    --writeSpinCount;
                    break;
                }
            }
        } while (writeSpinCount > 0);

        incompleteWrite(writeSpinCount < 0);
    }
```
首先判断ChannelOutboundBuffer消息环形数组中是否有待发送的消息，如果没有，直接清除写操作位然后返回。从消息环形数组中获取可发送的ByteBuffer
数组以及可发送的数量，如果消息只有一个，直接取第一个消息，将消息写入Channel，如果写入的字节数小于等于0，设置网络监听位为写操作位，然后后返回。
如果消息的数量大于大于1，就先取出可发送数组的总字节数。

#### 读写操作
```java
    @Override
    protected int doReadBytes(ByteBuf byteBuf) throws Exception {
        final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
        allocHandle.attemptedBytesRead(byteBuf.writableBytes());
        return byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead());
    }
```
首先通过RecvByteBufAllocator.Handle设置从NioSocketChannel读取的字节数为ByteBuf可写的字节数，然后调用ByteBuf的writeBytes从Channel
中读取指定长度的字节。

# Unsafe
Unsafe接口是Channel接口的辅助接口，它不应该被用户直接调用，实际的I/O读写操作都是由Unsafe接口负责完成。
方法名 | 返回值 | 功能说明
--- | --- | ---
recvBufAllocHandle() | RecvByteBufAllocator.Handle | 返回用于ByteBuf内存分配的分配器
localAddress() | SocketAddress | 返回本地绑定的Socket地址
remoteAddress() | SocketAddress | 返回通信端的Socket地址
register(EventLoop eventLoop, ChannelPromise promise) | void | 将Channel注册到多路复用器上，操作完成之后，通知ChannelFuture
bind(SocketAddress localAddress, ChannelPromise promise) | void | 绑定本地地址SocketAddress到Channel，操作完成之后，通知ChannelFuture
connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) | void | 绑定本地Socket地址后，连接服务端，操作完成通知ChannelFuture
disconnect(ChannelPromise promise) | void | 断开Channel的连接，操作完成通知ChannelFuture
close(ChannelPromise promise) | void | 关闭Channel的连接，操作完成通知ChannelFuture
closeForcibly() | void | 强制关闭Channel的连接
deregister(ChannelPromise promise) | void | 从多路复用器上取消Channel的注册，操作完成通知ChannelFuture
beginRead() | void | 设置网络操作位为读用于读取消息
write(Object msg, ChannelPromise promise) | void | 发送消息，操作完成之后通知ChannelFuture
flush() | void | 将消息缓冲数组中的消息写入Channel
voidPromise() | ChannelPromise | 返回一个特殊的可重用和传递的ChannelPromise，它不用于操作成功或失败的通知器，仅仅作为一个容器被使用
outboundBuffer() | ChanneOutboundBuffer | 返回消息发送缓冲区

## Unsafe源码
### AbstractUnsafe
#### register方法
该方法主要用于将当前Unsafe对应的Channel注册到EventLoop的多路复用器上，然后调用DefaultChannelPipeline的fireChannelRegistered()方法。
如果Channel被激活并且是第一次被注册，则调用DefaultChannelPipeline的fireChannelActive()方法。
```java
        @Override
        public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            if (eventLoop == null) {
                throw new NullPointerException("eventLoop");
            }
            if (isRegistered()) {
                promise.setFailure(new IllegalStateException("registered to an event loop already"));
                return;
            }
            if (!isCompatible(eventLoop)) {
                promise.setFailure(
                        new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
                return;
            }

            AbstractChannel.this.eventLoop = eventLoop;

            if (eventLoop.inEventLoop()) {
                register0(promise);
            } else {
                try {
                    eventLoop.execute(new Runnable() {
                        @Override
                        public void run() {
                            register0(promise);
                        }
                    });
                } catch (Throwable t) {
                    logger.warn(
                            "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                            AbstractChannel.this, t);
                    closeForcibly();
                    closeFuture.setClosed();
                    safeSetFailure(promise, t);
                }
            }
        }
```
首先判断EventLoop是否为空，如果为空，抛出NullPointerException异常；接着判断Channel是否被注册，如果已经注册通知ChannelFuture注册失败；
接着判断EventLoop是否和当前Channel兼容，如果不兼容通知ChannelFuture注册失败。上述校验通过之后，会设置当前Channel的EventLoop为参数中的
EventLoop。判断当前所在的线程是否是Channel对应的EventLoop线程，如果是同一个线程则不存在多线程并发操作问题，直接调用register0方法进行注册；
如果是由用户线程或者其他线程发起的注册操作，则将注册操作封装成Runnable，放到EventLoop任务队列中执行。此处不直接执行register0放的原因是
**避免多线程操作Channel的问题**。
```java
        private void register0(ChannelPromise promise) {
            try {
                // check if the channel is still open as it could be closed in the mean time when the register
                // call was outside of the eventLoop
                if (!promise.setUncancellable() || !ensureOpen(promise)) {
                    return;
                }
                boolean firstRegistration = neverRegistered;
                doRegister();
                neverRegistered = false;
                registered = true;

                // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
                // user may already fire events through the pipeline in the ChannelFutureListener.
                pipeline.invokeHandlerAddedIfNeeded();

                safeSetSuccess(promise);
                pipeline.fireChannelRegistered();
                // Only fire a channelActive if the channel has never been registered. This prevents firing
                // multiple channel actives if the channel is deregistered and re-registered.
                if (isActive()) {
                    if (firstRegistration) {
                        pipeline.fireChannelActive();
                    } else if (config().isAutoRead()) {
                        // This channel was registered before and autoRead() is set. This means we need to begin read
                        // again so that we process inbound data.
                        //
                        // See https://github.com/netty/netty/issues/4805
                        beginRead();
                    }
                }
            } catch (Throwable t) {
                // Close the channel directly to avoid FD leak.
                closeForcibly();
                closeFuture.setClosed();
                safeSetFailure(promise, t);
            }
        }
```
首先调用ensureOpen方法确认当前Channel是否打开，如果没有打开则无法进行注册，直接返回。否则调用doRegister方法进行注册，它由AbstractNioUnsafe
对应的AbstractNioChannel来实现。

AbstractNioChannel的doRegister()不再分析，详情见上面。

#### bind
该方法主要用于绑定指定的端口，对于服务端，用于绑定监听端口，可以设置backlog参数；对于客户端而言，主要用来指定客户端Channel的本地绑定Socket地址。
```java
        @Override
        public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
            assertEventLoop();

            if (!promise.setUncancellable() || !ensureOpen(promise)) {
                return;
            }

            // See: https://github.com/netty/netty/issues/576
            if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
                localAddress instanceof InetSocketAddress &&
                !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
                !PlatformDependent.isWindows() && !PlatformDependent.maybeSuperUser()) {
                // Warn a user about the fact that a non-root user can't receive a
                // broadcast packet on *nix if the socket is bound on non-wildcard address.
                logger.warn(
                        "A non-root user can't receive a broadcast packet if the socket " +
                        "is not bound to a wildcard address; binding to a non-wildcard " +
                        "address (" + localAddress + ") anyway as requested.");
            }

            boolean wasActive = isActive();
            try {
                doBind(localAddress);
            } catch (Throwable t) {
                safeSetFailure(promise, t);
                closeIfClosed();
                return;
            }

            if (!wasActive && isActive()) {
                invokeLater(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.fireChannelActive();
                    }
                });
            }

            safeSetSuccess(promise);
        }
```
首先也是判断当前Channel是否打开，如果没有打开直接退出，确认Channel打开之后，直接调用doBind方法进行绑定。doBind方法对于服务端(NioServerSocketChannel)
和客户端(NioSocketChannel)有不同的实现，关于NioServerSocketChannel和NioSocketChannel的doBind实现在前面已经分析过。

#### disconnect
disconnect用于客户端或者服务端主动关闭连接。

#### close
```java
        private void close(final ChannelPromise promise, final Throwable cause,
                           final ClosedChannelException closeCause, final boolean notify) {
            if (!promise.setUncancellable()) {
                return;
            }

            if (closeInitiated) {
                if (closeFuture.isDone()) {
                    // Closed already.
                    safeSetSuccess(promise);
                } else if (!(promise instanceof VoidChannelPromise)) { // Only needed if no VoidChannelPromise.
                    // This means close() was called before so we just register a listener and return
                    closeFuture.addListener(new ChannelFutureListener() {
                        @Override
                        public void operationComplete(ChannelFuture future) throws Exception {
                            promise.setSuccess();
                        }
                    });
                }
                return;
            }

            closeInitiated = true;

            final boolean wasActive = isActive();
            final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            this.outboundBuffer = null; // Disallow adding any messages and flushes to outboundBuffer.
            Executor closeExecutor = prepareToClose();
            if (closeExecutor != null) {
                closeExecutor.execute(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            // Execute the close.
                            doClose0(promise);
                        } finally {
                            // Call invokeLater so closeAndDeregister is executed in the EventLoop again!
                            invokeLater(new Runnable() {
                                @Override
                                public void run() {
                                    if (outboundBuffer != null) {
                                        // Fail all the queued messages
                                        outboundBuffer.failFlushed(cause, notify);
                                        outboundBuffer.close(closeCause);
                                    }
                                    fireChannelInactiveAndDeregister(wasActive);
                                }
                            });
                        }
                    }
                });
            } else {
                try {
                    // Close the channel and fail the queued messages in all cases.
                    doClose0(promise);
                } finally {
                    if (outboundBuffer != null) {
                        // Fail all the queued messages.
                        outboundBuffer.failFlushed(cause, notify);
                        outboundBuffer.close(closeCause);
                    }
                }
                if (inFlush0) {
                    invokeLater(new Runnable() {
                        @Override
                        public void run() {
                            fireChannelInactiveAndDeregister(wasActive);
                        }
                    });
                } else {
                    fireChannelInactiveAndDeregister(wasActive);
                }
            }
        }
```
当第一次关闭Channel时，直接将closeInitiated设置为true，接着获取该Channel是否是激活状态和消息循环数组存储到临时变量，然后将消息循环数组置为null，
不允许消息的写入，然后获取一个执行器，NioSocketChannel中获取的是GlobalEventExecutor执行器，然后封装一个Runnable的Task交给这个执行器取执行，
该执行器回去关闭Channel，最后封装一个Task交给EventLoop去执行，该Task的主要任务是如果过循环消息数组不为NULL则去失效所有的消息，然后最后失活Channel
连接和取消注册。

#### write
该方法实际上将消息添加到环形发送数组中，并不是真正的写Channel。
```java
        @Override
        public final void write(Object msg, ChannelPromise promise) {
            assertEventLoop();

            ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null) {
                // If the outboundBuffer is null we know the channel was closed and so
                // need to fail the future right away. If it is not null the handling of the rest
                // will be done in flush0()
                // See https://github.com/netty/netty/issues/2362
                safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION);
                // release message now to prevent resource-leak
                ReferenceCountUtil.release(msg);
                return;
            }

            int size;
            try {
                msg = filterOutboundMessage(msg);
                size = pipeline.estimatorHandle().size(msg);
                if (size < 0) {
                    size = 0;
                }
            } catch (Throwable t) {
                safeSetFailure(promise, t);
                ReferenceCountUtil.release(msg);
                return;
            }

            outboundBuffer.addMessage(msg, size, promise);
        }
```
首先判断是否在Channel对应的I/O线程上进行操作，校验通过之后，获取环形发送数组，如果该数组为空，通知ChannelFuture操作失败。并且释放msg消息
的内存。然后根据具体的实现类进行消息类型的转换，然后获取消息的字节长度，如果是FileRegion，则返回0，最后把消息添加到环形数组中，操作完成之后
通知ChannelFuture。

#### flush
该方法负责将发送缓冲区中待发送的消息全部写入Channel中，并发送给通信对方。
```java
        @Override
        public final void flush() {
            assertEventLoop();

            ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null) {
                return;
            }

            outboundBuffer.addFlush();
            flush0();
        }

        @SuppressWarnings("deprecation")
        protected void flush0() {
            if (inFlush0) {
                // Avoid re-entrance
                return;
            }

            final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null || outboundBuffer.isEmpty()) {
                return;
            }

            inFlush0 = true;

            // Mark all pending write requests as failure if the channel is inactive.
            if (!isActive()) {
                try {
                    if (isOpen()) {
                        outboundBuffer.failFlushed(FLUSH0_NOT_YET_CONNECTED_EXCEPTION, true);
                    } else {
                        // Do not trigger channelWritabilityChanged because the channel is closed already.
                        outboundBuffer.failFlushed(FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
                    }
                } finally {
                    inFlush0 = false;
                }
                return;
            }

            try {
                doWrite(outboundBuffer);
            } catch (Throwable t) {
                if (t instanceof IOException && config().isAutoClose()) {
                    /**
                     * Just call {@link #close(ChannelPromise, Throwable, boolean)} here which will take care of
                     * failing all flushed messages and also ensure the actual close of the underlying transport
                     * will happen before the promises are notified.
                     *
                     * This is needed as otherwise {@link #isActive()} , {@link #isOpen()} and {@link #isWritable()}
                     * may still return {@code true} even if the channel should be closed as result of the exception.
                     */
                    close(voidPromise(), t, FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
                } else {
                    try {
                        shutdownOutput(voidPromise(), t);
                    } catch (Throwable t2) {
                        close(voidPromise(), t2, FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
                    }
                }
            } finally {
                inFlush0 = false;
            }
        }
```
首先调用ChannelOutboundBuffer的addFlush()设置发送消息缓冲区的范围，然后后调用flush0()方法进行发送。flush0()方法首先判断Channel是否失活，
如果失活，则通知ChannelFuture发送失败。否则调用doWrite方法进行具体的发送，该方法由具体的实现子类实现，关于NioSocketChannel的doWrite方法
在前面已经介绍过，这里不再重复。

### AbstractNioUnsafe
AbstractNioUnsafe继承AbstractUnSafe，主要实现了connect、finishConnect方法。
#### connect
```java
        @Override
        public final void connect(
                final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
            if (!promise.setUncancellable() || !ensureOpen(promise)) {
                return;
            }

            try {
                if (connectPromise != null) {
                    // Already a connect in process.
                    throw new ConnectionPendingException();
                }

                boolean wasActive = isActive();
                if (doConnect(remoteAddress, localAddress)) {
                    fulfillConnectPromise(promise, wasActive);
                } else {
                    connectPromise = promise;
                    requestedRemoteAddress = remoteAddress;

                    // Schedule connect timeout.
                    int connectTimeoutMillis = config().getConnectTimeoutMillis();
                    if (connectTimeoutMillis > 0) {
                        connectTimeoutFuture = eventLoop().schedule(new Runnable() {
                            @Override
                            public void run() {
                                ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                                ConnectTimeoutException cause =
                                        new ConnectTimeoutException("connection timed out: " + remoteAddress);
                                if (connectPromise != null && connectPromise.tryFailure(cause)) {
                                    close(voidPromise());
                                }
                            }
                        }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
                    }

                    promise.addListener(new ChannelFutureListener() {
                        @Override
                        public void operationComplete(ChannelFuture future) throws Exception {
                            if (future.isCancelled()) {
                                if (connectTimeoutFuture != null) {
                                    connectTimeoutFuture.cancel(false);
                                }
                                connectPromise = null;
                                close(voidPromise());
                            }
                        }
                    });
                }
            } catch (Throwable t) {
                promise.tryFailure(annotateConnectException(t, remoteAddress));
                closeIfClosed();
            }
        }
```
首先判断connectPromise是否为空，不为空，说明该Channel正在连接远程端点，此时抛出ConnectionPendingException异常，否则执行doConnect方法
进行真正的远程连接，该方法由具体的实现子类来执行，NioSocketChannel的doConnect方法在前文已经讲解过。如果返回true说明连接成功，则去触发
ChannelActive事件，如果返回false说明还没有成功建立连接可能还没收到应答，此时需要获取连接超时的事件，然后封装一个Runnable的Task，该Task会在
连接超时之后触发，通知ChannelFuture连接失败并发起取消注册操作，接着为ChannelPromise添加一个监听器，当连接完成之后会受到通知，如果连接操作被取消，
会取消超时任务的执行，并且发起取消注册操作。

#### finnishConnect
该方法会在接受到服务端TCP的握手应答消息时调用。
```java
        @Override
        public final void finishConnect() {
            // Note this method is invoked by the event loop only if the connection attempt was
            // neither cancelled nor timed out.

            assert eventLoop().inEventLoop();

            try {
                boolean wasActive = isActive();
                doFinishConnect();
                fulfillConnectPromise(connectPromise, wasActive);
            } catch (Throwable t) {
                fulfillConnectPromise(connectPromise, annotateConnectException(t, requestedRemoteAddress));
            } finally {
                // Check for null as the connectTimeoutFuture is only created if a connectTimeoutMillis > 0 is used
                // See https://github.com/netty/netty/issues/1770
                if (connectTimeoutFuture != null) {
                    connectTimeoutFuture.cancel(false);
                }
                connectPromise = null;
            }
        }
```
首先会缓存连接状态，紧接着调用具体子类的doFinishConnect方法对连接结果进行判断，只要连接失败(连接失败或发生链路被关闭、链路终端等异常)
doFinishConnect就会抛出Error，异常抛出之后，会关闭句柄释放资源，如果成功则调用fulfillConnectPromise方法，该方法的主要作用是将Channel的
监听位修改为读操作位。最后会将超时定时任务取消。

### NioByteUnsafe
#### read
```java
        @Override
        public final void read() {
            final ChannelConfig config = config();
            if (shouldBreakReadReady(config)) {
                clearReadPending();
                return;
            }
            final ChannelPipeline pipeline = pipeline();
            final ByteBufAllocator allocator = config.getAllocator();
            final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
            allocHandle.reset(config);

            ByteBuf byteBuf = null;
            boolean close = false;
            try {
                do {
                    byteBuf = allocHandle.allocate(allocator);
                    allocHandle.lastBytesRead(doReadBytes(byteBuf));
                    if (allocHandle.lastBytesRead() <= 0) {
                        // nothing was read. release the buffer.
                        byteBuf.release();
                        byteBuf = null;
                        close = allocHandle.lastBytesRead() < 0;
                        if (close) {
                            // There is nothing left to read as we received an EOF.
                            readPending = false;
                        }
                        break;
                    }

                    allocHandle.incMessagesRead(1);
                    readPending = false;
                    pipeline.fireChannelRead(byteBuf);
                    byteBuf = null;
                } while (allocHandle.continueReading());

                allocHandle.readComplete();
                pipeline.fireChannelReadComplete();

                if (close) {
                    closeOnRead(pipeline);
                }
            } catch (Throwable t) {
                handleReadException(pipeline, byteBuf, t, close, allocHandle);
            } finally {
                // Check if there is a readPending which was not processed yet.
                // This could be for two reasons:
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
                //
                // See https://github.com/netty/netty/issues/2254
                if (!readPending && !config.isAutoRead()) {
                    removeReadOp();
                }
            }
        }
    }
```
1. 首先获取NioSocketChannel的SocketChannelConfig，主要用来设置客户端连接的TCP参数。紧接着对allocHandle进行初始化。如果是首次调用，
将从SocketChannelConfig的RecvByteBufAllocator创建Handler。RecvByteBufAllocator的实现主要有两种，分别是AdaptiveRecvByteBufAllocator
和FixedRecvByteBufAllocator。
2. 接着通过接收缓冲区分配器RecvByteBufAllocator.Handle计算获得下次预分配的缓冲区容量，然后根据缓冲区容量进行缓冲区分配
3. 缓冲区分配完成之后，调用doReadBytes从Channel中进行消息的异步读取
4. 读取完成后，判断读取的字节数，如果读取的字节数小于等于0，表示没有就绪的消息可读或者发生了I/O异常，此时需要释放缓冲区，接着判断如果读取的字节数
小于0，需要将readPending = false并退出循环
5. 每一次读取消息，都会触发ChannelRead事件
6. 最后判断是否可以继续读取，如果不能则退出循环，触发ChannelReadComplete事件。

### AdaptiveRecvByteBufAllocator
AdaptiveRecvByteBufAllocator指的是缓冲区可以动态调整的分配器。
```java
    static final int DEFAULT_MINIMUM = 64;
    static final int DEFAULT_INITIAL = 1024;
    static final int DEFAULT_MAXIMUM = 65536;

    private static final int INDEX_INCREMENT = 4;
    private static final int INDEX_DECREMENT = 1;

    private static final int[] SIZE_TABLE;
```
1. 定义了系统的三个默认值，最小缓冲区的长度64，初始容量1024个字节，最大容量65535字节。
2. 定义了两个动态调整容量时的步进参数：扩张的步进索引为4，收缩的步进索引为1。
3. 定义了长度的向量表SIZE_TABLE

向量数组每个值对应一个Buffer容量，当容量小于512字节时，由于缓冲区已经比较小，需要降低步进值；当容量大于512时，说明需要解码的信息码流比较大，
这时采用调大步进幅度的方式减少动态扩张的频率，采用翻倍的方式进行扩张。

#### AdaptiveRecvByteBufAllocator.getSizeTableIndex
```java
    private static int getSizeTableIndex(final int size) {
        for (int low = 0, high = SIZE_TABLE.length - 1;;) {
            if (high < low) {
                return low;
            }
            if (high == low) {
                return high;
            }

            int mid = low + high >>> 1;
            int a = SIZE_TABLE[mid];
            int b = SIZE_TABLE[mid + 1];
            if (size > b) {
                low = mid + 1;
            } else if (size < a) {
                high = mid - 1;
            } else if (size == a) {
                return mid;
            } else {
                return mid + 1;
            }
        }
    }
```
该方法的主要作用是通过二分查找法根据容量size查找容量向量表的的对应索引。

### AdaptiveRecvByteBufAllocator.HandlerImpl
HandlerImpl是AdaptiveRecvByteBufAllocator的静态内部类。

使用动态缓冲区分配器的优点:
1. Netty是一个I/O框架，不同的应用场景，传输的码流大小千差万别。因此Netty根据上次实际读取的码流大小对下次的接受Buffer缓冲区进行预测和调整，
能够最大限度的满足各种场景
2. 性能更高，容量过大会导致内存占用开销增加，后续的Buffer处理性能下降；容量过小时需要频繁的内存扩张来接受大的请求消息，同样会导致性能下降。
3. 节约内存，避免一直进行大内存分配。

#### 成员变量
```java
        private final int minIndex;
        private final int maxIndex;
        private int index;
        private int nextReceiveBufferSize;
        private boolean decreaseNow;
```
主要有5个成员变量，分别是对应向量表中的最小索引、最大索引、当前索引、下一次预分配的Buffer大小和是否立即执行容量收缩操作。
#### record
当NioSocketChannel执行完读操作后，会计算获得本次轮询读取的总字节数，它就是参数actualReadBytes，执行record方法，根据实际读取的字节数对
ByteBuf进行动态伸缩和扩张。
```java
        private void record(int actualReadBytes) {
            if (actualReadBytes <= SIZE_TABLE[max(0, index - INDEX_DECREMENT - 1)]) {
                if (decreaseNow) {
                    index = max(index - INDEX_DECREMENT, minIndex);
                    nextReceiveBufferSize = SIZE_TABLE[index];
                    decreaseNow = false;
                } else {
                    decreaseNow = true;
                }
            } else if (actualReadBytes >= nextReceiveBufferSize) {
                index = min(index + INDEX_INCREMENT, maxIndex);
                nextReceiveBufferSize = SIZE_TABLE[index];
                decreaseNow = false;
            }
        }
```
首先，对当前索引做步进缩减，然后获取收缩后索引对应的容量，与实际读取的字节数进行对比，如果发现小于收缩后的容量，则重新对当前索引进行赋值，取
收缩后的索引和最小索引中的较大者作为最新的索引。然后后，为下一次缓冲区容量分配赋值。相反，如果当前实际读取的字节总数大于之前预分配的初始容量，
则说明实际分配的容量不足，需要动态扩张。重新计算索引，选取当前索引+扩张步进和最大索引中的较小值作为当前索引，然后后对下一次缓冲区容量分配赋值，
完成缓冲区容量的动态扩张。