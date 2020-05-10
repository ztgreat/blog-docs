## 前言
在前面中，我们已经 了解了ZooKeeper 集群的启动，Leader 选举相关流程，下面我们一起来看看ZooKeeper 读写请求处理中的一些相关细节。

ZooKeeper 系列文章 会讲一些重要的功能和概念，主要包括：

- [重要概念介绍 - 整体视图的认识](http://blog.ztgreat.cn/article/90)
- [服务端启动流程](http://blog.ztgreat.cn/article/91)
- [集群选举过程](http://blog.ztgreat.cn/article/92)
- 读写请求操作
- 会话管理
- 数据与存储

本节主要 讲一下读写请求操作 流程,对于前面内容可以点击相关链接进行跳转。

## 概述

在Zookeeper中对于请求分为两类：

- 事务性请求
- 非事务性请求

所谓事务性请求，说白了就是 写操作。

>更新操作、新增操作、删除操作，因为这些操作是会影响数据的，所以要保证这些操作在整个集群内的事务性，所以这些操作就是事务性请求。

那么非事务性请求就好理解的，像查询操作 这些不影响数据的操作，就不需要集群来保持事务性，所以这些操做就是非事务性请求。

客户端使用 Zookeeper 时会连接到集群中的任意节点，所有的节点都能够直接对外提供读操作，但是写操作都会被从节点(非Leader服务器)路由到主节点(Leader 服务器)，由主节点进行处理。

>为了保证事务请求被顺序执行，从而确保ZooKeeper 集群的数据一致性，**所有的事务请求都必须由Leader 服务器来处理**，所有非Leader 服务器如果接收到了来自客户端的事务请求，那么必须将其**转发给Leader** 服务器来处理。



非事务性的请求 则是直接本地服务器 处理。

>因此读请求存在不能及时读到最新数据的情况



## 请求链

在ZooKeeper 中，对请求的处理 是通过责任链的模式来实现，不同的"责任" 由不同的请求处理器 来处理，这种对功能模块 能实现很好的解耦，在Netty 中的 handler 也是一样的思想。

>ZooKeeper代码里有一个叫RequestProcessor的接口。这个接口的主要方法是processRequest，它接受一个Request参数。

```
public interface RequestProcessor {

    @SuppressWarnings("serial")
    class RequestProcessorException extends Exception {
        public RequestProcessorException(String msg, Throwable t) {
            super(msg, t);
        }
    }
    void processRequest(Request request) throws RequestProcessorException;
    void shutdown();
}
```

`RequestProcessor` 有很多子类，每个子类实现不同功能，一些主要的子类 我们会在后面分析中给出。

对于请求链的初始化，这个是在服务器启动的时候，选举完成后会进行，关于服务端的启动大致启动流程，可以参考前面的文章。

因为请求立链的代码比较复杂，涉及到Leader 和 Follower的交互逻辑，因此这里我们主要简单介绍一下相关的请求里职责，然后再配合流程图来理解，更多的，可以自己阅读相关源码来了解。

在ZooKeeper 中，不同角色的服务器，它们的处理链是不同，下面我们来简单看一下，各自角色它们的处理链情况

### leader 请求链

Leader服务器的请求处理链如下:

![leader 请求链](http://img.blog.ztgreat.cn/document/zookeeper/1588923942088.png)

>备注:图中有队列的处理器，表示 都是一个线程类
>
>调用方通过调用接口 把请求数据放到队列中，处理器本身再不断的从队列中取出数据来处理
>
>后续会简单介绍涉及到的相关队列



##### LeaderRequestProcessor

Leader调用链开始, 这个处理器 主要是 处理 本地  session 相关的，关于会话相关的介绍，我们在后面分析，这里就不多讲了。

>注意：
>
>如果是非Leader 转发过来请求，不会经过这个处理器
>
>只有客户端如果连接到Leader 机器的时候，会经过这个处理器
>
>相关代码可以参考：LeaderZooKeeperServer#submitLearnerRequest

##### PrepRequestProcessor

PrepRequestProcessor:  请求预处理器。在Zookeeper中，那些会改变服务器状态的请求称为事务请求（创建节点、更新数据、删除节点、创建会话等），PrepRequestProcessor能够识别出当前客户端请求是否是事务请求。对于事务请求，PrepRequestProcessor处理器会对其进行一系列预处理，如创建请求事务头、事务体、会话检查、ACL检查和版本检查等。

>注意：PrepRequestProcessor 本身是一个线程类
>
>它会 将请求 放入submittedRequests，然后 不停的从 submittedRequests 队列中取数据出来处理
>
>处理结果则是生成一个事务
>
>对于读操作并不会产生任何事务。因此，对于读请求的Request对象中，事务的成员属性的引用值则为null

ProposalRequestProcessor将会把所有请求都转发给CommitRequestProcessor，而且，对于写操作请求，还会将请求转发给SyncRequestProcessor处理器。



##### ProposalRequestProcessor

ProposalRequestProcessor:  事务投票处理器。Leader服务器事务处理流程的发起者。

对于非事务性请求，ProposalRequestProcessor会直接将请求转发到CommitProcessor处理器，不再做任何处理；

而对于事务性请求，除了将请求转发到CommitProcessor外，还会根据请求类型创建对应的Proposal提议，并发送给所有的Follower服务器来发起一次集群内的事务投票。

同时，ProposalRequestProcessor还会将事务请求交付给SyncRequestProcessor进行事务日志的记录。

```
/* In the following IF-THEN-ELSE block, we process syncs on the leader.
* If the sync is coming from a follower, then the follower
* handler adds it to syncHandler. Otherwise, if it is a client of
* the leader that issued the sync command, then syncHandler won't
* contain the handler. In this case, we add it to syncHandler, and
* call processRequest on the next processor.
*/
// 从节点同步请求
if (request instanceof LearnerSyncRequest) {
	zks.getLeader().processSync((LearnerSyncRequest) request);
} else {
    // 后续处理器 处理
	nextProcessor.processRequest(request);
	// 如果是事务请求，则需要进行 Proposal
	if (request.getHdr() != null) {
		// We need to sync and get consensus on any transactions
		try {
		    // leader 发起 提议
			zks.getLeader().propose(request);
		} catch (XidRolloverException e) {
			throw new RequestProcessorException(e.getMessage(), e);
		}
		// 同步处理器处理(记录事务日志，这个后面会介绍)
		syncProcessor.processRequest(request);
	}
}
```



`zks.getLeader().propose(request)`  就是 Proposal 流程：

1. 准备发起投票

   如果当前请求是事务请求，那么Leader 服务器就会发起一轮事务投票。在发起事务投票之前，首先会进行一系列的检查

2. 生成提议Proposal

   在检查通过后，ZooKeeper 会生成一个提议

3. 广播提议

   生成提议后，Leader 服务器会把提议 放入待发放队列中，此后从队列中取出提议，发给每个Follower

   相关逻辑在 Leader.sendPacket 和 LearnerHandler 中


主要有两个队列：

queuedPackets: 将需要发送的提议 入队，准备发给 follower，也就是待广播的 提议

outstandingProposals: 可以理解为 广播中的提议，后续 需要提交的提议 就从这里面取

##### SyncRequestProcessor

SyncRequestProcessor：事务日志记录处理器。负责将事务持久化到磁盘上。实际上就是将事务数据按顺序追加到事务日志中，同时会触发Zookeeper进行数据快照。

>注意：SyncRequestProcessor 本身是一个线程类
>
>将 请求 放入 queuedRequests 队列后，它会不断的从队列里面取数据进行处理
>
>执行完日志记录后会触发AckRequestProcessor处理器

##### AckRequestProcessor

AckRequestProcessor: 负责在SyncRequestProcessor完成事务日志记录后，向Proposal的投票收集器发送ACK反馈，以通知投票收集器当前服务器已经完成了对该Proposal的事务日志记录。

>Leader 服务器本身也需要对 自己提交的事务请求 进行ACK ,这个 就是通过 该处理器完成的



```
public void processRequest(Request request) {
	QuorumPeer self = leader.self;
	if (self != null) {
		request.logLatency(ServerMetrics.getMetrics().PROPOSAL_ACK_CREATION_LATENCY);
		// leader ack
		leader.processAck(self.getId(), request.zxid, null);
	} else {
		LOG.error("Null QuorumPeer");
	}
}
```



##### CommitProcessor

CommitProcessor：事务提交处理器。对于非事务请求，该处理器会直接将其交付给下一级处理器处理；

对于事务请求，其会等待集群内针对Proposal的投票直到该Proposal可被提交，利用CommitProcessor，每个服务器都可以很好地控制对事务请求的顺序处理。

>注意：CommitProcessor 本身也是一个线程类
>
>里面有四个队列：
>
>queuedRequests： 所有的请求都会放到该队列
>
>queuedWriteRequests：事务请求也会放到该队列
>
>committedRequests：存放已经被提交的请求
>
>pendingRequests：存放等待被提交的请求



当有请求提交给 该处理器时，首先会 放入 queuedRequests 队列中，如果是事务请求 还会放入 queuedWriteRequests 队列中。

此后，它不断的从 queuedRequests队列中取出请求 进行处理，如果是事务请求，则从 pendingRequests 取出

提交给后续的处理器。

>CommitProcessor 会根据请求的SessionId 将请求分配给worker线程，因此同一个SessionId的写请求会分配给同一个worker线程，保证了请求的顺序性
>
>相关代码可以参考：CommitProcessor#sendToNextProcessor



>为了保证执行的顺序，CommitRequestProcessor处理器会在收到一个写请求处理器时暂停后续的请求处理。这就意味着，在一个写请求之后接收到的任何读取请求都将被阻塞，直到读取请求转给
>CommitRequestProcessor处理器。通过等待的方式，请求可以被保证按照接收的顺序来被执行。

##### ToBeAppliedRequestProcessor

ToBeAppliedRequestProcessor：该处理器有一个toBeApplied队列，用来存储那些已经被CommitProcessor处理过的可被提交的Proposal。其会将这些请求交付给FinalRequestProcessor处理器处理，待其处理完后，再将其从toBeApplied队列中移除。

>该处理器会首先将请求 提交给后续处理器，然后自己再来处理该请求(从相关队列中移除已提交请求)。



##### FinalRequestProcessor

FinalRequestProcessor：用来进行客户端请求返回之前的操作，包括创建客户端请求的响应，针对事务请求，该处理还会负责将事务应用到内存数据库中去。

>如果Request对象包含事务数据，该处理器将会接受对ZooKeeper数据树的修改，否则，该处理器会从数据树中读取数据并返回给客户端。

### Follower 请求链

Follower也采用了责任链模式组装的请求处理链来处理每一个客户端请求，由于不需要对事务请求的投票处理，因此Follower的请求处理链会相对简单，其处理链如下



![Follower 请求链](http://img.blog.ztgreat.cn/document/zookeeper/1588923942089.png)



相关代码可以参考 `FollowerZooKeeperServer.setupRequestProcessors()`

```
protected void setupRequestProcessors() {
	RequestProcessor finalProcessor = new FinalRequestProcessor(this);
	commitProcessor = new CommitProcessor(finalProcessor, Long.toString(getServerId()), true, getZooKeeperServerListener());
	commitProcessor.start();
	firstProcessor = new FollowerRequestProcessor(this, commitProcessor);
((FollowerRequestProcessor) firstProcessor).start();
	syncProcessor = new SyncRequestProcessor(this, new 	SendAckRequestProcessor(getFollower()));
	syncProcessor.start();
}
```



##### FollowerRequestProcessor

FollowerRequestProcessor: 其用作识别当前请求是否是事务请求，若是，那么Follower就会将该请求转发给Leader服务器，Leader服务器是在接收到这个事务请求后，就会将其提交到请求处理链，按照正常事务请求进行处理。

>注意：该处理器 本身也是一个线程类，当有请求到达该处理器时，请求会被 放入 queuedRequests 队列中，然后不断的从队列中取数据出来处理。
>
>提交的请求在后续RequestProcessor 处理完后，才会把事务请求转发给Leader 服务器
>
>相关代码可以参考 FollowerRequestProcessor#run



##### SendAckRequestProcessor

SendAckRequestProcessor: 其承担了事务日志记录反馈的角色，在完成事务日志记录后，会向Leader服务器发送ACK消息以表明自身完成了事务日志的记录工作。当Leader服务器接收到足够确认消息来提交这个提议时，Leader就会发送提交事务消息给追随者(**同时也会发送INFORM消息给观察者服务器**)。当接收到提交事务消息
时，追随者就通过CommitRequestProcessor处理器进行处理。

>Follower : 追随者
>
>Observer: 观察者

### Observer 请求链

Observer充当观察者角色，观察Zookeeper集群的最新状态变化并将这些状态同步过来，其对于非事务请求可以进行独立处理，对于事务请求，则会转发给Leader服务器进行处理。Observer不会参与任何形式的投票，包括事务请求Proposal的投票和Leader选举投票。其处理链如下：



![Observer 请求链](http://img.blog.ztgreat.cn/document/zookeeper/1588923942090.png)

## 非事务请求流程

非事务请求这个很简单，我们以Follower 为例，相关逻辑 在FinalProcessor 里面处理的

相关代码可以参考：`FinalRequestProcessor#processRequest`

![Follower非事务请求流程](http://img.blog.ztgreat.cn/document/zookeeper/1588923942091.png)





## 事务请求流程

同样的，我们也以Follower 为例，因为有了前面的请求链的介绍，这里就要简单许多了，我们演示的是客户端的写请求 是请求在 Follower 上的，Follower 会将请求转发给Leader 服务器 :



![Follower非事务请求流程](http://img.blog.ztgreat.cn/document/zookeeper/1588923942092.png)

1. Follower 会将写请求转发给Leader 服务器
2. Leader 将事务请求放入 待提交队列中，然后发起Proposal 投票 (广播提议)，
3. Follower 处理 提议，记录事务日志，然后响应ACK。
4. Leader 处理Follower Ack, 尝试 提交 这个事务操作。
5. 如果 投票 统计 过半，则提交事务操作，通过Follower 提交，然后进入 CommitProcessor 环节



## 总结

我们在这篇文章中一起了解了  Zookeeper 中不同角色的服务器的请求处理链，对于每个请求链，我们大致也了解了它的作用，部分流程可能存在不完整的地方，那里交互逻辑确实比较麻烦，更多的可以自行阅读代码。

## 参考

《从PAXOS到ZOOKEEPER分布式一致性原理与实践》

《ZooKeeper：分布式过程协同技术详解》

https://www.cnblogs.com/sunshine-2015/p/10977200.html