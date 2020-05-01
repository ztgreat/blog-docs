## 前言
最近打算写几篇 深入学习 ZooKeeper 的文章，结合自己的情况和经验，打算从核心功能出发进行研究一下，整个部分不会深入很多细节的代码上面，只做重要的介绍，因为时间的原因 很多点上可能也不会涉及到，对于读者 建议结合源码一起看，不然可能没什么帮助，也记不住的，要学习就适当多看代码，否则只是单纯背书而已，意义不大。

在前面也看过部分的Dubbo 的源码，但是写的文章 都太细节化了，不便于了解整个框架，后面的zookeeper 结束后，会重新整理一些相关的文章，希望能从更高的视角去看待它。 

## 概览
zookeeper 系列文章 会讲一些重要的功能和概念，主要包括：

 - 重要概念介绍 - 整体视图的认识
 - 服务端启动流程
 - 集群选举过程
 - 会话管理
 - 读写请求操作
 - 数据与存储

本节主要 讲一下 zookeeper 的一些总要概念，方便大家对它有一个总体的粗略认识。

## 概念
ZooKeeper 是一个开源的分布式协调服务，ZooKeeper 的设计目标是将那些复杂且容易出错的分布式一致性服务封装起来，构成一个高效可靠的原语集，并以一系列简单易用的接口提供给用户使用。

> 原语： 操作系统或计算机网络用语范畴。是由若干条指令组成的，用于完成一定功能的一个过程。具有不可分割性·即原语的执行必须是连续的，在执行过程中不允许被中断。

ZooKeeper 是一个典型的分布式数据一致性解决方案，分布式应用程序可以基于 ZooKeeper 实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能。

## 数据模型

ZooKeeper 的视图结构和标准的Unix 文件系统非常类似，但是没有引入传统文件系统中目录和文件等相关概念，而是使用了其特有的"数据节点"概念，我们称之为ZNode。

ZNode 是ZooKeeper中数据的最小单元，每个ZNode上都可以保持数据，同时还可以挂载子节点，因此构成了一个层次化的命名空间，我们称之为树。

首先，我们来看下图所示 ZooKeeper数据节点示意图，从而对ZooKeeper 上的数据节点有一个大体的认识。

在ZooKeeper中，每一个数据节点都被称为一个ZNode,所有ZNode 按照层次化结构进行组织，形成一棵树。

ZNode的节点路径标识方式和Unix文件系统路径非常相似，都是由一系列使用斜杠(/) 进行分割的路径标识，开发人员可以向这个节点中写入数据也可以在节点下面创建子节点。

