## 前言

2018年最后几天，争取憋一篇文章出来( >﹏<。)～

在上文 [Netty源码分析一 初识Netty](http://blog.ztgreat.cn/article/66) 中利用官方的例子Echo 把Netty的整体流程走马观花的过了一遍，现在我们就一起来分析Netty中的一些具体模块，看Netty 是如何封装实现Java 中的NIO 的，今天我们来看看Netty 中的Channel(主要是 **NioSocketChannel**和**NioServerSocketChannel**)。

每篇文章我尽量不要太长，信息量不太多，我平常也看别人的博客，有时候 内容太长，确实没有看下去的欲望(｡•ˇ‸ˇ•｡) ，而且也不容易理清楚。

## Channel

在Netty 中，实现了自己的Channel，我们先来看看Java 中的Channel.

### Java 中Channel

`java.nio.channels.Channel`:

```
public interface Channel extends Closeable {
    public boolean isOpen();
    public void close() throws IOException;
}
```

很简单，就只有两个方法，判断当前Channle 是否是开启的，以及关闭当前Channel.

### Netty中的Channel

```
public interface Channel extends AttributeMap, ChannelOutboundInvoker, Comparable<Channel> {

    ChannelId id();
    
    //留意一下
    EventLoop eventLoop();
    
    Channel parent();
    ChannelConfig config();
    boolean isOpen();
    boolean isRegistered();
    boolean isActive();
    boolean isWritable();
    
    //留意一下
    ChannelPipeline pipeline();

    @Override
    Channel read();
    @Override
    Channel flush();

    /**
     * <em>Unsafe</em> operations that should <em>never</em> be called from user-code. These methods
     * are only provided to implement the actual transport, and must be invoked from an I/O thread except for the
     */
    interface Unsafe {

        void register(EventLoop eventLoop, ChannelPromise promise);
        void bind(SocketAddress localAddress, ChannelPromise promise);
        void connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);
        void disconnect(ChannelPromise promise);
        void close(ChannelPromise promise);
        void write(Object msg, ChannelPromise promise);
        void flush();
        //省略部分方法
    }
    //省略部分方法
}
```

为了减小篇幅，省略了部分的方法，方便阅读。

相比而言，Netty 中的Channel 就比Java 中的Netty 复杂和丰富多了，每个Channel 都和EventLoop，ChannelPipeline挂钩，同时还有一个Unsafe 接口，这个和Java 中的Unsafe 有异曲同工之妙，Java 中的Unsafe 主要是面向的Java自身使用，并非面向用户而言，把一些底层封装到了Unsafe 中，而Netty 中也是一样的，它封装了对 Java 底层 Socket 的操作, 因此实际上是沟通 Netty 上层和 Java 底层的重要的桥梁.

 注意Netty 的Channel 中还有一个parent，这个说明 channel是有等级的。我们可以通过调用Channel的`parent()`方法获取，`parent()`方法的返回取决于该Channel是怎么创建出来的。比如一个`SocketChannel`由一个`ServerSocketChannel`接收，因此当调用`SocketChannel`的`parent()`方法时将返回`ServerSocketChannel`

大概了解就可以了，这里也不会深入，我们一步一步来。

## NioSocketChannel

### 继承体系

![NioSocketChannel](http://img.blog.ztgreat.cn/document/netty/20181230120027.png)

**这里面没有Java NIO的任何身影**，AttributeMap这是绑定在Channel上的一个附件，相当于附件一样。

### AttributeMap

```
/**
 * Holds {@link Attribute}s which can be accessed via {@link AttributeKey}.
 * Implementations must be Thread-safe.
 */
public interface AttributeMap {
    <T> Attribute<T> attr(AttributeKey<T> key);
    <T> boolean hasAttr(AttributeKey<T> key);
}
```

我们可以看到这个是**线程安全**的，因此可以方便大胆的使用，有时候我们需要保存一会回话参数或者一些变量，通过AttributeMap就可以很方便的实现，使用的地方还是很多的。

## NioServerSocketChannel

相比NioSocketChannel，这个NioServerSocketChannel 是面向服务端的。

### 继承体系

![NioServerSocketChannel](http://img.blog.ztgreat.cn/document/netty/20181230120055.png)

其继承体系大体差不多。

除了 TCP 协议以外, Netty 还支持很多其他的连接协议, 并且每种协议还有 NIO(异步 IO) 和 BIO( 即传统的阻塞 IO) 版本的区别. 不同协议不同的阻塞类型的连接都有不同的 Channel 类型与之对应。

下面是一些常用的 Channel 类型:

- NioSocketChannel, 代表异步的客户端 TCP Socket 连接.
- NioServerSocketChannel, 异步的服务器端 TCP Socket 连接.
- NioDatagramChannel, 异步的 UDP 连接
- NioSctpChannel, 异步的客户端 Sctp 连接.
- NioSctpServerChannel, 异步的 Sctp 服务器端连接.
- OioSocketChannel, 同步的客户端 TCP Socket 连接.
- OioServerSocketChannel, 同步的服务器端 TCP Socket 连接.
- OioDatagramChannel, 同步的 UDP 连接
- OioSctpChannel, 同步的 Sctp 服务器端连接.
- OioSctpServerChannel, 同步的客户端 TCP Socket 连接.



>上面的异步是相对阻塞来说的，严格来说，是非完全异步模式的



在前面我们看到不管是NioSocketChannel 还是NioServerSocketChannel 它们的继承体系中都没有和Java的SocketChannel产生直接关系，我们来看看 NioSocketChannel 是怎么和 Java的 SocketChannel 联系在一起的，它们是一对一的关系。NioServerSocketChannel 和 ServerSocketChannel 同理，也是一对一的关系。


回想一下我们在**客户端**连接代码的初始化 Bootstrap 中, 会调用 channel() 方法, 传入 **NioSocketChannel.class**, 我们就先从这里入手。

## NioSocketChannel的实现

在 Bootstrap（客户端） 和 ServerBootstrap（服务端） 的启动过程中都会调用 channel(…) 方法：

```
//这里只列出了客户端的情况
Bootstrap b = new Bootstrap();
b.group(group);
b.channel(NioSocketChannel.class);
```

下面，我们来看 channel(…) 方法的源码：

```
// AbstractBootstrap
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    }
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
}
```

我们可以看到，这个方法只是设置了 channelFactory 为 ReflectiveChannelFactory 的一个实例，然后我们看下这里的 ReflectiveChannelFactory 到底是什么：

```
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {

    private final Class<? extends T> clazz;

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
    //省略部分方法
}
```

**newChannel()** 方法是 ChannelFactory 接口中的唯一方法，我们可以看到，ReflectiveChannelFactory#newChannel() 方法中使用了反射调用 Channel 的无参构造方法来创建 Channel。

既然这里只是产生的工厂类，那什么时候才真正的创建Channel呢？

- 对于 NioSocketChannel，由于是客户端，它的创建时机在 `connect(…)` 的时候；
- 对于 NioServerSocketChannel 来说，它充当服务端功能，它的创建时机在绑定端口 `bind(…)` 的时候。

接下来，我们来简单追踪下客户端的 Bootstrap 中 NioSocketChannel 的创建过程，看看 NioSocketChannel 是怎么和 Java 中的 SocketChannel 关联在一起的：

```
// Bootstrap
public ChannelFuture connect(String inetHost, int inetPort) {
    return connect(InetSocketAddress.createUnresolved(inetHost, inetPort));
}
```

然后再往里看，到这个方法：

```
public ChannelFuture connect(SocketAddress remoteAddress) {
    if (remoteAddress == null) {
        throw new NullPointerException("remoteAddress");
    // validate 顾名思义 只是校验一下，不重要
    validate();
    return doResolveAndConnect(remoteAddress, config.localAddress());
}
```

继续看 `doResolveAndConnect`：

```

private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
    // 初始化和注册，很明显我们需要关注一下这个方法
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    ......
}
```

我们看 `initAndRegister()` 方法：

```
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        // Channel 的实例化
        channel = channelFactory.newChannel();
        init(channel);
    } catch (Throwable t) {
        //省略代码
    }
    //省略代码
    return regFuture;
}
```

我们找到了 `channel = channelFactory.newChannel()` 这行代码，这个就和我们前面的分析联系起来了，这里会调用相应 Channel 的无参构造方法，创建Channel,至于ChannelFuture 这个我们后面再来看。

然后我们就可以去看 NioSocketChannel 的构造方法了：

```
public NioSocketChannel() {
    // SelectorProvider 实例用于创建 JDK 的 SocketChannel 实例
    this(DEFAULT_SELECTOR_PROVIDER);
}

public NioSocketChannel(SelectorProvider provider) {
    // 到这里，newSocket(provider) 方法会创建 JDK 的 SocketChannel
    this(newSocket(provider));
}
```

我们可以看到，在调用 newSocket(provider) 的时候，会创建 JDK NIO 的一个 SocketChannel 实例：

```
private static SocketChannel newSocket(SelectorProvider provider) {
    try {
        // 创建 SocketChannel 实例
        return provider.openSocketChannel();
    } catch (IOException e) {
        throw new ChannelException("Failed to open a socket.", e);
    }
}
```

NioServerSocketChannel 同理，也非常简单，从 `ServerBootstrap#bind(...)` 方法一路点进去就清楚了。

现在我们知道了，NioSocketChannel 在实例化过程中，**会先实例化 JDK 底层的 SocketChannel**，NioServerSocketChannel 也一样，会先实例化 ServerSocketChannel 实例：

说到这里，我们再继续往里看一下 NioSocketChannel 的构造方法：

```
public NioSocketChannel(SelectorProvider provider) {
    this(newSocket(provider));
}
```

刚才我们看到这里，newSocket(provider) 创建了底层的 SocketChannel 实例，我们继续往下看构造方法：

```
public NioSocketChannel(SocketChannel socket) {
        this(null, socket);
}

```

并传入参数 parent 为 null, socket 为刚才使用 newSocket 创建的 Java NIO SocketChannel, 因此生成的 NioSocketChannel 的 parent channel 是空的.

```
public NioSocketChannel(Channel parent, SocketChannel socket) {
    super(parent, socket);
    config = new NioSocketChannelConfig(this, socket.socket());
}
```

上面代码很简单，实例化了内部的 NioSocketChannelConfig 实例，它用于保存 channel 的配置信息，这里没有我们现在需要关心的内容，直接跳过。

调用父类构造器：

```
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
    // 客户端关心的是 OP_READ 事件，等待读取服务端返回数据
    super(parent, ch, SelectionKey.OP_READ);
}
```

因为客户端关心的是读事件，因此这里传入的是`SelectionKey.OP_READ`;

我们继续看下去：

```
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    // 这里只是保存了 SelectionKey.OP_READ 这个信息
    this.readInterestOp = readInterestOp;
    try {
        //配置 Java NIO SocketChannel 为非阻塞的.
        ch.configureBlocking(false);
    } catch (IOException e) {
        //...
    }
}
```

**设置了 SocketChannel 的非阻塞模式**

然后继续调用父类 AbstractChannel 的构造器:

```
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    //实例化 unsafe
    unsafe = newUnsafe();
    
    //创建 pipeline,没有channel 都有一个pipeline
    pipeline = new DefaultChannelPipeline(this);
}
```

到这里,  NioSocketChannel 就初始化完成了, 稍微总结一下构造一个 NioSocketChannel 所需要做的工作:

- 通过 NioSocketChannel.newSocket(DEFAULT_SELECTOR_PROVIDER) 打开一个新的 Java NIO SocketChannel
- AbstractChannel(Channel parent) 中初始化 AbstractChannel 的属性:
  - parent 属性设置为 null
  - unsafe 通过newUnsafe() 实例化一个 unsafe 对象, 它的类型是 AbstractNioByteChannel.NioByteUnsafe 内部类
  - pipeline 是 new DefaultChannelPipeline(this) 新创建的实例. 
- AbstractNioChannel 中的属性:
  - SelectableChannel ch 被设置为 Java SocketChannel
  - readInterestOp 被设置为 SelectionKey.OP_READ
  - SelectableChannel ch 被配置为非阻塞的 **ch.configureBlocking(false)**
- NioSocketChannel 中的属性:
  - SocketChannelConfig config = new NioSocketChannelConfig(this, socket.socket())



对于NioServerSocketChannel 而言，其构造方法类似，也**设置了非阻塞**，然后**设置服务端关心的 SelectionKey.OP_ACCEPT 事件**：

```
public NioServerSocketChannel(ServerSocketChannel channel) {
    // 对于服务端来说，关心的是 SelectionKey.OP_ACCEPT 事件，等待客户端连接
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

这节关于 Channel 的内容我们先介绍这么多，主要就是实例化了 JDK 层的 SocketChannel 或 ServerSocketChannel，然后设置了非阻塞模式，对于客户端，关心的是读事件，对于服务端关心的是Accept 事件。

最后，回答一个问题：

**NioSocketChannel 或者 NioServerSocketChannel 是如何与JDK 中的SocketChannel 联系起来的？**