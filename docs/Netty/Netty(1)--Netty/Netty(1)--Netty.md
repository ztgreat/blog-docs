## 前言

开始进入Java NIO系列的深入学习了，Netty 是Java系的一个著名NIO框架，Netty在互联网领域获得了广泛的应用，一些著名的开源组件也基于Netty构建，比如RPC框架、zookeeper等。

Netty从使用的角度来说非常的简单，套官方的Demo就可以了，当然对于我们大部分的猿类来说仅仅是使用是不可能的，对于一些核心的技术知识，必须要知其所以然，不然长江后浪推前浪，前浪死在沙滩上(＞﹏＜)

既然好用，那么说明设计得友好，当然各个类之间的关系肯定也就错综复杂，不断记录学习的过程，同时也不断的提高自己写作的能力，尽量用通俗的方式阐述清楚。

我自己的风格喜欢先看大概，有一个框框，很多知识点都了解一下，但是不深入，这样再深入研究的时候，不至于不知所云，因此对于Netty系列的文章，我尽量从整体框架入手，从整体再到局部。

Java NIO的知识 可以说从应用层到底层都有涉及，因此具体想如何学习，这个要看自己的定位，这里推荐几本书，尤其适合还在学校读书的朋友看，这个也是我以前看过的：`UNIX网络编程 卷1：套接字联网API`，`UNIX网络编程 卷2：进程间通信`，`UNIX环境高级编程`,整体偏向中下层，当然还有`TCP协议栈`相关的至少 对构建自己知识体系很有帮助，LZ是网络专业出身的，因此这方面相对比较熟悉。

**注:本系列文章中用到的Netty 版本为 4.x**



## 基础

首先你得要有NIO基础吧，下面是我前段时间简单些的关于Java NIO的文章，可以参考一下

