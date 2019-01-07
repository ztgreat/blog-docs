Java NIO 由以下几个核心部分组成：

1、Buffer

2、Channel

3、Selector

在[Java NIO之Channel、Buffer](http://blog.ztgreat.cn/article/43)中 简单的介绍了Buffer，Channel ，本文主要讲解NIO的Selector相关知识,Selector是整个NIO的核心，理解selector机制是理解整个NIO的关键所在。

理解selector 之前，最好要知道I/O模型，如果对Linux 网络编程(poll epoll select)熟悉那就更棒了，因为我以前接触过Linux 网络编程，所以在接触Java Nio的时候感觉很亲切.

# 举个栗子

```
/**
 * server 端
 */
public class Server {

    private ByteBuffer readBuffer = ByteBuffer.allocateDirect(1024);
    private ByteBuffer writeBuffer = ByteBuffer.allocateDirect(1024);
    private Selector selector;

    public Server() throws IOException{
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        //设置非阻塞模式
        serverSocketChannel.configureBlocking(false);
        ServerSocket serverSocket = serverSocketChannel.socket();
        serverSocket.bind(new InetSocketAddress(8080));
        System.out.println("listening on port 8080");
        //打开 selector
        this.selector = Selector.open();

        //在 selector 注册感兴趣的事件
        serverSocketChannel.register(this.selector, SelectionKey.OP_ACCEPT);
    }

    private void start() throws Exception{

        while(true){
            //调用阻塞的select,等待 selector上注册的事件发生
            this.selector.select();

            //获取就绪事件
            Iterator<SelectionKey> iterator = this.selector.selectedKeys().iterator();
            while(iterator.hasNext()){
                SelectionKey selectionKey = iterator.next();
                //先移除该事件,避免重复通知
                iterator.remove();
                // 新连接
                if(selectionKey.isAcceptable()){
                    System.out.println("isAcceptable");
                    ServerSocketChannel server = (ServerSocketChannel)selectionKey.channel();

                    // 新注册channel
                    SocketChannel socketChannel  = server.accept();
                    if(socketChannel==null){
                        continue;
                    }
                    //非阻塞模式
                    socketChannel.configureBlocking(false);

                    //注册读事件（服务端一般不注册 可写事件）
                    socketChannel.register(selector, SelectionKey.OP_READ);


                    ByteBuffer buffer = ByteBuffer.allocateDirect(1024);
                    buffer.put("hi new channel".getBytes());
                    buffer.flip();
                    int writeBytes= socketChannel.write(buffer);

                }

                // 服务端关心的可读，意味着有数据从client传来了数据
                if(selectionKey.isReadable()){
                    System.out.println("isReadable");
                    SocketChannel socketChannel = (SocketChannel)selectionKey.channel();

                    readBuffer.clear();
                    socketChannel.read(readBuffer);
                    readBuffer.flip();

                    String receiveData= Charset.forName("UTF-8").decode(readBuffer).toString();
                    System.out.println("receiveData:"+receiveData);


                    //这里将收到的数据发回给客户端
                    writeBuffer.clear();
                    writeBuffer.put(receiveData.getBytes());
                    writeBuffer.flip();
                    while(writeBuffer.hasRemaining()){
                        //防止写缓冲区满，需要检测是否完全写入
                        System.out.println("写入数据:"+socketChannel.write(writeBuffer));
                    }
                }

            }
        }
    }

    public static void main(String[] args) throws Exception{
        new Server().start();
    }

}
```
先贴一个简单的例子，后面根据这个进行拆分。
# Selector

Selector 一般称 为选择器 （或 多路复用器）  。它是Java NIO核心组件中的一个，用于检查一个或多个NIO Channel（通道）的状态是否处于可读、可写。可以实现**单线程管理多个channels**,也就是说可以管理多个网络连接。
使用 Selector 的图解如下:

![这里写图片描述](http://img.blog.ztgreat.cn/document/nio/20180813104125886.png)

图片来自：[Java NIO Overview](http://tutorials.jenkov.com/java-nio/overview.html)

为了使用 Selector, 我们首先需要将 Channel 注册到 Selector 中, 随后调用 Selector 的 select()方法, 这个方法会**阻塞**, 直到注册在 Selector 中的 Channel 发送可读写事件（或其它注册事件）. 当这个方法返回后, 当前的这个线程就可以处理 Channel 的事件了（已准备就绪的Channel）.


## 创建Selector 
通过 Selector.open()方法, 我们可以创建一个选择器:

```
Selector selector = Selector.open();
```
这里提一下Selector 在Windows和Linux 上有不同的实现

## 将 Channel 注册到Selector 中
我们需要将 Channel 注册到Selector 中，这样才能通过 Selector  监控  Channel :
```
//非阻塞模式
channel.configureBlocking(false);

SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```

> 注意, 如果一个 Channel 要注册到 Selector 中, 那么这个 Channel 必须是非阻塞的, 即channel.configureBlocking(false);
> 因为 Channel 必须要是非阻塞的, 因此 FileChannel 是不能够使用选择器的, 因为 FileChannel 都是阻塞的.

因为 channel 是非阻塞的，因此当没有数据的时候会理解返回，因此 实际上 Selector  是不断的在轮询其注册的 channel 是否有数据就绪。


在使用 Channel.register()方法时, 第二个参数指定了我们对 Channel 的什么类型的事件感兴趣, 这些事件有:

 - Connect, 连接事件(TCP 连接), 对应于SelectionKey.OP_CONNECT
 - Accept, 确认事件, 对应于SelectionKey.OP_ACCEPT
 - Read, 读事件, 对应于SelectionKey.OP_READ, 表示 buffer 可读.
 - Write, 写事件, 对应于SelectionKey.OP_WRITE, 表示 buffer 可写.

我们可以使用或运算|来组合多个事件, 例如:
```
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```

##  SelectionKey
当我们使用 register 注册一个 Channel 时, 会返回一个 SelectionKey 对象, 这个对象包含了如下内容:

 - interest set, 即我们感兴趣的事件集
 - ready set
 - channel
 - selector
 - attached object, 可选的附加对象

### interest set
我们可以通过如下方式获取 interest set:

```
int interestSet = selectionKey.interestOps();

boolean isInterestedInAccept  = interestSet & SelectionKey.OP_ACCEPT;
boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT;
boolean isInterestedInRead    = interestSet & SelectionKey.OP_READ;
boolean isInterestedInWrite   = interestSet & SelectionKey.OP_WRITE; 
```

### ready set
代表了 Channel 已经就绪的操作.，我们可以使用如下方法进行判断:
```
int readySet = selectionKey.readyOps();

selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();
```
### Channel 和 Selector

我们可以通过 SelectionKey 获取相对应的 Channel 和 Selector:

```
Channel  channel  = selectionKey.channel();
Selector selector = selectionKey.selector();  
```

### Attaching Object

我们可以在selectionKey中附加一个对象:

```
selectionKey.attach(theObject);
Object attachedObj = selectionKey.attachment();
```

或者在注册时直接附加:

```
SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);
```

##  阻塞 的 select 

### select 
调用 Selector 的 select()方法, 这个方法会**阻塞**, 直到注册在 Selector 中的 Channel 发送可读写事件（或其它注册事件）. 当这个方法返回后, 当前的这个线程就可以处理 Channel 的事件了（已准备就绪的Channel）.

### select(long timeout)
 select(long timeout)，超时阻塞等待timeout毫秒(参数)，而不是 select()那样一直阻塞等待，直到有事件就绪。

>  注意, select()方法返回的值表示有多少个 Channel 可操作.


## 获取就绪的 Channel（或 事件）

如果 select()方法返回值表示有多个 Channel 准备好了, 那么我们可以通过 Selected key set 访问这个 Channel:

```
Set<SelectionKey> selectedKeys = selector.selectedKeys();

Iterator<SelectionKey> keyIterator = selectedKeys.iterator();

while(keyIterator.hasNext()) {
    
    SelectionKey key = keyIterator.next();
	keyIterator.remove();
	
	//可能有多个注册事件就绪
	
    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.

    } 
    if (key.isConnectable()) {
        // a connection was established with a remote server.

    }
    if (key.isReadable()) {
        // a channel is ready for reading

    } 
    if (key.isWritable()) {
        // a channel is ready for writing
    }

    
}
```

注意, 在每次迭代时, 我们都调用 "keyIterator.remove()" 将这个 key 从迭代器中删除, 因为 select() 方法仅仅是简单地将就绪的 IO 操作放到 selectedKeys 集合中, 因此如果我们从 selectedKeys 获取到一个 key, 但是没有将它删除, 那么下一次 select 时, 这个 key 所对应的 IO 事件还在 selectedKeys 中.

## 唤醒

选择器执行选择的过程，系统底层会依次询问每个通道是否已经就绪，这个过程可能会造成调用线程进入阻塞状态,wakeup方式可以唤醒在select（）方法中阻塞的线程。
```
selector.wakeup()
```

## Selector 整体使用 流程

 ![Selector 整体使用 流程](http://img.blog.ztgreat.cn/document/nio/20180813115608209.png)

1、建立  ServerSocketChannel
2、通过 Selector.open() 打开一个 Selector.
    3、将 Channel 注册到 Selector 中, 并设置需要监听的事件
    4、循环:
           1、调用 select() 方法
           2、调用 selector.selectedKeys() 获取 就绪 Channel
           3、迭代每个 selected key:
    
                - 就处理 就绪的IO事件
    
                - 根据需要更改 selected key 的监听事件.
    
                - 将已经处理过的 key 从 selected keys 集合中删除.

最后这里附上和前面对应的客户端的代码：

```
/**
 * client 端
 */
public class Client{

    private final ByteBuffer sendBuffer=ByteBuffer.allocate(1024);
    private final ByteBuffer receiveBuffer=ByteBuffer.allocate(1024);
    private Selector selector;
    private SocketChannel socketChannel;

    public Client()throws IOException{
        this.socketChannel = SocketChannel.open();
        this.socketChannel.connect(new InetSocketAddress(InetAddress.getLocalHost(),8080));
        this.socketChannel.configureBlocking(false);
        System.out.println("连接建立成功");
        this.selector=Selector.open();
        this.socketChannel.register(selector,SelectionKey.OP_READ);
    }

    public static void main(String[] args) throws Exception{
        final Client client=new Client();
        Thread sendMsg=new Thread(client::sendInputMsg);
        sendMsg.start();

        client.start();
    }

    private void start()throws IOException {
        while (selector.select() > 0 ){

            Iterator<SelectionKey> it = selector.selectedKeys().iterator();
            while (it.hasNext()){
                SelectionKey key = it.next();
                it.remove();

                if (key.isReadable()) {
                    System.out.println("isReadable");
                    receive(key);
                }

            }

        }
    }

    /**
     * 接收服务端发送的内容
     * @param key
     * @throws IOException
     */
    private void receive(SelectionKey key)throws IOException{
        SocketChannel socketChannel=(SocketChannel)key.channel();
        socketChannel.read(receiveBuffer);
        receiveBuffer.flip();
        String receiveData=Charset.forName("UTF-8").decode(receiveBuffer).toString();

        System.out.println("receive server message:"+receiveData);
        receiveBuffer.clear();
    }

    /**
     * 发送控制台输入内容至服务器
     */
    private void sendInputMsg() {
        BufferedReader bufferedReader=new BufferedReader(new InputStreamReader(System.in));
        try{
            String msg;
            while ((msg = bufferedReader.readLine()) != null){
                synchronized(sendBuffer){
                    sendBuffer.put((msg+"\r\n").getBytes());
                    sendBuffer.flip();
                    while(sendBuffer.hasRemaining()){
                        socketChannel.write(sendBuffer);
                    }
                    sendBuffer.compact();

                }
            }
        }catch(Exception e){
            e.printStackTrace();
        }
    }
}
```



# 参考

[Java NIO Selector](http://tutorials.jenkov.com/java-nio/overview.html)

[Java NIO之Selector（选择器）](https://juejin.im/post/5afad7b1f265da0b7f44b60b)