![zk存储数据结构](http://img.blog.ztgreat.cn/document/zookeeper/1537065381513e3180f784d.png)



### 节点特性

ZooKeeper的命名空间是由一系列数据节点组成的，下面我们对数据节点做稍微详细的讲解。

#### 节点类型

在ZooKeeper 中，每个数据节点都是由生命周期的，其生命周期的长短取决于数据节点的节点类型。在ZooKeeper中，节点类型可以分为**持久节点**，**临时节点**和**顺序节点**三大类，具体在节点创建过程中，通过组合使用，可以生成下面四种组合型节点类型:

##### 持久节点

所谓持久节点 是指改数据节点被创建后，就会一直存在于ZooKeeper服务器上，直到有删除操作来主动清除这个节点。

##### 持久顺序节点

持久顺序节点的基本特性和持久节点是一致的，额外的特性表现在顺序性上。

在ZooKeeper中，每个父节点都会为它的第一级子节点维护一份顺序，用于记录下每个子节点创建的先后顺序。

基于这个顺序特性，在创建子节点的时候，可以设置这个标记，那么在创建节点过程中，ZooKeeper会自动为给定节点名加上一个数字后缀，作为一个新的，完整的节点名。

##### 临时节点

和持久节点不同是，**临时节点的生命周期和客户端的会话绑定在一起**，也就是说，如果客户端会话失效，那么这个节点就会被自动清理掉。注意，这里提到的客户端会话失效，而非TCP连接断开。关于ZooKeeper 客户端会话和连接，后面有简单介绍，也会有专门的博客进行介绍。

另外，ZooKeeper 规定了不能基于临时节点来创建子节点，即临时节点只能作为叶子节点。

##### 临时顺序节点

临时顺序节点的基本特性和临时节点也是一致的，同样的是在临时节点的基础上，添加了顺序的特性。



最后关于 代码层面的数据结构，可以参考：`org.apache.zookeeper.server.DataTree` 相关代码：

```
public class DataTree {

    /**
     * This map provides a fast lookup to the datanodes. The tree is the
     * source of truth and is where all the locking occurs
     */
    private final NodeHashMap nodes;
    
}

public class NodeHashMapImpl implements NodeHashMap {
    
    // path to DataNode
    private final ConcurrentHashMap<String, DataNode> nodes;

}

public class DataNode implements Record {


    /** the data for this datanode */
    byte[] data;

    /**
     * the list of children for this node. note that the list of children string
     * does not contain the parent path -- just the last part of the path. This
     * should be synchronized on except deserializing (for speed up issues).
     */
    private Set<String> children = null;
}
```



### 版本信息

ZooKeeper 中为数据节点引入了版本的概念，每个数据节点都具有三种类型的版本信息，对于数据节点的任何更新操作都会引起版本号的变化。

每个 ZNode 都有 3 类版本信息：

- version：数据节点数据内容的版本号
- cversion：数据节点子节点的版本号
- aversion：数据节点 ACL 权限变更 版本号

特别说明：

1. ZooKeeper 中`版本`就是`修改次数`：即使修改前后，内容不变，但`版本`仍会`+1`：`version=0` 表示节点创建之后，修改的次数为 0。
2. `cversion` 子节点列表：ZNode，其中 cversion 只会感知`子节点列表`变更信息，新增子节点、删除子节点，而不会感知子节点数据内容的变更。

>ACL(Access Control List) 权限控制 这里不多讲



### Watcher-数据变更的通知

ZooKeeper 提供了分布式数据的发布/订阅功能，在ZooKeeper 中，引入了Wather 机制来实现这种分布式的通知功能。

ZooKeeper 允许客户端向服务端注册一个Watcher监听，当服务端的一些指定事件触发了这个Watcher,那么就会向指定客户端发送一个事件通知来实现分布式的通知功能。

稍微详细的介绍 在文章后面我会给出来。



## 简单的API

ZooKeeper的设计目标之一是提供一个非常简单的编程界面。因此，它仅支持以下操作：

 - create：在树中的某个位置创建一个节点
 - delete：删除节点
 - 存在：测试节点是否存在于某个位置
 - 获取数据：从节点读取数据
 - 设置数据：将数据写入节点
 - 获取子节点：获取节点子节点的列表
 - sync：等待数据传播

## zk的自我保证
ZooKeeper非常快速且非常简单。但是，由于其目标是作为构建更复杂的服务（例如同步）的基础，因此它提供了一组保证。这些是：

 - 顺序一致性-更新请求顺序进行，来自同一个client的更新请求按其发送顺序依次执行
 - 原子性- 数据更新原子性，一次数据更新要么成功，要么失败
 - 单个系统映像-无论客户端连接到哪个服务器，客户端都将看到相同的服务视图。也就是说，即使客户端故障转移到具有相同会话的其他服务器，客户端也永远不会看到系统的较旧视图。
 - 可靠性-应用更新后，此更新将一直持续到客户端覆盖更新为止。
 - 及时性-确保系统的客户视图在特定时间范围内是最新的。

要做到上面的几点，单机肯定是不行了，需要集群模式，在分布式系统中，典型的架构 就是（master-slave）工作模式了。

## 集群架构

Zookeeper 服务端运行于两种模式下：单机模式(独立模式)和集群模式。独立模式只有一个单独的服务器，Zookeeper状态服务复制。而在集群模式下，具有一组Zookeeper 服务器，它们之间可以进行状态的复制，并同时服务客服端的请求。

下面我们简单了解一下集群模式下Zookeeper各个角色

### Zookeeper的角色

Zookeeper的角色

» 领导者（leader）：负责进行投票的发起和决议，更新系统状态。

» 学习者（learner）：包括跟随者（follower）和观察者（observer），follower用于接受客户端请求并想客户端返回结果，在选主过程中参与投票。

» Observer：可以接受客户端连接，将写请求转发给leader，但**observer不参加投票过程**，只同步leader的状态，observer的目的是为了扩展系统，提高读取速度

» 客户端（client）：请求发起方

一个leader，多个follower和Observer:
![Zookeeper集群](http://img.blog.ztgreat.cn/document/zookeeper/1537065381513e3180f784c.png)



![zk集群角色](http://img.blog.ztgreat.cn/document/zookeeper/847253254-5b9731766c940.jpg)



> 内容 参考：https://segmentfault.com/a/1190000016349824

### Zab协议 

既然集群中 只有一个 leader,那么就存在选举通信问题：

Zookeeper的核心是原子广播，这个机制保证了各个Server之间的同步。实现这个机制的协议叫做**Zab协议**。

Zab协议有两种模式，它们分别是恢复模式（选主）和广播模式（同步）。当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数Server完成了和leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和Server具有相同的系统状态。

这里简单了解一下 Zab 协议就可以了，后面再细说。

## 会话管理

会话（Session）是Zookeeper的一个重要的抽象。保证请求有序、 临时znode节点、监事点都与会话密切相关。因此会话的跟踪机制对 ZooKeeper来说也非常重要。

ZooKeeper服务器的一个重要任务就是跟踪并维护这些会话。在单机模式下，单个服务器会跟踪所有的会话，而在集群模式下则由leader服务器来跟踪和维护。leader服务器上运行会话跟踪器（参考SessionTracker类和SessionTrackerImpl类）。

而追随者服务器仅仅是简单地把客户端连接的会话信息转发给leader服务器（参 考LearnerSessionTracker类和LearnerZooKeeperServer类）。

为了保证会话的存活，服务器需要接收会话的心跳信息。心跳的形 式可以是一个新的请求或者显式的ping消息（参考 LearnerHandler.run（））。
在集群模式下，leader服务器发送一个PING消息给它的追随者们，追随者们返回自从最新一次PING消息之后的一个session列表。

```
protected void ping(QuorumPacket qp) throws IOException {
      // Send back the ping with our session data
      ByteArrayOutputStream bos = new ByteArrayOutputStream();
      DataOutputStream dos = new DataOutputStream(bos);
      Map<Long, Integer> touchTable = zk.getTouchSnapshot();
      for (Entry<Long, Integer> entry : touchTable.entrySet()) {
            dos.writeLong(entry.getKey());
            dos.writeInt(entry.getValue());
      }
      qp.setData(bos.toByteArray());
      writePacket(qp, true);
 }


  /**
   * Returns the current state of the session tracker. This is only currently
   * used by a Learner to build a ping response packet.
   *
   */
protected Map<Long, Integer> getTouchSnapshot() {
   if (sessionTracker != null) {
       return ((LearnerSessionTracker) sessionTracker).snapshot();
    }
    Map<Long, Integer> map = Collections.emptyMap();
    return map;
 }

```

## Watch 机制

在前面我们说了Zookeeper支持分布式订阅-发布功能。
所谓订阅发布功能，其实说白了就是观察者模式。观察者会订阅一些感兴趣的主题，然后这些主题一旦变化了，就会自动通知到这些观察者。

Zookeeper 的订阅发布也就是watcher机制，一个zk的节点可以被监控，包括这个目录中存储的数据的修改，子节点目录的变化，一旦变化(产生event)可以通知设置监控的客户端（我们所说的事件 （event）表示一个znode节点执行了更新操作。而一个监视点（watcher） 表示一个与之关联的znode节点和事件类型组成的单次触发器）。

>参考 WatchManager

```
/**
 * This class manages watches. It allows watches to be associated with a string
 * and removes watchers and their watches in addition to managing triggers.
 */
public class WatchManager implements IWatchManager {

    private final Map<String, Set<Watcher>> watchTable = new HashMap<>();

    private final Map<Watcher, Set<String>> watch2Paths = new HashMap<>();
 
}
```

Zookeeper watch 机制 是一个轻量级的设计。因为它采用了一种**推拉结合**的模式。一旦服务端感知主题(节点)变了，那么只会发送一个事件类型和节点信息给关注的客户端，而不会包括具体的变更内容，所以事件本身是轻量级的，这就是所谓的“推”部分。然后，收到变更通知的客户端需要自己去拉变更的数据，这就是“拉”部分。

![watch机制](http://img.blog.ztgreat.cn/document/zookeeper/1537065381513e3180f785d.png)

ZooKeeper  watcher机制 是**一次性触发**的：
当数据改变的时候，那么一个event事件会产生并且被发送到客户端中。但是客户端只会收到一次这样的通知，如果以后这个数据再次发生改变的时候，之前设置Watch的客户端将不会再次收到改变的通知，因为Watch机制规定了它是一个一次性的触发器。    
因此 如果我们要一直对某个节点保持有效监视 需要 每收到变更通知后，重新注册watch，类似下面的代码：

```
public void subscribe(final SubscriberCallBack subscriberCallBack) throws Exception {
     final CuratorWatcher curatorWatcher = new CuratorWatcher() {
          @Override
          public void process(WatchedEvent watchedEvent) throws Exception {
             byte[] data = client.getData().usingWatcher(this).forPath(ZookeeperPaths.getTopicBase());
                subscriberCallBack.fire(data);
          }
      };
    // 再次注册
    client.getData().usingWatcher(curatorWatcher).forPath(ZookeeperPaths.getTopicBase());
}
```



## 读写请求操作

> 读请求本地服务器处理（本地服务器指的是当前客户端连接的服务器）
> 写请求转发至leader 服务器处理
> 参考：
>  org.apache.zookeeper.server.quorum.FollowerRequestProcessor#run
>  org.apache.zookeeper.server.quorum.Learner#request

ZooKeeper服务器会在本地处理只读请求（exists、getData和 getChildren）。假如一个服务器接收到客户端的getData请求，服务器读取该状态信息，并将这些信息返回给客户端。因为服务器会在本地处理 请求，所以ZooKeeper在处理以只读请求为主要负载时，性能会很高。 我们还可以增加更多的服务器到ZooKeeper集群中，这样就可以处理更多的读请求，大幅提高整体处理能力。

那些会改变ZooKeeper状态的客户端请求（create、delete和 setData）将会被转发给leader，leader执行相应的请求，并形成状态的更 新，我们称为事务（transaction），事务请求必须由Leader服务器来处理。

所有非Leader服务器如果接收到了来自客户端的事务请求，那么必须将其转发给Leader 服务器来处理。



## 总结

我们在这篇文章中简单介绍了  Zookeeper的一些相关概念和功能，从而对ZooKeeper 有一个大体的认识。

作为分布式协调服务，Zookeeper 的应用场景非常广泛，不仅能够用于服务配置的下发、命名服务、协调分布式事务以及分布式锁，还能够用来实现微服务治理中的服务注册以及发现等功能，这些其实都源于 Zookeeper 能够提供高可用的分布式协调服务，能够为客户端提供分布式一致性的支持，在后面的文章中我也会详细介绍Zookeeper的一些核心功能点。

## 参考

​    [官网](https://zookeeper.apache.org/doc/r3.6.1/zookeeperOver.html)

​    [深入理解ZooKeeper](https://www.jianshu.com/p/2b2f49155e83)

 《从PAXOS到ZOOKEEPER分布式一致性原理与实践》

 《ZooKeeper：分布式过程协同技术详解》