[Java NIO之Channel、Buffer](http://blog.ztgreat.cn/article/43)

[Java NIO之Selector 浅析](http://blog.ztgreat.cn/article/47)

如果有Linux 相关的网络编程基础那就更好了（类似Linux 的select，epoll），了解Java 线程池。

## 例子

废话扯完，现在可以开始正文了(๑•̀ㅂ•́)و✧。我们用官方源码中的例子（**Echo**），先来进行简单分析，可以自行clone 官方源码，里面有很多例子，值得学习和研究，调试也更加的方便。

## 客户端

客户端的代码我稍微调整了一下，方便理解和后面的介绍

```
public final class EchoClient {

    static final String HOST = System.getProperty("host", "127.0.0.1");
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));
    static final int SIZE = Integer.parseInt(System.getProperty("size", "256"));

    public static void main(String[] args) throws Exception {

        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();

            b.group(group);
            b.channel(NioSocketChannel.class);
            b.option(ChannelOption.TCP_NODELAY, true);
            b.handler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     p.addLast(new EchoClientHandler());
                 }
             });
            // Start the client.
            ChannelFuture future = b.connect(HOST, PORT);
            future.sync();

            // Wait until the connection is closed.
            Channel channel= future.channel();
            ChannelFuture closeFuture=channel.closeFuture();
            closeFuture.sync();
        } finally {
            // Shut down the event loop to terminate all threads.
            group.shutdownGracefully();
        }
    }
}
```

可以看到代码非常的简洁，但是涵盖的信息量却不少，接下来我们来大概看看一下框框，并不会深入。

### EventLoopGroup

从名字上了解，这个是事件循环相关的，没关系，我们直接看官方注释：

```
/**
 * Special {@link EventExecutorGroup} which allows registering {@link Channel}s that get
 * processed for later selection during the event loop.
 */
```

注册Channel以及在后续处理一些事件循环，emmmm，没关系，再来看看它的继承关系

#### 继承关系

![netty eventLoopGroup](http://img.blog.ztgreat.cn/document/netty/20181221183913.png)

原来这个EventLoopGroup 可以看做是线程池，通过它来进行任务调度执行，完成Channel的注册，以及后面一些事件的处理，到这里，我们知道它的大概用途了，虽然不是很清晰和准确，但是大概心里有谱了。

### Bootstrap

接下来我们看到的就是Bootstrap，从名字知道，这个是启动类相关的，看看源码注释

```
/**
 * A {@link Bootstrap} that makes it easy to bootstrap a {@link Channel} to use
 * for clients.
 *
 * <p>The {@link #bind()} methods are useful in combination with connectionless transports such as datagram (UDP).
 * For regular TCP connections, please use the provided {@link #connect()} methods.</p>
 */
```

Bootstrap 是Netty 把复杂的启动过程进行封装后，方便我们用户使用，通过Bootstrap 我们可以很简单的建立TCP连接，可以理解成这是一个启动辅助类。

```
b.group(group);
b.channel(NioSocketChannel.class);
```

（1）首先把EventLoopGroup 放到了Bootstrap 中。

（2）设置channel，这里出现了NioSocketChannel，通过名字相关，这个应该和Java 中的Channel有联系，看看继承关系

![netty channel](http://img.blog.ztgreat.cn/document/netty/20181221185422.png)

```
/**
 * {@link io.netty.channel.socket.SocketChannel} which uses NIO selector based implementation.
 */
```

现在我们明白了NioSocketChannel 是 NIO selector 的一种实现。

我们看看 `channel`这个方法

```
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    }
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
}
```

这里并没有直接创建Channel（肯定啊，不知到ip这些，怎么建立socket (￣_,￣ )），而是创建了一个channelFactory ，很明显这个是一个工厂模式，通过Factory来创建Channel。

同样的 进入这个`ReflectiveChannelFactory`里面：

```
public ReflectiveChannelFactory(Class<? extends T> clazz) {
    if (clazz == null) {
        throw new NullPointerException("clazz");
    }
    this.clazz = clazz;
}

@Override
public T newChannel() {
    try {
        return clazz.getConstructor().newInstance();
    } catch (Throwable t) {
        throw new ChannelException("Unable to create Channel from class " + clazz, t);
    }
}
```

它提供了一个重要的方法, 即 **newChannel**.

根据上面代码, 我们就可以确定:

- 客户端 中的Bootstrap 中的 ChannelFactory 的实现是 ReflectiveChannelFactory
- 生成的 Channel 的具体类型是 NioSocketChannel. 
  Channel 的实例化过程, 其实就是调用的 ChannelFactory#newChannel 方法, 而实例化的 Channel 的具体的类型又是和在初始化 Bootstrap 时传入的 channel() 方法的参数相关. 因此对于我们这个例子中的客户端的 Bootstrap 而言, 生成的的 Channel 实例就是 NioSocketChannel.



接下里我们继续往下看:

```
b.option(ChannelOption.TCP_NODELAY, true);
```

其实这个时候，我们可以不用care 这些的，如果了解TCP协议的话，知道这个应该是设置一些属性（option），不过这里还是提一下算了，名字表达的意思是`禁止延迟`，看下文档：

>**TCP_NODELAY**
>
>If set, disable the Nagle algorithm. This means that segments are always sent as soon as possible, even if there is only a small amount of data. When not set, data is buffered until there is a sufficient amount to send out, thereby avoiding the frequent sending of small packets, which results in poor utilization of the network. This option is overridden by **TCP_CORK**; however, setting this option forces an explicit flush of pending output, even if **TCP_CORK** is currently set.

参考[TCP protocol](https://linux.die.net/man/7/tcp)

TCP/IP协议中有一个`Nagle`算法。Nagle算法通过减少需要传输的数据包，来优化网络。关于Nagle算法，在内核实现中，数据包的发送和接受会先做缓存，分别对应于写缓存和读缓存。
启动TCP_NODELAY，就意味着禁用了Nagle算法，允许小包的发送。对于延时敏感型，同时数据传输量比较小的应用，开启TCP_NODELAY选项无疑是一个正确的选择。

如果开启了Nagle算法，就很可能出现频繁的延时，数据只有在写缓存中累积到一定量之后，才会被发送出去，这样明显提高了网络利用率。但是这由不可避免地增加了延时。

### handler

```
b.handler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline p = ch.pipeline();
        p.addLast(new EchoClientHandler());
    }
});
```

在这段代码中，关键词handler,ChannelPipeline 这里不会太细讲，但是会提出相关概念，具体的分析会在后面给出。

handler  中 传入的是ChannelInitializer ，字面意思呢就是Channel 初始化，在其方法中 initChannel，参数便是创建好的 Channel,再获取ChannelPipeline，设置我们的逻辑逻辑方法 EchoClientHandler。

下面简单大致描述一下Channel的内部逻辑结构：

每个 Channel 内部都有一个 pipeline，pipeline 由**多个 handler** 组成，**handler 之间的顺序是很重要的**，因为 IO 事件将按照顺序顺次经过 pipeline 上的 handler，这样每个 handler 只关注自己的业务逻辑，由多个 handler 组合来完成一些复杂的逻辑。

![netty pipeline](http://img.blog.ztgreat.cn/document/netty/20181223152945.png)

首先，两个重要的概念：**Inbound** 和 **Outbound**。在 Netty 中，IO 事件被分为 Inbound 事件和 Outbound 事件。

**Outbound** 的 **out** 指的是 **出去**，比如 connect、write、flush 这些 IO 操作是往外部方向进行的(**数据往外传输**)，它们就属于 Outbound 事件，其他的，类似 accept、read 这种就属于 Inbound 事件（**有远程数据进入**）。

定义处理 Inbound 事件的 handler 需要实现 ChannelInboundHandler，定义处理 Outbound 事件的 handler 需要实现 ChannelOutboundHandler，通常我们继承其相应的适配器，然后实现相关方法就可以了。

![netty handler](http://img.blog.ztgreat.cn/document/netty/20181223154151.png)

特别的，如果我们希望定义一个 handler 能同时处理 Inbound 和 Outbound 事件（可以认为是双工模式），我们可以继承 ChannelDuplexHandler 的方式(继承关系图中未列出来)。

关于Handler 和Pipeline 我们就暂说到这里。

### connect

```
ChannelFuture future = b.connect(HOST, PORT);
```

很明显，这里是进行connect，连接远程主机，建立TCP连接，那么这里面肯定也会创建Channle,我们跟踪查看一下

#### doResolveAndConnect

```
private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();

    if (regFuture.isDone()) {
        if (!regFuture.isSuccess()) {
            return regFuture;
        }
        return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
    } else {
        
        //省略代码
    }
}
```

中间省略几个方法调用，最终我们看这个doResolveAndConnect 方法，里面对Channel进行初始化和注册（initAndRegister 方法），我们看看 initAndRegister 方法：

#### initAndRegister

```
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        channel = channelFactory.newChannel();
        init(channel);
    } catch (Throwable t) {
        if (channel != null) {
            channel.unsafe().closeForcibly();
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }
    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }
    return regFuture;
}
```



这里我们看到 channel = channelFactory.newChannel(); 这个便是创建Channel,而这个channelFactory便是我们前面看到的 `ReflectiveChannelFactory`

### ChannelFuture

我们继续往下看

```
// Start the client.
ChannelFuture future = b.connect(HOST, PORT);
future.sync();

// Wait until the connection is closed.
Channel channel= future.channel();
ChannelFuture closeFuture=channel.closeFuture();
closeFuture.sync();
```

通过connect后，返回了一个ChannelFuture，查看这个ChannelFuture的继承关系，我们发现它继承JDK 中的Future。

关于 JDK 中的Future 接口，大家应该都比较熟悉吧，在使用 Java 的线程池 ThreadPoolExecutor 的时候了。在 **submit** 一个任务到线程池中的时候，返回的就是一个 **Future** 实例，通过它来获取提交的任务的执行状态和最终的执行结果，我们最常用它的 `isDone()` 和 `get()` 方法。

既然如此，虽然我们还没有研究ChannelFuture，但是我们可以猜测其实和JDK 中的Future 是差不多的，future.sync(); 方法，通过sync这个名字，我们知道这个是在同步等待，等待什么？，等待连接建立成功.

当连接建立完毕后，我们就可以通过ChannelFuture 来获取Channel。通过Channel又获取了一个Future,通过名字我们知道这个一个关于Channel 关闭后的Future，然后执行sync 方法，等待Channel 关闭

如果没有closeFuture.sync();那么执行完连接后（假设我们连接成功了），客户端会直接退出,这肯定是我们不想要的。

ok,到这里我们就把客户端的大致逻辑梳理了一下，虽然没有深入，但是对于大致的功能，流程还是有所明白的，下面再来看看服务端的代码。

## 服务端

```
public final class EchoServer {

    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));

    public static void main(String[] args) throws Exception {

        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();

            b.group(bossGroup, workerGroup);
            b.channel(NioServerSocketChannel.class);
            b.option(ChannelOption.SO_BACKLOG, 100);
            b.handler(new LoggingHandler(LogLevel.INFO));

            b.childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     p.addLast(new EchoServerHandler());
                 }
             });

            // Start the server.
            ChannelFuture future = b.bind(PORT);
            future.sync();

            // Wait until the server socket is closed.
            Channel channel = future.channel();
            ChannelFuture closeFuture = channel.closeFuture();
            closeFuture.sync();
        } finally {
            // Shut down all event loops to terminate all threads.
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

我们看到其实服务端和客户端非常的相似，这里我们也只是大概看一下差异性。

### ServerBootstrap

相比客户端的 Bootstrap，这里变成了ServerBootstrap，这个也好理解，毕竟一个是客户端，一个是服务端，两者分开，更好的理解和设计维护，从功能上来上都是一样，都是便于用户的使用，通过ServerBootstrap 我们可以很轻松的开启一个服务端的Netty 应用。

###  EventLoopGroup

这里我们看到有两个EventLoopGroup，而客户端只有一个，也就是说服务端用了两个线程池来处理一些任务。

对于客户端而言，通常就是连接服务器，然后与服务器进行交互。

对于服务端而言，服务端需要监听是否有客户端来进行连接，也就是对客户端的Accept的处理，当accept后，才是真正的与客户端进行交互。

我们可以从源码了解到一些信息：

```
/**
 * Specify the {@link EventLoopGroup} which is used for the parent (acceptor) and the child (client).
 */
@Override
public ServerBootstrap group(EventLoopGroup group) {
    return group(group, group);
}

/**
 * Set the {@link EventLoopGroup} for the parent (acceptor) and the child (client). These
 * {@link EventLoopGroup}'s are used to handle all the events and IO for {@link ServerChannel} and
 * {@link Channel}'s.
 */
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
    super.group(parentGroup);
    if (childGroup == null) {
        throw new NullPointerException("childGroup");
    }
    if (this.childGroup != null) {
        throw new IllegalStateException("childGroup set already");
    }
    this.childGroup = childGroup;
    return this;
}
```



其中一个EventLoopGroup则是负责处理客户端的**连接请求**; 而 另一个 就是负责**客户端连接后的 IO 交互**.，这里，我们不深入，有个概念就可以了。

### NioServerSocketChannel

同样的，这里也和客户端的NioSocketChannel 不一样，NioServerSocketChannel 是针对Netty 服务端的NIO selector的实现。

```
/**
 * A {@link io.netty.channel.socket.ServerSocketChannel} implementation which uses
 * NIO selector based implementation to accept new connections.
 */
```

#### ChannelOption

```
b.option(ChannelOption.SO_BACKLOG, 100);
```

查看 `java.nio.channels.ServerSocketChannel` bind 方法:

```
/**
* Binds the channel's socket to a local address and configures the socket to
* listen for connections.
*
* <p> This method is used to establish an association between the socket and
* a local address. Once an association is established then the socket remains
* bound until the channel is closed.
*
* <p> The {@code backlog} parameter is the maximum number of pending
* connections on the socket. Its exact semantics are implementation specific.
* In particular, an implementation may impose a maximum length or may choose
* to ignore the parameter altogther. If the {@code backlog} parameter has
* the value {@code 0}, or a negative value, then an implementation specific
* default is used.
*/

public abstract ServerSocketChannel bind(SocketAddress local, int backlog)
        throws IOException;
```

最大等待建立连接的scoket数量（等待建立socket连接 排队数量）

**注:下面说法可能不严谨，仅供 提供相关信息，可以自己查阅资料**

（1）一种说法:

在linux系统内核中有一个队列：syns queue

用于保存**半连接状态**的请求（TCP三次握手），其大小通过/proc/sys/net/ipv4/tcp_max_syn_backlog指定，常见的TCP SYN FLOOD恶意DOS攻击方式就是建立大量的半连接状态的请求，然后丢弃，导致syns queue不能保存其它正常的请求。

>*tcp_max_syn_backlog* (integer; default: see below; since Linux 2.2)
>
>The maximum number of queued connection requests which have still not received an acknowledgement from the connecting client. If this number is exceeded, the kernel will begin dropping requests. The default value of 256 is increased to 1024 when the memory present in the system is adequate or greater (>= 128Mb), and reduced to 128 for those systems with very low memory (<= 32Mb). It is recommended that if this needs to be increased above 1024, 

参考[TCP protocol](https://linux.die.net/man/7/tcp)

更多可以参考了解 [Linux网络编程---TCP三次握手，SYN洪水攻击](https://blog.csdn.net/u014634338/article/details/49154685)

（2）第二种说法

`backlog`参数的行为在`Linux`2.2之后有所改变。现在，它指定了等待`accept`系统调用的已建立连接队列的长度，而不是待完成连接请求数

更多可以参考:

[How TCP backlog works in Linux](http://veithen.io/2014/01/01/how-tcp-backlog-works-in-linux.html)(需要科学上网)

[[译文]深入理解Linux TCP backlog](https://www.jianshu.com/p/7fde92785056)

https://linux.die.net/man/2/listen



具体的和Linux系统实现有关，这个我觉得自己明白怎么回事就可以了。

### handler

服务器端的 handler 的添加过程和客户端的有点区别, 和 EventLoopGroup 一样, 服务器端的 handler 也有两个, 一个是通过 handler() 方法设置 handler 字段, 另一个是通过 childHandler() 设置 childHandler 字段. 

类比  EventLoopGroup，这个 handler 字段与 accept 过程有关, 也就是说这个 handler 负责处理客户端的**连接请求**; 而 childHandler 就是负责**客户端连接后的 IO 交互**.



##  结语

走马观花的把Netty的demo过了一遍，对每个模块进行了简单的提及和分析，没有探讨任何细节的地方，首先我们对Netty有了一个感性的认识，了解了一些关键点，知道了大概的流程，这样，并不需要我们非要一行一行代码的弄清楚，有了大致认识后，可以去了解相关的知识点，有了这些基础知识后，再对netty 深入分析，相当于把零散的东西再进行一次整合，这样从整体到局部，再从局部到整体，我觉得这样对框架的整体认识才会比较深刻。