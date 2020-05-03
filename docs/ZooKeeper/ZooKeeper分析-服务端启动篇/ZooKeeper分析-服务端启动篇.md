## 前言
今年五一认真整理以前凌凌散散的学习内容，发现虽然看得多，但是如果不去看看源码，真的是不深刻，没什么感觉，这也是为什么要重新整理ZooKeeper的相关学习笔记了，在前面的时候我们介绍ZooKeeper的一些重要的概念，在整体上对ZooKeeper 有了一个简单的认识，下面我们再从服务端的启动流程来了解了解，本小节不会将集群的选举过程，这个将放在下一篇文章中介绍，本节知识整体了解一下服务端的启动运作流程。

ZooKeeper 系列文章 会讲一些重要的功能和概念，主要包括：

- [重要概念介绍 - 整体视图的认识](http://blog.ztgreat.cn/article/90)
- 服务端启动流程
- 集群选举过程
- 会话管理
- 读写请求操作
- 数据与存储

本节主要 讲一下 服务端启动流程,对于前面内容可以点击相关链接进行跳转。

## 概览
以 `3.6.1` 版本作为分析。主要从服务端，结合部分源码,来了解 ZooKeeper的启动流程。

ZooKeeper 服务的启动方式分为三种，即单机模式、伪分布式模式、分布式模式。本章节主要研究分布式模式的启动模型，其主要要经过 Leader 选举，集群数据同步，启动服务器（走马观花的看）。

分布式模式下的启动过程主要包括如下阶段，

- 解析 config 文件；
- 本地文件数据恢复；
- Leader选举；
- 其它服务器和Leader数据同步；
- 初始化 ZooKeeperServer 进入各自角色服务；

### 整体架构

首先 我们先来看看ZooKeeperServer服务端的整体架构，如下图所示：

![ZooKeeperServer服务端整体架构](http://img.blog.ztgreat.cn/document/zookeeper/4871751-92c0da6dc6ea8287.png)

看不懂也没关系，先了解一下大概的内容，后面会细说里面的东西，下面简单介绍一下里面的名词和功能

»  ZooKeeper Client : 代表的客户端,本次我们主要看服务端，因此客户端的东西就省略了.

» ServerCnxnFactory : ZooKeeper中使用ServerCnxnFactory管理与客户端的连接,其有两个实现

一个是NIOServerCnxnFactory,使用Java原生NIO实现;

一个是NettyServerCnxnFactory,使用netty实现;使用ServerCnxn代表一个客户端与服务端的连接.

» Session Tracker: 服务端对会话管理器，负责会话的创建，管理和清理等工作.

» RequestProcessor:  ZooKeeper的请求处理链(责任链模式).

» LearnerCnxAcceptor: 用于负责接收所有非Leader服务器的连接请求

>用 Learner 指代 除Leader 服务器以外的其余的服务器

» LearnerHandler: 负责Leader 和 **某个Learner**服务器之间的消息通信和数据同步

>Leader 接收到来自其他机器的连接创建请求后，会创建一个 LearnerHandler实例，每个LearnerHandler实例都对应了一个Leader 和Learner服务器之间的连接

» ZKDatabase : 内存数据库，负责管理ZooKeeper的所有会话，数据存储和事务日志.

>ZKDatabase 会定时向磁盘dump快照数据，同时在ZooKeeper服务器启动的时候，会通过磁盘的事务日志和快照数据文件恢复成一个完整的内存数据库.

» FileTxnSnapLog：事务日志和快照数据访问层，用于衔接上层业务和底层数据存储

>底层数据包含了事务日志和快照数据两部分，因此 FileTxnSnapLog 内部又分为FileTxnLog和FileSnap，分别负责事务日志管理器和快照数据管理器

» Leader Election：封装了 Leader 选举算法的逻辑.



## 集群模式启动

### 流程图

![启动流程图](http://img.blog.ztgreat.cn/document/zookeeper/5871751-92c0da6dc6ea8287.png)



入口在 `QuorumPeerMain#main`

### 配置文件解析

```
QuorumPeerConfig config = new QuorumPeerConfig();
if (args.length == 1) {
	config.parse(args[0]);
}
```

解析配置文件zoo.cfg ，该文件配置了ZooKeeper 运行时的基本参数，包括 tickTime,dataDir和clientPort等参数信息，这里就不多讲了。

### 历史文件清理器

```
// Start and schedule the the purge task
DatadirCleanupManager purgeMgr = new DatadirCleanupManager(
	config.getDataDir(),
	config.getDataLogDir(),
	config.getSnapRetainCount(),
	config.getPurgeInterval());
purgeMgr.start();
```

从3.4.0 版本开始，ZooKeeper 增加了自动清理历史数据文件的机制，包括对事物日志和快照数据文件进行定时清理。

### 初始化 QuorumPeer

 QuorumPeer 是集群模式下特有的对象，ZooKeeper实例的托管者，从集群层面看，代表了ZooKeeper集群中的一台机器，在运行期间，QuorumPeer 会不断的检测当前服务器实例的运行状态，同时根据情况发起Leader选举。

>```
>QuorumPeer 是 Thread 的子类
>```

下面贴了部分代码，删减了部分，方便后面对照看，建议读者对照源码阅读

```
// 管理与客户端的连接
ServerCnxnFactory cnxnFactory = null;
ServerCnxnFactory secureCnxnFactory = null;

if (config.getClientPortAddress() != null) {
	cnxnFactory = ServerCnxnFactory.createFactory();
	cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), 		config.getClientPortListenBacklog(), false);
}


quorumPeer = getQuorumPeer();
/**
 * 创建 ZooKeeper 数据管理器 FileTxnSnapLog
 * FileTxnSnapLog 是ZooKeeper上层服务器和底层数据存储之间的对接层
 * 提供了一列操作数据文件的接口，包括事务日志文件和快照数据文件
 * ZooKeeper根据zoo.cfg 文件解析出来的快照数据目录 dataDir
 * 和事务日志目录dataLogDir 来创建FileTxnSnapLog
 */
quorumPeer.setTxnFactory(new FileTxnSnapLog(config.getDataLogDir(), config.getDataDir()));
quorumPeer.enableLocalSessions(config.areLocalSessionsEnabled());
quorumPeer.enableLocalSessionsUpgrading(config.isLocalSessionsUpgradingEnabled());
quorumPeer.setElectionType(config.getElectionAlg());
// 设置 myId(serverId)
quorumPeer.setMyid(config.getServerId());
quorumPeer.setTickTime(config.getTickTime());
quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
quorumPeer.setInitLimit(config.getInitLimit());
quorumPeer.setSyncLimit(config.getSyncLimit());
quorumPeer.setConnectToLearnerMasterLimit(config.getConnectToLearnerMasterLimit());
quorumPeer.setObserverMasterPort(config.getObserverMasterPort());
quorumPeer.setConfigFileName(config.getConfigFilename());
quorumPeer.setClientPortListenBacklog(config.getClientPortListenBacklog());
// 设置内存数据库
quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
quorumPeer.setQuorumVerifier(config.getQuorumVerifier(), false);
if (config.getLastSeenQuorumVerifier() != null) {
	quorumPeer.setLastSeenQuorumVerifier(config.getLastSeenQuorumVerifier(), false);
}
quorumPeer.initConfigInZKDatabase();
quorumPeer.setCnxnFactory(cnxnFactory);
quorumPeer.setSecureCnxnFactory(secureCnxnFactory);
quorumPeer.setSslQuorum(config.isSslQuorum());
quorumPeer.setUsePortUnification(config.shouldUsePortUnification());
// 设置角色，如果配置文件中指定了是Observer 的话，则会Observer 否则就是 PARTICIPANT(参与者)
quorumPeer.setLearnerType(config.getPeerType());
quorumPeer.setSyncEnabled(config.getSyncEnabled());
quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());
if (config.sslQuorumReloadCertFiles) {
	quorumPeer.getX509Util().enableCertFileReloading();
}
quorumPeer.initialize();
// 主要关注这个方法
quorumPeer.start();
ZKAuditProvider.addZKStartStopAuditLog();
quorumPeer.join();
```



#### 创建ServerCnxnFactory

ZooKeeper中使用ServerCnxnFactory管理与客户端的连接,其有两个实现,

一个是NIOServerCnxnFactory,使用Java原生NIO实现;

一个是NettyServerCnxnFactory,使用netty实现;使用ServerCnxn代表一个客户端与服务端的连接.

从名字我们可以知道 ServerCnxnFactory 是一个工厂类，负责管理 ServerCnxn，

ServerCnxn 这个类代表了一个客户端与一个 server 的连接，每个客户端连接过来都会被封装成一个 ServerCnxn 实例用来维护了服务器与客户端之间的 Socket 通道。

```
/**
 * Interface to a Server connection - represents a connection from a client
 * to the server.
 */
public abstract class ServerCnxn implements Stats, Watcher {
 ...
}
```

关于Netty 和NIo的封装，读者可以自己阅读，毕竟工作中很少用到，就算用的话，都是经过封装好的，看看别人怎么使用的，对自己也是大有裨益。

我们接着看 `quorumPeer.start()`方法

#### 恢复本地数据

每次在ZooKeeper启动的时候，都需要从本地快照数据文件和事务日志文件中进行数据恢复，更多的内容我们放到后面的数据存储章节来分析。

```
@Override
public synchronized void start() {
	if (!getView().containsKey(myid)) {
		throw new RuntimeException("My id " + myid + " not in the peer list");
	}
	// 恢复本地文件数据
	loadDataBase();
	startServerCnxnFactory();
	try {
		adminServer.start();
	} catch (AdminServerException e) {
		LOG.warn("Problem starting AdminServer", e);
		System.out.println(e);
}
	startLeaderElection();
	startJvmPauseMonitor();
	super.start();
}
```

#### 启动网络IO管理器

server端要启动的factory有两个，cnxnFactory和securityCnxnFactory。

每个Factory的实现均有两种方式，`Netty`和 `NIO`方式。前面说过ServerCnxn是个处理网络I/O的组件，需要建立socket通道。这两种方式就是建立网络socket的不同方式

```
private void startServerCnxnFactory() {
	if (cnxnFactory != null) {
		cnxnFactory.start();
	}
	if (secureCnxnFactory != null) {
		secureCnxnFactory.start();
	}
}
```

这里已NettyServerCnxnFactory 为例，简单看一下代码内容：

```
@Override
public void start() {
	if (listenBacklog != -1) {
		bootstrap.option(ChannelOption.SO_BACKLOG, listenBacklog);
	}
	LOG.info("binding to port {}", localAddress);
	parentChannel = bootstrap.bind(localAddress).syncUninterruptibly().channel();
	// Port changes after bind() if the original port was 0, update
	// localAddress to get the real port.
	localAddress = (InetSocketAddress) parentChannel.localAddress();
	LOG.info("bound to port {}", getLocalPort());
}
```

主要做的其实就是绑定服务端端口。

adminServer 是一个内嵌的web 服务，提供简单的命令管理，这里我们不用管。

#### 准备Leader选举

```
public synchronized void startLeaderElection() {
	try {
		if (getPeerState() == ServerState.LOOKING) {
			currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
		}
	} catch (IOException e) {
		...
}
	this.electionAlg = createElectionAlgorithm(electionType);
}
```

这里里面做的主要就是为Leader 选举做准备，包括创建初始投票信息已经选举算法，同时在这里还会创建相关的队列和线程 为 选举中消息交换 做准备。

```
// 代码略有删减
protected Election createElectionAlgorithm(int electionAlgorithm) {
	Election le = null;
    // 选举网络IO管理器
	QuorumCnxManager qcm = createCnxnManager();
	QuorumCnxManager oldQcm = qcmRef.getAndSet(qcm);
	if (oldQcm != null) {
		LOG.warn("Clobbering already-set QuorumCnxManager (restarting leader election?)");
		oldQcm.halt();
	}
	QuorumCnxManager.Listener listener = qcm.listener;
	if (listener != null) {
	    // 启动对选举端口的监听，等待集群中其他服务器创建连接
	    // listener 是 Thread 子类，相关功能会委托给 ListenerHandler
        // 相关代码可以自己阅读 Listener和ListenerHandler 中的run 方法
		listener.start();
		// 创建选举算法
		FastLeaderElection fle = new FastLeaderElection(this, qcm);
		// 开启选举算法，还没有开始选举
		fle.start();
		le = fle;
	} else {
		LOG.error("Null listener when initializing cnx manager");
	}
	return le;
}
```

目前 在ZooKeeper 中 只支持FastLeaderElection选举算法（以前还有两种，被废弃了）。

ZooKeeper 会首先创建选举锁需要的网络IO层QuorumCnxManager,同时启动对选举端口的监听，等待集群中其他服务器创建连接。

>QuorumCnxManager 是用于服务端选举过程中 处理网络IO的一个管理器，每台服务器启动的时候
>
>都会启动一个QuorumCnxManager，负责各台服务器之间的底层Leader选举过程中的网络通信

##### QuorumCnxManager

在QuorumCnxManager 这个类内部维护了一系列的队列，用于保存接收到的，待发送的消息，以及消息的发送器。除接收队列以外，这里提到的所有队列都有一个共同点―按SID分组形成队列集合，我们已发送队列为例说明这个分组的概念。假设集群中自身外还有4台机器，那么当前服务器就会为这4台服务器分别创建一个发送队列，互不干扰。

> 主要相关代码 都可以通过 `listener.start()`和`fle.start()`为入口进行阅读
>
> SID 用来标识一台ZooKeeper 集群中的机器，每台机器不能重复，和myId的值一致

###### 消息队列

- recvQueue: 消息接收队列，用于存放哪些从其他服务器接收到的消息
- queueSendMap:消息发送队列，用于保存那些待发送的消息。queueSendMap 是一个Map,按照SID进行分组，分别为集群中的每台机器分配了一个单独队列，从而保证各机器之间的消息发送互补影响。
- senderWorkerMap:发送器集合。每个SendWorker消息发送器，都对应一台远程ZooKeeper服务器，负责消息的发送。同样，在senderWorkerMap 中，也按照SID进行了分组
- lastMessageSent:最近发送过的消息。在这个集合中，为每个SID保留最近发送过的一个消息。



```
/*
 * 接收队列的最大容量
 * Maximum capacity of thread queues
 */
static final int RECV_CAPACITY = 100;

// 对应每个远程服务器发送队列的容量
// Initialized to 1 to prevent sending
// stale notifications to peers
static final int SEND_CAPACITY = 1;

/*
 * Listener thread
 */
public final Listener listener;

/*
 * Mapping from Peer to Thread number
 */
final ConcurrentHashMap<Long, SendWorker> senderWorkerMap;
final ConcurrentHashMap<Long, BlockingQueue<ByteBuffer>> queueSendMap;
final ConcurrentHashMap<Long, ByteBuffer> lastMessageSent;

/*
 * Reception queue
 */
public final BlockingQueue<Message> recvQueue;
    
```

值得一提的是 QuorumCnxManager中的消息队列 是一个阻塞队列，该阻塞队列是ZooKeeper 自己实现的

```
public class CircularBlockingQueue<E> implements BlockingQueue<E> {

  private static final Logger LOG = LoggerFactory.getLogger(CircularBlockingQueue.class);

  /** Main lock guarding all access */
  private final ReentrantLock lock;

  /** Condition for waiting takes */
  private final Condition notEmpty;

  /** The array-backed queue */
  private final ArrayDeque<E> queue;

  private final int maxSize;

  private long droppedCount;

  public CircularBlockingQueue(int queueSize) {
    this.queue = new ArrayDeque<>(queueSize);
    this.maxSize = queueSize;

    this.lock =  new ReentrantLock();
    this.notEmpty = this.lock.newCondition();
    this.droppedCount = 0L;
  }
  ...
}
```

内部是用双端队列+锁来实现阻塞队列的，这个阻塞队列和java 里面的阻塞队列不一样，如果队列满了，ZooKeeper 是会丢掉队头的元素的，更多的可以自己查阅相关源代码。

![quorumcnxManager交互示意图](http://img.blog.ztgreat.cn/document/zookeeper/6871751-92c0da6dc6ea8287.png)



###### 创建连接

为了能进行互相投票，ZooKeeper集群中的所有机器都要两两建立起网络连接。

QuorumCnxManager 在启动的时候，会创建一个 ServerSocket来监听Leader选举端口，开启端口监听过后，

ZooKeeper 就能够不断得接收到来自其他服务器的 "创建连接"请求，在接收到其他服务器的TCP连接请求时，会交由`receiveConnection`函数来处理。

为了避免两台服务器之间重复得创建TCP连接，ZooKeeper 设计了一个建立TCP连接的规则：

只允许SID大的服务器主动和其他服务器建立连接，否则端口连接。服务器通过对比自己和远程服务器的SID值，来判断是否接受连接请求。如果当前服务器发现自己的SID值更大，那么会断开当前连接，然后自己主动去和远程服务器建立连接。



```
/**
 * If this server receives a connection request, then it gives up on the new
 * connection if it wins. Notice that it checks whether it has a connection
 * to this server already or not. If it does, then it sends the smallest
 * possible long value to lose the challenge.
 * 
 */
public void receiveConnection(final Socket sock) {
	DataInputStream din = null;
	try {
		din = new DataInputStream(new BufferedInputStream(sock.getInputStream()));
		handleConnection(sock, din);
	} catch (IOException e) {
 		sock.getRemoteSocketAddress());
		closeSocket(sock);
	}
}
```

一旦建立起连接，就会根据远程服务器的SID 来创建相应的消息发送器`SendWorker`和消息接收器`RecvWorker`，并启动他们。

```
SendWorker sw = new SendWorker(sock, sid);
RecvWorker rw = new RecvWorker(sock, din, sid, sw);
sw.setRecv(rw);

SendWorker vsw = senderWorkerMap.get(sid);

if (vsw != null) {
	vsw.finish();
}

senderWorkerMap.put(sid, sw);

queueSendMap.putIfAbsent(sid, new CircularBlockingQueue<>(SEND_CAPACITY));

sw.start();
rw.start();
```



###### 消息接收与发送

消息的接收过程是由消息接收器`RecvWorker` 来负责的，在上面我们已经说到ZooKeeper 会为每个远程服务器分配一个单独的 RecvWorker，因此，每个RecvWorker 只需要不断得从这个TCP连接中读取消息，并将其保存到recvQueue队列中。

消息的发送过程也是同理的，在SendWorker 的具体实现中，有一个细节需要我们注意一下：

一旦SendWorker 退出后(连接断了或者其它原因)，再重新连接后 ，如果发送队列为空，那么这个时候就需要从lastMessageSent中取出一个最近发送过的消息来进行再次发送。这个细节的处理主要是为了解决这―类分布式问题：接收方在消息接收前，或者是在接收到消息后服务器挂掉了，导致消息尚未被正确处理。那么如此重复发送是否会导致其他问题呢？当然。这里可以放心的一点是，ZooKeeper 能够保证接收方在处理消息的时候，会对重复消息进行正确的处理。



>我看有点书上或者网上说的是，一旦ZooKeeper 发现针对当前远程服务器的消息发送队列为空，就会发送 
>
>lastMessageSent，但是看了代码后，发现 这部分代码在入口里面，不是在发送消息的主循环里面，因此个人感觉这种说法有问题，具体的可以自己阅读相关代码。



```
/**
 * If there is nothing in the queue to send, then we
 * send the lastMessage to ensure that the last message
 * was received by the peer. The message could be dropped
 * in case self or the peer shutdown their connection
 * (and exit the thread) prior to reading/processing
 * the last message. Duplicate messages are handled correctly
 * by the peer.
 *
 * If the send queue is non-empty, then we have a recent
 * message than that stored in lastMessage. To avoid sending
 * stale message, we should send the message in the send queue.
 */
BlockingQueue<ByteBuffer> bq = queueSendMap.get(sid);
	if (bq == null || isSendQueueEmpty(bq)) {
		ByteBuffer b = lastMessageSent.get(sid);
	if (b != null) {
		LOG.debug("Attempting to send lastMessage to sid={}", sid);
		send(b);
	}
}
```

在 大概了解 了 `startLeaderElection()` 代码后，我们继续看`QuorumPeer#start` 最后的start 方法



### 服务准备启动

因为代码比较多，我截取了部分关键代码，方便阅读

```
/**
 * 不停得检测当前服务器的状态
 * 根据服务器的状态作出相应的处理
 */
while (running) { 
    // 检测当前服务器状态
	switch (getPeerState()) {
		//当前服务处于looking状态
		case LOOKING:
		//省略无关代码
		......
		try {
			roZkMgr.start();
			reconfigFlagClear();
			if (shuttingDownLE) {
				shuttingDownLE = false;
				//开始leader选举算法(为选取做准备，在上面我们有分析)
				startLeaderElection();
			}
			//选举leader
			setCurrentVote(makeLEStrategy().lookForLeader());
		} catch (Exception e) {
			setPeerState(ServerState.LOOKING);
		} finally {
			// If the thread is in the the grace period, interrupt
			// to come out of waiting.
			roZkMgr.interrupt();
			roZk.shutdown();
		}
	}
	break; 
	case OBSERVING:
	    // 设置服务器状态
		setObserver(makeObserver(logFactory));
		observer.observeLeader();
		......
	break;
	case FOLLOWING:
	    setFollower(makeFollower(logFactory));
        follower.followLeader();
		......
	break;
	case LEADING:
	    setLeader(makeLeader(logFactory));
        leader.lead();
        setLeader(null);
		......
	break;
```

从上面代码中我们知道，当QuorumPeer start 启动后，会不断的检测当前服务器的状态，并作出相应的处理。在正常情况下，ZooKeeper 服务器的状态 在 `LOOKING`,`LEADING` ,`FOLLOWING`,`OBSERVING` 之间切换。如果配置文件指定了当前服务器是 Observer 角色的话，那就不会参与选举，直接进入 `OBSERVING`状态。

否则 QuorumPeer 的初始化状态是LOOKING，开始进行Leader 选举,

#### Leader 选举

ZooKeeper 的Leader 选举过程，简单来讲，就是一个集群中所有的机器相互之间进行一系列的投票，选举产生最合适的机器成为Leader,同时其余的机器成为Follower 或少 Observer的集群机器角色初始化过程。

关于Leader 选举算法，简而言之，就是集群中哪个机器处理的数据越新，其越有可能成为Leader。当然如果集群中的所有机器处理的数据都是一致的话，那么SID 最大的服务器将成为Leader。关于详细的选举过程，我们后面再细说。

当选举结束以后，服务器会首先判断当前被过半服务器认可的投票所对应的服务器是否是自己，如果是自己的话。那么就会将自己的服务器状态更新为`LEADING`。如果不是的话，那么就会根据情况来确定自己是`FOLLOWING`还是`OBSERVING`(**Observer 机器不参与投票**，因此不是Leader的话，正常情况下就是Follower)。



```
protected Leader makeLeader(FileTxnSnapLog logFactory) throws IOException, X509Exception{
    return new Leader(this, new LeaderZooKeeperServer(logFactory, this, this.zkDb)); 
}
protected Follower makeFollower(FileTxnSnapLog logFactory) throws IOException {
	return new Follower(this, new FollowerZooKeeperServer(logFactory, this, this.zkDb));
}

protected Observer makeObserver(FileTxnSnapLog logFactory) throws IOException {
	return new Observer(this, new ObserverZooKeeperServer(logFactory, this, this.zkDb));
}
```

Leader 服务器 需要构建 LeaderZooKeeperServer 和一个Leader类，同样的Follower 和 Observer 也是如此

##### Leader

Leader 服务器是整个ZooKeeper 集群工作机制中的核心，其主要工作有一下两个：

- 事务请求的唯一调度和处理者，保证集群事务处理的顺序性
- 集群内部各服务器的调度者

##### Follower

Follower 服务器是ZooKeeper 集群状态的跟随者，其主要工作有下面三个：

- 处理客户端非事务请求，转发事务请求给Leader 服务器
- 参与事务请求 Proposal 的投票
- 参与Leader 选举投票

##### Observer

Observer 服务器是ZooKeeper 集群状态的观察者，顾名思义 就是观察ZooKeeper 集群的最新状态变化并将这些状态同步过来。

Observer 服务器在工作原理上和Follower 基本是一致的，对于非事务请求，都可以进行独立的处理，而对于事务请求，则会转发给Leader 服务器进行处理。

>Observer 不参与任何形式的投票，包括事务请求Proposal的投票和Leader 的选举投票，通常用于在不影响集群事务处理鞥那里的前提下提升集群的非事务处理能力

#### 启动服务

在选举结束后，各自将会进入各自角色该进入的工作，也就是上面代码中的：

```
leader.lead();
follower.followLeader();
observer.observeLeader();
```

因为里面的代码比较多和复杂,先来看看大致的流程图：

![选举后](http://img.blog.ztgreat.cn/document/zookeeper/7871751-92c0da6dc6ea8287.png)



Leader 和Follower服务器启动期交互过程包括如下步骤：

1. 创建Leader 服务器和Follower 服务器

   完成Leader选举之后，每个服务器都会根据自己的服务器角色创建相应的服务器实例，并开始进入各自角色的主流程。

   ```
   leader.lead();
   follower.followLeader();
   observer.observeLeader();
   ```

   Learner 服务器开始和Leader 建立连接

   所有的Learner 服务器在启动完毕后，会从Leader 选举的投票结果中找到当前集群中的Leader 服务器，然后与其建立连接。

   相关代码 可以参考 `Follower#followLeader` 和 `Observer#observeLeader`

2. Leader 服务器 等待 非Leader 服务器连接

   在ZooKeeper 集群运行期间，Leader 服务器需要和所有其余的服务器(这里用"learner"来指代这类机器)保持连接以确定集群的机器存活情况。LearnerCnxAcceptor 接收器用于负责接收所有非Leader 服务器的连接请求。

   `org.apache.zookeeper.server.quorum.Leader#lead`

   ```
   
   // Start thread that waits for connection requests from
   // new followers.
   cnxAcceptor = new LearnerCnxAcceptor();
   cnxAcceptor.start();
   ```

   `LearnerCnxAcceptor` 是Thread的子类，相关的代码可以参考其run 方法

   在和 learner 建立连接后，会创建 `LearnerHandler` , LearnerHandler 是Leader 和其它 Learner 通信的处理器

   ```
   socket = serverSocket.accept();
   
   // start with the initLimit, once the ack is processed
   // in LearnerHandler switch to the syncLimit
   socket.setSoTimeout(self.tickTime * self.initLimit);
   socket.setTcpNoDelay(nodelay);
   
   BufferedInputStream is = new BufferedInputStream(socket.getInputStream());
   // 注意最后一个参数，传入的是 Leader,形参是 learnerMaster
   LearnerHandler fh = new LearnerHandler(socket, is, Leader.this);
   fh.start();
   ```

   相关代码可以 参考 `org.apache.zookeeper.server.quorum.LearnerHandler#run`

   Leader 接收到来自其他机器的连接创建请求后，会创建一个LearnerHandler实例。每个LearnerHandler 实例都对应了一个Leader 与Learner 服务器之间的连接，其负责Leader 和Learner 服务器之间几乎所有的消息通信和数据同步。

3. 向Leader 注册

   相关带 可以参考 `follower.followLeader()` ,`Learner#registerWithLeader`。

   当和Leader 建立起连接收，Learner 就会开始向Leader 进行注册-所谓的注册，其实就是将Learner 服务器自己的基本信息发送给Leader 服务器，我们称之为**LearnerInfo**,包括当前服务器的SID和服务器处理的最新的ZXID(事务标识ID)

   >ZXID 是一个64位事务ID,用来标识一次服务器状态的变更。在某一个时刻，集群中每台机器的ZXID值都不一定全都一致。
   >
   >它高32位是epoch（ZAB协议通过epoch编号来 区分 Leader 周期变化的策略）用来标识 leader 关系是否 改变，每次一个 leader 被选出来，它都会有一个新的 epoch=（原来的epoch+1），标识当前属于那个leader的 统治时期。低32位用于递增计数。

   ```
   //org.apache.zookeeper.server.quorum.Follower#followLeader
   // 连接Leader
   connectToLeader(leaderServer.addr, leaderServer.hostname);
   connectionTime = System.currentTimeMillis();
   // 向Leader 发送 LearnerInfo 信息
   long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);
   ```

4. Leader 解析 Learner 信息，计算新的epoch

   相关代码可以参考 `LearnerHandler#run`

   Leader 服务器在接收到Learner 的基本信息后，会解析出该Learner 的SID 和ZXID,然后根据该Learner 的ZXID 解析出其对应的epoch_of_learer，和当前Leader 服务器的epoch_of_leader 进行比较，如果该Learner的epoch_of_learner更大的话，那么久更新Leader 的epoch;

   ```
   epoch_of_leader = epoch_of_learner + 1
   ```

   然后，LearnerHandler 会进行等待，直到过半的Learner 已经向Leader 进行了注册，同时更新了 epoch_of_leader 之后，Leader 就可以确定当前集群的epoch 了

   计算出新的epoch 之后，Leader 会将该信息以一个**LEADERINFO**消息的形式发送给Learner,同时等待Learner 的响应。



   Follower 在收到来自Leader 的LEADERINFO 消息后，会解析出epoch 和ZXID，然后向Leader 反馈一个ACKEPOCH响应。

   相关代码可以参考  `Learner#registerWithLeader`

5. 数据同步

   Leader 服务器接收到Learner 的这个ACK消息后，就可以开始与其进行数据同步了。

   相关代码可以参考 `LearnerHandler#run`

6. 启动Leader 和Learner 服务器

   当有过半的Learner 已经完成了数据同步，那么Leader 和Learner 服务器实例就可以开始正式启动了，其主要有 创建并**启动会话管理器**，初始化ZooKeeper 的请求处理链

   ```
   
   //org.apache.zookeeper.server.ZooKeeperServer#startup
   
   public synchronized void startup() {
   
   	// 这里会创建会话管理器
   	if (sessionTracker == null) {
   		createSessionTracker();
   	}
   	startSessionTracker();
   	setupRequestProcessors();
   
   	...
   }
   ```

   会话管理器会在这里创建，关于会话相关的我们在后面再来介绍，这里先知道大概就可以了。

   至此，集群版的ZooKeeper 服务器启动完毕。



## 总结

我们在这篇文章中一起了解了  Zookeeper 集群版 服务端的启动流程，对于启动过程中的任何一步，我们都没有过多的深入去细究。目的 在于了解一个整体的流程，读者在阅读的过程中，最好能参照源码，自己简单过一遍。

对于里面的一些细究 包括选举，会话管理，数据存储等等，这个我们在后面再来详细介绍。

## 参考

 《从PAXOS到ZOOKEEPER分布式一致性原理与实践》

 《ZooKeeper：分布式过程协同技术详解》



