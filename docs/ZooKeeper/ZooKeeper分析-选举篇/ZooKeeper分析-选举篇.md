## 前言
前面几篇文章讲了整体概念，服务器启动的流程，对于选举过程是一笔带过，我们已经了解了ZooKeeper 集群中的三种服务器角色：Leader,Follower 和Observer,本篇接下来主要讲述Leader选举的相关流程。

我们从选举算法概述、服务器启动Leader选举详细过程两个方面探讨实现细节。

>仅仅分析选举流程 还是比较简单的

ZooKeeper 系列文章 会讲一些重要的功能和概念，主要包括：

- [重要概念介绍 - 整体视图的认识](http://blog.ztgreat.cn/article/90)
- [服务端启动流程](http://blog.ztgreat.cn/article/91)
- 集群选举过程
- 会话管理
- 读写请求操作
- 数据与存储

本节主要 讲一下 Leader选举流程,对于前面内容可以点击相关链接进行跳转。

## 选举算法概述

ZooKeeper 的Leader 选举过程，简单来讲，就是一个集群中所有的机器相互之间进行一系列的投票，选举产生最合适的机器成为Leader,同时其余的机器成为Follower 或少 Observer的集群机器角色初始化过程。

关于Leader 选举算法，简而言之，**就是集群中哪个机器处理的数据越新，其越有可能成为Leader。当然如果集群中的所有机器处理的数据都是一致的话，那么SID 最大的服务器将成为Leader**.具体的判断逻辑，可以看后面的选票PK内容。

>SID 用来标识一台ZooKeeper 集群中的机器，每台机器不能重复，和myId的值一致
>
>ZXID 是一个64位事务ID,用来标识一次服务器状态的变更。在某一个时刻，集群中每台机器的ZXID值都不一定全都一致。
>
>它高32位是epoch（ZAB协议通过epoch编号来 区分 Leader 周期变化的策略）用来标识 leader 关系是否 改变，每次一个 leader 被选出来，它都会有一个新的 epoch=（原来的epoch+1），标识当前属于那个leader的 统治时期。低32位用于递增计数。

选举的条件是集群中至少有两台机器。主要流程如下：

1. 每个server发出一个投票
2. 接收各个服务器的投票消息
3. 处理投票
4. 统计投票
5. 改变服务器状态

QuorumPeer.OrderState定义了服务器的四种状态，分表是：

- **LOOKING：** 寻找Leader状态，服务器处于该状态时，表示集群中没有leader，需要进入leader选举流程。
- **FOLLOWING：** 跟随者状态，表明当前服务器角色是Follower
- **LEADER：** 领导者状态，表明当前服务器角色是leader
- **OBSERVING：** 观察者状态

服务器启动时的Leader 选举和服务器运行期间的Leader 选举 基本是一样的，这里我们主要分析服务启动时的Leader选举流程。

## Leader选举

服务器启动Leader选举只在集群模式启动时触发。根据上一篇文章
[ZooKeeper分析-服务端启动流程](http://blog.ztgreat.cn/article/91)，描述了项目启动过程，在执行`QuerumPeer` start()方法时会触发选举逻辑。

### 选举流程图

同样的我们先来看一下整体的流程图，方便梳理有个整体的认识。

![选举流程图](http://img.blog.ztgreat.cn/document/zookeeper/9271751-92c0da6dc6ea8287.png)

其实也就是`FastLeaderElection#lookForLeader` 方法的逻辑。

### 初始化选举算法

`QuerumPeer` super.start()方法中会开始选举过程(相关代码在QuerumPeer. run 中)。

```
 @Override
public synchronized void start() {
    if (!getView().containsKey(myid)) {
        throw new RuntimeException("My id " + myid + " not in the peer list");
    }
    loadDataBase();
    startServerCnxnFactory();
    try {
        adminServer.start();
    } catch (AdminServerException e) {
        LOG.warn("Problem starting AdminServer", e);
        System.out.println(e);
    }
    // 设置选举算法，初始化相关工作
    // 初始选举算法，和 选举网络IO管理器
    startLeaderElection();
    super.start();
}
```

在 `startLeaderElection()` 中，根据`electionAlg`来决定实现哪种选举算法，这个参数是在zoo.cfg配置文件中配置的。`electionAlg`的值有0，1，2，3四种。3为TCP版本的`FastLeaderElection`。

3.4.0之后的版本都推荐使用`FastLeaderElection`模式，下面主要讲这种实现。

同时还会生成初始投票(初始投票投自己)

### 投票数据结构

投票的数据结构如下：

```
//相关代码可以参考 org.apache.zookeeper.server.quorum.Vote

//被推举的Leader的SID值
private final long id;

//被推举的Leader的事务ID
private final long zxid;

//选举轮次
private final long electionEpoch;

//被推举的Leader的epoch
private final long peerEpoch;

//当前服务器的状态
private final ServerState state;

//选举的版本
private final int version;
```

只有当server state为LOOKING状态是才触发选举过程。

```
if (getPeerState() == ServerState.LOOKING) {
    // 初始投票，投自己
    currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
}
```

生成投票的规则为首次都选择自己作为Leader进行投票，传入`myid`,`zxid`,`epoch`值。分表代表机器编号ID，事务ID，当前的轮询次数。生成的投票结果再下面会用到，会作为message发给其他机器。

### 初始化QuorumCnxManager

```
//org.apache.zookeeper.server.quorum.QuorumPeer#createElectionAlgorithm
QuorumCnxManager qcm = createCnxnManager();
```

在上篇文章中，我们介绍过 QuorumCnxManager：

QuorumCnxManager 是用于服务端选举过程中 处理网络IO的一个管理器，每台服务器启动的时候

都会启动一个QuorumCnxManager，负责各台服务器之间的底层Leader选举过程中的网络通信

在QuorumCnxManager 这个类内部维护了一系列的队列，用于保存接收到的，待发送的消息，以及消息的发送器。除接收队列以外，这里提到的所有队列都有一个共同点―按SID分组形成队列集合，我们已发送队列为例说明这个分组的概念。假设集群中自身外还有4台机器，那么当前服务器就会为这4台服务器分别创建一个发送队列，互不干扰。

![QuorumCnxManager交互示意图](http://img.blog.ztgreat.cn/document/zookeeper/6871751-92c0da6dc6ea8287.png)



####  初始化监听器

如果选举网络I/O管理器创建成功，需要注册一个监听器，监听器里维护着两个线程，进行消息发送和接收

```
QuorumCnxManager.Listener listener = qcm.listener;
if (listener != null) {
    listener.start();
    FastLeaderElection fle = new FastLeaderElection(this, qcm);
    // 初始化/开启选举算法
    fle.start();
    le = fle;
} else {
    LOG.error("Null listener when initializing cnx manager");
}
```

`listener.start()` 后续 会进行 注册端口，监听事件，部分功能委托 `ListenerHandler`,这部分可以自行阅读。

#### 初始化选举算法

在初始化监听器后，紧接着会开启 选举算法:

```
FastLeaderElection fle = new FastLeaderElection(this, qcm);
fle.start();
```

下面的代码只截图了部分，为了方便里面，先来看看FastLeaderElection 部分数据结构：

![](http://img.blog.ztgreat.cn/document/zookeeper/9071751-92c0da6dc6ea8287.png)

FastLeaderElection 中有两个**阻塞队列**(上图中没有画出来)：sendqueue，recvqueue 用于收发消息。

收发消息委托给了 Messenger 类，而 Messenger 有两个子类，WorkerSender和WorkerReceiver 分别来承担这两个工作，更多的可以阅读相关源码。

![选举相关数据结构示意图](http://img.blog.ztgreat.cn/document/zookeeper/9171751-92c0da6dc6ea8287.png)



```
// FastLeaderElection 构造函数
public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager) {
	this.stop = false;
	this.manager = manager;
	// 初始化相关环境
	starter(self, manager);
}


private void starter(QuorumPeer self, QuorumCnxManager manager) {
	this.self = self;
	proposedLeader = -1;
	proposedZxid = -1;
	
	// 消息发送队列
	sendqueue = new LinkedBlockingQueue<ToSend>();
	// 消息接收队列
	recvqueue = new LinkedBlockingQueue<Notification>();
	this.messenger = new Messenger(manager);
}

/**
* This method starts the sender and receiver threads.
*/
public void start() {
	this.messenger.start();
}


// 消息发送 worker
WorkerSender ws;
// 消息接收 worker
WorkerReceiver wr;

// 消息发送线程
Thread wsThread = null;

// 消息接收线程
Thread wrThread = null;

Messenger(QuorumCnxManager manager) {

	this.ws = new WorkerSender(manager);

	this.wsThread = new Thread(this.ws, "WorkerSender[myid=" + self.getId() + "]");
	this.wsThread.setDaemon(true);

	this.wr = new WorkerReceiver(manager);

	this.wrThread = new Thread(this.wr, "WorkerReceiver[myid=" + self.getId() + "]");
	this.wrThread.setDaemon(true);
}
```



### 建立连接

当接收到连接后，会初始化进行相关工作(是否主动发起连接，或者初始化消息发送和接收线程)。

>为了避免两台服务器之间重复得创建TCP连接，ZooKeeper 设计了一个建立TCP连接的规则：
>
>只允许SID大的服务器主动和其他服务器建立连接，否则端口连接。服务器通过对比自己和远程服务器的SID值，来判断是否接受连接请求。如果当前服务器发现自己的SID值更大，那么会断开当前连接，然后自己主动去和远程服务器建立连接。
>
>这个我们在服务启动篇也提及过这个

相关流程的代码可以参考：

`QuorumCnxManager#handleConnection`

```
//If wins the challenge, then close the new connection.
if (sid < self.getId()) {
    /*
     * This replica might still believe that the connection to sid is
     * up, so we have to shut down the workers before trying to open a
     * new connection.
     */
    SendWorker sw = senderWorkerMap.get(sid);
    if (sw != null) {
        sw.finish();
    }

    /*
     * Now we start a new connection
     */
    LOG.debug("Create new connection to server: {}", sid);
    // 关闭连接
    closeSocket(sock);

	// 主动建立连接
    if (electionAddr != null) {
        connectOne(sid, electionAddr);
    } else {
        connectOne(sid);
    }

} else { 
    // Otherwise start worker threads to receive data.
    SendWorker sw = new SendWorker(sock, sid);
    RecvWorker rw = new RecvWorker(sock, din, sid, sw);
    sw.setRecv(rw);

    SendWorker vsw = senderWorkerMap.get(sid);

    if (vsw != null) {
        vsw.finish();
    }

    senderWorkerMap.put(sid, sw);

    queueSendMap.putIfAbsent(sid,
            new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY));

    sw.start();
    rw.start();
}
```

`SendWorker`主要职责是发送消息给集群中其他机器，在其他机器存活的情况下，他会第一时间发送请求。如果队列中没有消息发送，他会发送lastMessage，确认集群中其他机器都收到了消息。

`RecvWorker`用来接收消息，监听socket端口。如果channel关闭，RecvWorker会从线程池中移除。



当有建立连接后，就要正式开始选举了，其实也就是`FastLeaderElection#lookForLeader` 方法的逻辑。

当ZooKeeper 服务器检测到当前服务器状态变成LooKING时，就会触发Leaderx选举，即调用`lookForLeader`方法来进行选举。



### 自增选举轮次

在FastLeaderElection 实现中，有一个logicalclock 属性，用于标识当前Leader 的选举轮次，ZooKeeper规定了所有有效的投票都必须在同一轮次中。ZooKeeper 在开始新一轮的投票时，会首先对logicalclock 进行自增操作。

```
synchronized (this) {
	logicalclock.incrementAndGet();
	// 初始化投票信息
	updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
}
// 发送初始化投票
sendNotifications();
```

### 初始化投票

在开始进行新一轮的投票之前，每个服务器都会首先初始化自己的选票，在初始化阶段，每台服务器都会将自己推选为Leader。

### 发送投票消息

在完成选票初始化后，服务器就会发起第一次投票。ZooKeeper会将刚刚初始化好的选票放入`sendqueue`队列中，由发送器`WorkerSender`负责发送出去。

`WorkerSender`的处理逻辑在`private void process(ToSend m)`中，它根据messge type来判断消息类型进行不同的处理。用switch case来处理。总共有四类不同的消息类型

- crequest：发起选举的请求信息
- challenge：选举信息
- notification：通知消息
- ack：响应消息

这里代码粗略看一下就可以了。

```
static enum mType {
            crequest, challenge, notification, ack
        }

ToSend(mType type, long tag, long leader, long zxid, long epoch,
        ServerState state, InetSocketAddress addr) {

    switch (type) {
    case crequest:
        this.type = 0;
        this.tag = tag;
        this.leader = leader;
        this.zxid = zxid;
        this.epoch = epoch;
        this.state = state;
        this.addr = addr;

        break;
    case challenge:
        this.type = 1;
        this.tag = tag;
        this.leader = leader;
        this.zxid = zxid;
        this.epoch = epoch;
        this.state = state;
        this.addr = addr;

        break;
    case notification:
        this.type = 2;
        this.leader = leader;
        this.zxid = zxid;
        this.epoch = epoch;
        this.state = QuorumPeer.ServerState.LOOKING;
        this.tag = tag;
        this.addr = addr;

        break;
    case ack:
        this.type = 3;
        this.tag = tag;
        this.leader = leader;
        this.zxid = zxid;
        this.epoch = epoch;
        this.state = state;
        this.addr = addr;

        break;
    default:
        break;
    }
}
```

case 0:构造一个选举开始请求给其他机器。

```
/*
 * Building challenge request packet to send
 */
requestBuffer.clear();
requestBuffer.putInt(ToSend.mType.crequest.ordinal());
requestBuffer.putLong(m.tag);
requestBuffer.putInt(m.state.ordinal());
zeroes = new byte[32];
requestBuffer.put(zeroes);
requestPacket.setLength(48);
requestPacket.setSocketAddress(m.addr);

if (challengeMap.get(m.tag) == null) {
    mySocket.send(requestPacket);
}
```

case 1: 发送选举信息给其他机器

```
/*
 * Building challenge packet to send
 */
long newChallenge;
ConcurrentHashMap<Long, Long> tmpMap = addrChallengeMap.get(m.addr); 
if(tmpMap != null){
    Long tmpLong = tmpMap.get(m.tag);
    if (tmpLong != null) {
        newChallenge = tmpLong;
    } else {
        newChallenge = genChallenge();
    }

    tmpMap.put(m.tag, newChallenge);

    requestBuffer.clear();
    requestBuffer.putInt(ToSend.mType.challenge.ordinal());
    requestBuffer.putLong(m.tag);
    requestBuffer.putInt(m.state.ordinal());
    requestBuffer.putLong(newChallenge);
    zeroes = new byte[24];
    requestBuffer.put(zeroes);
    requestPacket.setLength(48);
    requestPacket.setSocketAddress(m.addr);
    mySocket.send(requestPacket);   
}
```

case 2：构造通知消息去发送，有重试机制，最多重试`maxAttempts`次

case 3：发送ack消息

```
case 3:
    requestBuffer.clear();
    requestBuffer.putInt(m.type);
    requestBuffer.putLong(m.tag);
    requestBuffer.putInt(m.state.ordinal());
    requestBuffer.putLong(m.leader);
    requestBuffer.putLong(m.zxid);
    requestBuffer.putLong(m.epoch);
    requestPacket.setLength(48);
    try {
        requestPacket.setSocketAddress(m.addr);
    } catch (IllegalArgumentException e) {
    }
    try {
        mySocket.send(requestPacket);
    } catch (IOException e) {
        LOG.warn("Exception while sending ack: ", e);
    }
    break;
```

### 接收外部投票

每台服务器都会不断的从 recvqueue 队列中 获取外部投票。

`WorkerReceiver`也是根据消息类型来进行处理的。
当message type = 0时，表示其他机器发起了选举的请求，当前机器也会生成内部投票消息去发送。每台服务器都会不断的从recvqueue队列中获取外部投票，如果服务器无法获取任何外部投票时，会立即确认自己是否和集群中其他服务器保持着有效连接，如果没有建立连接，那么会马上建立连接，如果已经建立连接，那么就再次发送当前的内部投票

```
case 0:
    // Receive challenge request
    ToSend c = new ToSend(ToSend.mType.challenge, tag,
            current.getId(), current.getZxid(),
            logicalclock.get(), self.getPeerState(),
            (InetSocketAddress) responsePacket.getSocketAddress());
    sendqueue.offer(c);
    break;
```

type = 1时，接收其他机器发来的选举信息，保存到本地。是通过`challengeMap`来保存的，是个`ConcurrentHashMap`

```
case 1:
    // Receive challenge and store somewhere else
    long challenge = responseBuffer.getLong();
    saveChallenge(tag, challenge);
    break;
```

### 判断选举轮次

在处理外部投票的时候，会根据选举轮次来进行不同的处理

- 如果通知消息的选举轮次比本身的高，则更新自己的选举轮次，并接收通知中的选举信息作为自己的选举信息进行发送。然后把通知消息放入recvqueue中，生成的自身的选举消息放入sendqueue中。
- 如果外部投票的选举轮次小于内部投票，那么会忽略该外部投票，不做任何处理。
- 外部投票和内部投票选举轮次一致，则开始进行选票PK。

```
if ((myMsg.lastEpoch <= n.epoch)
        && ((n.zxid > myMsg.lastProposedZxid) 
        || ((n.zxid == myMsg.lastProposedZxid) 
        && (n.leader > myMsg.lastProposedLeader)))) {
    myMsg.lastProposedZxid = n.zxid;
    myMsg.lastProposedLeader = n.leader;
    myMsg.lastEpoch = n.epoch;
}

recvqueue.offer(n);
ToSend a = new ToSend(ToSend.mType.ack, tag,
        current.getId(), current.getZxid(),
        logicalclock.get(), self.getPeerState(),
        (InetSocketAddress) responsePacket
                .getSocketAddress());
sendqueue.offer(a);
```

### 选票PK

`totalOrderPredicate`会判断一个外部选票是否大于内部选票。判断的逻辑为：

- 外部选票的选举轮次更高
- 外部选票的选举轮次跟内部一样，但是zxid更高
- 外部选票的选举轮次跟内部一样,zxid也相同，但是sid更高

这三种情况都会是外部选票胜出。

```
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
    if(self.getQuorumVerifier().getWeight(newId) == 0){
        return false;
    }

    /*
     * We return true if one of the following three cases hold:
     * 1- New epoch is higher
     * 2- New epoch is the same as current epoch, but new zxid is higher
     * 3- New epoch is the same as current epoch, new zxid is the same
     *  as current zxid, but server id is higher.
     */
    return ((newEpoch > curEpoch) ||
            ((newEpoch == curEpoch) &&
            ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));
}
```

### 变更投票

通过选票PK，确定了外部选票优于内部投票，那么就进行选票变更。使用外部投票的选票信息覆盖内部投票。变更完成后，再次将这个变更后的内部投票发送出去。

```
synchronized void updateProposal(long leader, long zxid, long epoch){
    if(LOG.isDebugEnabled()){
        LOG.debug("Updating proposal: " + leader + " (newleader), 0x"
                + Long.toHexString(zxid) + " (newzxid), " + proposedLeader
                + " (oldleader), 0x" + Long.toHexString(proposedZxid) + " (oldzxid)");
    }
    proposedLeader = leader;
    proposedZxid = zxid;
    proposedEpoch = epoch;
}
```

发送通知：

```
/**
 * Send notifications to all peers upon a change in our vote
 */
private void sendNotifications() {
    for (long sid : self.getCurrentAndNextConfigVoters()) {
        QuorumVerifier qv = self.getQuorumVerifier();
        ToSend notmsg = new ToSend(ToSend.mType.notification,
                proposedLeader,
                proposedZxid,
                logicalclock.get(),
                QuorumPeer.ServerState.LOOKING,
                sid,
                proposedEpoch, qv.toString().getBytes());
        if(LOG.isDebugEnabled()){
            LOG.debug("Sending Notification: " + proposedLeader + " (n.leader), 0x"  +
                  Long.toHexString(proposedZxid) + " (n.zxid), 0x" + Long.toHexString(logicalclock.get())  +
                  " (n.round), " + sid + " (recipient), " + self.getId() +
                  " (myid), 0x" + Long.toHexString(proposedEpoch) + " (n.peerEpoch)");
        }
        sendqueue.offer(notmsg);
    }
}
```

### 选票归档

无论是否进行了投票变更，都会将收到的外部投票放入选票集合recvset中进行归档，recvset用于记录当前服务器在本轮次的选举中收到的所有外部投票

```
voteSet = getVoteTracker(
        recvset, new Vote(proposedLeader, proposedZxid,
                logicalclock.get(), proposedEpoch));

if (voteSet.hasAllQuorums()) {

    // Verify if there is any change in the proposed leader
    while((n = recvqueue.poll(finalizeWait,
            TimeUnit.MILLISECONDS)) != null){
        if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                proposedLeader, proposedZxid, proposedEpoch)){
            recvqueue.put(n);
            break;
        }
    }
```

### 统计投票

投票统计的过程就是为了统计集群中是否已经有了过半的服务器认可了当前的内部投票，如果是，则终止投票

`FastLeaderElection#lookForLeader` :

```
// 投票是否过半
if (voteSet.hasAllQuorums()) {

    // Verify if there is any change in the proposed leader
	while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {
		if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, 			proposedZxid, proposedEpoch)) {
			recvqueue.put(n);
			break;
		}
	}

    /*
    * This predicate is true once we don't read any new
    * relevant message from the reception queue
    */
	if (n == null) {
		setPeerState(proposedLeader, voteSet);
		Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), 	proposedEpoch);
		leaveInstance(endVote);
		return endVote;
	}
}
```



### 更新服务器状态

统计投票后，如果已经确定可以终止投票，那么就更新服务器状态。先判断投票结果的Leader是否是自己，如果是的话，就会将自己的服务器状态更新为Leading，如果不是自己的话，根据情况来确定自己是FOLLOWING还是OBSERVING,这部分的内容，我们上篇文章(启动流程)中有介绍，这里就不重复了。

## 总结

我们在这篇文章中一起了解了  Zookeeper 选举流程，单独看选举过程的话，过程思路还是比较简单，对于ZAB协议等等什么的就没有介绍了，这里就了解整个选举过程就可以了，在整个过程中，很多地方都用到队列，和单独的一些线程，这种思想很值得我们学习。

## 参考

《从PAXOS到ZOOKEEPER分布式一致性原理与实践》

[ZooKeeper源码分析(四) - Leader选举](http://guobingwei.tech/articles/2019/03/29/1553815655905.html)



