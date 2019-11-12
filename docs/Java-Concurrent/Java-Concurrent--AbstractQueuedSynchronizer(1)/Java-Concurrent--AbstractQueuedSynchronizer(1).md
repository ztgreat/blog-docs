**基于jdk 1.8**

前面我们了解到可以通过synchronized关键字实现锁功能，使用该关键字不需要我们显式的获取和释放锁，操作很方便简单，但是如果我们需要对锁进行操作或者干预，那么这个是没有办法的。
在Lock接口出现之前，Java程序是靠synchronized关键字实现锁功能的，而Java SE5之后，并发包中新增了Lock（和相关实现类）用来实现锁功能，来看看两者有什么区别：

 - Lock可以让等待锁的线程响应中断，而synchronized却不行，使用synchronized时，等待的线程会一直等待下去，不能够响应中断；
 - lock可以尝试获取锁，如果锁被其他线程持有，则返回 false，不会使当前线程休眠（尝试非阻塞获取锁）。
 - lock 可以 超时获取锁。
 - synchronized 会自动释放锁，lock 则不会自动释放锁。
 - Lock可以提高多个线程进行读操作的效率。

##  **Java Lock接口：**

```java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```
###  **lock()，unlock()**
lock() 可以用于进行加锁，unlock()当然是释放锁，需要注意， **lock不会像 synchronized 那样自动释放锁** ，所以： 一定要放在 try-finally块中，保证锁的释放。 例如：

```java
try {
    lock.lock();
    ......
} finally {
    lock.unlock();  
}
```
### **tryLock()**

 - tryLock()：尝试获得锁，如果成功，返回 true，否则，返回 false。
 - tryLock(long time,TimeUnit unit)：超时的获取锁，当前线程在下面3种情况下会返回：
    - 当前线程在超时时间内获得了锁。	
     - 当前线程在超时时间内被中断。
     - 超时时间结束，返回false。


### **lockInterruptibly 方法 **
 - 当通过这个方法去获取锁时，如果线程正在**等待获取锁**，则这个线程能够**响应中断**，例如当两个线程同时通过lock.lockInterruptibly()想获取某个锁时，假若线程A获取到了锁，而线程B在等待，那么对线程B调用threadB.interrupt()方法能够中断线程B的等待过程，让其返回。
 - 用synchronized修饰的话，当一个线程处于**等待某个锁的状态**，是无法被中断的，只有一直等待下去。
     newCondition()

### **newCondition()**
用于获取一个 Conodition 对象,使得某个，或者某些线程一起等待**某个条件**（Condition）,只有当该条件具备( signal 或者 signalAll方法被带调用)时 ，这些等待线程才会被唤醒，从而重新争夺锁。

lock 方法大体就介绍到这里，通过lock接口分析，我们知道了，通过lock的接口方法，可以实现锁的获取，释放等操作，这些接口都是统一的，对用户（开发者）来说都是透明的，不需要明白底层实现细节就可以实现调用，但是作为开发者仅仅会用也太没意思了，我们来看看lock 有哪些实现类：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170814230132778.png)

我们暂且不讨论每个实现类，我们看看ReentrantLock（可重入锁，synchronized可以重入吗？）会发现该类里面有个重要的内部类Syn：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170814230910935.png)

Syn继承AbstractQueuedSynchronizer（同步器） ，而它就是我们今天讨论的重点，Lock接口的实现基本都是通过聚合一个同步器的子类来完成线程访问控制的。

队列同步器 AbstractQueuedSynchronizer 使用来构建锁或者其他同步组件的基础框架：

 - 使用一个**int成员变量表示同步状态**
 - 通过**内置的FIFO队列**来完成资源获取线程的排队工作

同步器的设计是基于**模板方法模式**，也就是说，使用者需要继承同步器并重写指定的方法，随后将同步器**组合**在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些**模板方法将会调用使用者重写的方法**。
重写同步器指定的方法时，需要使用同步器提供的如下三个方法来访问或者修改同步状态：

 - getState():获取当前同步状态。
 - setState():设置当前同步状态。
 - compareAndSetState():使用CAS设置当前状态。

### **同步器可重写的方法 **
| 方法名称                      | 描述                                       |
| ------------------------- | ---------------------------------------- |
| tryAcquire(int arg)       | 独占获取同步状态，实现该方法需要查询当前状态，并判断同步状态是否符合预期状态，然后再进行CAS设置同步状态。 |
| treRelease(int arg)       | 独占式释放同步状态，等待获取同步状态的线程将有机会获取同步状态          |
| tryAcquireShared(int arg) | 共享式获取同步状态，返回大于等于0的值，表示获取成功，反之失败          |
| tryReleaseShared(int arg) | 共享式释放同步状态                                |
| isHeldExclusively()       | 当前同步器是否在独占模式下被线程占用，一般该方法表示是否被当前线程所独占     |

### **同步器提供的模板方法**
（1）void acquire(int arg)


 - 独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则，将会进入同步队列等待，该方法将会调用重写的tryAcquire(intarg)方法。

（2）void acquireInterruptibly(int arg)

 - 与acquire(int arg)相同，但是该方法响应中断，当前线程未获取到同步状态而进入同步队列中，如果当前线程被中断，则该方法会抛出InterruptedException并返回

（3）boolean tryAcquireNanos(int arg,long nanos)

 - 在acquireInterruptibly(int arg)基础上增加了超时限制，如果当前线程在超时时间内没有获取到同步状态，那么就会返回false,如果获取到就返回true

（4）void acquireShared(int arg)


 - 共享式的获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式获取的主要区别是在同一时刻可以有多个线程获取到同步状态。

（5）void acquireSharedInterruptibly(int arg)


 - 与acquireShared(int arg)相同，该方法响应中断。

（6）boolean tryAcquireSharedNanos(int arg,long nanos)


 - 在acquireSharedInterruptibly(int arg)基础上增加了超时限制。


（7）boolean release(int arg)

 - 独占式的释放同步状态，该方法会在释放同步状态之后，将同步队列中第一个节点包含的线程唤醒。

（8）boolean releaseShared(int arg) 
​	
 - 共享式的释放同步状态

（9）Collection<Thread>getQueuedThreads()


 - 获取等待在同步队列上的线程集合

到这里我们就可以根据前面的知识自己写一个同步组件WriteLock,用于对某个资源"写"的访问:

```java
public class WriteLock implements Lock {

    private  static  Sync sync=new Sync();

    private  static   class  Sync extends AbstractQueuedSynchronizer{

        @Override
        protected boolean tryAcquire(int acquires) {

            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int arg)  {
            if (getState()==0)
                throw  new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;

        }

        public Condition newCondition() {
            return new ConditionObject();
        }

        /*
           是否处于独占状态
         */
        @Override
        protected boolean isHeldExclusively() {
            return  getState()==1;
        }
    }
    public void lock() {
        sync.acquire(1);
    }

    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    public boolean tryLock() {
        return  sync.tryAcquire(1);
    }

    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    public void unlock() {
        sync.release(1);
    }

    public Condition newCondition() {
        return sync.newCondition();
    }


}
```

整体过程很简单（因为是独占模式，所以状态只有0和1）：

 - **继承同步器，重写里面的指定方法**，其实就是对同步状态安装自己的需求进行修改和判断。
 - **组合Sync**，在WriteLock类中，提供对外接口，对外接口实际依赖Sync类中的方法。

简单测试一下，模拟了100个线程对一个非线程安全的集合的操作

```java
public class TestMyLock {

    public  static List<String>list=new ArrayList<>();

    static  WriteLock writeLock=new WriteLock();
    static ReadLock readLock=new ReadLock(4);

    static  class  Task implements  Runnable{

        @Override
        public void run() {

            for (int i=0;i<100;i++){
//                writeLock.lock();
                list.add(Thread.currentThread().getName()+"---"+i);
//                writeLock.unlock();
            }
        }
    }
    public static void main(String[] args) throws  Exception{

        List<Thread>list=new ArrayList<>();

        Task task=new Task();
        for (int i=0;i<100;i++){
            list.add(new Thread(task,"thread"+i));
        }
        for (int i=0;i<100;i++){
            list.get(i).start();
        }
        for (int i=0;i<100;i++){
            list.get(i).join();
        }
        System.out.println(TestMyLock.list.size());

    }
}
```
如果不加锁，那么总的大小可能不会是100*100，加锁后，总的集合大小就是100*100

## 队列同步器的实现分析(独占模式)
同步器依赖内部的同步队列（一个FIFO双向队列）来完成同步状态的管理，**当前线程获取同步状态失败时，同步器会将当前线程以及等待状态信息构造成一个节点（Node）并加入到同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态**

### 定义

```java
static final class Node {

    //表示当前线程处于共享状态
    static final Node SHARED = new Node();

    //独占状态
    static final Node EXCLUSIVE = null;
    
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;
    volatile int waitStatus;
    
    //前驱节点
    volatile Node prev;
    
    //后继节点
    volatile Node next;
    
    //获取同步状态的线程
    volatile Thread thread;
    //指向下一个处于阻塞等待的节点
    Node nextWaiter;
        ...
}
```

 waitStatus变量，用于描述节点当前的状态，一共有5种状态：

 - CANCELLED  取消状态（需要从同步队列中取消等待）
 - SIGNAL 等待触发状态（**后继节点的线程处于等待状态，如果当前线程释放了同步状态或者被取消，将会通知后继节点**）
 - CONDITION 等待条件状态（节点在等待队列（不是同步队列）中，等待在Condition上，当其他线程对Condition调用signal()后，该节点会从等待队列中转移到同步队列中）
 - PROPAGATE 状态需要向后传播
 - INITIAL，值为0 ，初始状态

## 同步队列的基本结构

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170819153303970.png)

同步器包含了两个节点类型的引用，一个指向头节点，一个指向尾节点，一个双向链表，当线程获取同步状态失败时，会被构造成一个Node,然后加入到队列尾中。
同步队列遵循FIFO,**首节点是获取同步状态成功的节点，首节点的线程在释放同步状态时，会唤醒后继节点，而后继节点将会在获取同步状态成功时将自己设置为首节点**

## 独占式同步状态获取
在我们WriteLock中，通过调用lock方法，进行独占式同步状态获取

```
public void lock() {
    sync.acquire(1);
}
```

内部调用了son中的acquire()方法，这个方法是同步器中的方法

```
/**
 * Acquires in exclusive mode, ignoring interrupts.  Implemented
 * by invoking at least once {@link #tryAcquire},
 * returning on success.  Otherwise the thread is queued, possibly
 * repeatedly blocking and unblocking, invoking {@link
 * #tryAcquire} until success.  This method can be used
 * to implement method {@link Lock#lock}.
 *
 * @param arg the acquire argument.  This value is conveyed to
 *        {@link #tryAcquire} but is otherwise uninterpreted and
 *        can represent anything you like.
 */
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

主要逻辑：
（1）首先调用我们自定义同步器实现的tryAcquire(int arg)方法。

```
protected boolean tryAcquire(int acquires) {
    if (compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false;
}
```

 该方法保证线程安全的获取同步状态（CAS）,如果获取失败则返回false,如果获取成功，因为这个是独占式的，因此需要设置当前独占线程。返回true。

 （2）如果获取同步状态成功，那么获取锁成功，如果获取失败，则需要执行：acquireQueued(addWaiter(Node.EXCLUSIVE), arg)，它由两步构成：

 - addWaiter，添加一个等待者
 - acquireQueued，尝试从等待队列中去获取执行一次acquire动作
     分别看一下每一步做了什么。

### addWaiter
传入的参数Node.EXCLUSIVE，我们知道这是独占模式：	

```
/**
 * Creates and enqueues node for current thread and given mode.
 *
 * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
 * @return the new node
 */
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

首先生成一个与当前线程相关的节点（node），然后获得当前数据结构中的尾节点：

（1）如果有尾节点（队列不为空），新生成的node的前驱节点指向尾节点，再更新尾节点（设置当前尾节点为新节点node）,因为这里存在并发，因此需要通过CAS来设置，设置成功后，新的node节点成为了尾节点，那么更新原来的尾节点的next域（指向新的尾节点 node）。假如当前节点没有被设置为尾节点，那么执行enq方法。

（2）如果尾节点为空（队列为空）则执行enq方法。

```
/**
 * Inserts node into queue, initializing if necessary. See picture above.
 * @param node the node to insert
 * @return node's predecessor
 */
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```



**能执行enq方法：队列为空，或者设置当前节点为尾节点失败。**
```
/**
 * Inserts node into queue, initializing if necessary. See picture above.
 * @param node the node to insert
 * @return node's predecessor
 */
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

这段代码的逻辑为：

（1）如果尾节点为空，也就是队列为空，那么new一个**不带任何状态的Node作为头节点**，此时该节点既是尾节点也是头结点，然后回到循环开头再执行。

（2）如果尾节点不为空，那么并发下使用CAS算法将当前Node追加成为尾节点，由于是一个死循环，因此所有没有成功acquire的Node最终都会被追加到同步队列中



以上过程流程图展示如下：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170819204120318.png)



### acquireQueued
节点进入同步队列之后，需要让队列中**符合条件的节点**获取同步状态。
```
/**
 * Acquires in exclusive uninterruptible mode for thread already in
 * queue. Used by condition wait methods as well as acquire.
 *
 * @param node the node
 * @param arg the acquire argument
 * @return {@code true} if interrupted while waiting
 */
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

当前线程在死循环中尝试获取同步状态：
   （1）获取当前线程节点的前驱节点（要么是head 要么是其它节点）
​	   

```
final Node predecessor() throws NullPointerException {
    Node p = prev;
    if (p == null)
        throw new NullPointerException();
    else
        return p;
}
```

（2）如果前驱节点是head，那么tryAcquire(尝试获取同步状态)，如果获取成功，设置当前获取成功的节点为head,将head 的next域置空（原head会被回收）。返回是否被中断。

（3）如果前驱节点不是head,或者是head但是获取同步状态失败，那么需要判断当前线程是否需要被挂起(执行shouldParkAfterFailedAcquire)，我们来看看shouldParkAfterFailedAcquire

```
/**
 * Checks and updates status for a node that failed to acquire.
 * Returns true if thread should block. This is the main signal
 * control in all acquire loops.  Requires that pred == node.prev.
 *
 * @param pred node's predecessor holding status
 * @param node the node
 * @return {@code true} if thread should block
 */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

这里需要判断它前驱节点的状态，如果：
（1）**它的前驱节点是SIGNAL状态的**，返回true，表示当前节点应当被挂起（阻塞），否则以下都返回false,暂时不忙挂起，以为后序在操作的过程中，同步状态可能已经改变。

（2）**它的前驱节点的waitStatus>0**，相当于CANCELLED（因为状态值里面只有CANCELLED是大于0的,表示该线程需要从同步队列中取消等待），那么CANCELLED的节点作废，当前节点不断向前找并重新连接为双向队列，直到找到一个前驱节点waitStats不是CANCELLED的为止

（3）**它的前驱节点不是SIGNAL状态且waitStatus<=0**，也就是waitStatus 为 0 or PROPAGATE，也就是说，前面还有线程在等待，那么目前该节点是需要等待的，如果阻塞了需要前面节点唤醒，那么前面节点如何知道才需不需要唤醒后继节点呢，那么就是利用CAS机制，更新前驱节点等待状态为SIGNAL状态。那么前驱节点在释放同步状态是通过判断状态就可知道需不需要唤醒后继节点了。

如果一直获取不到同步状态，那么线程执行（可能会多次） shouldParkAfterFailedAcquire就会返回true，表示该线程应该被挂起，接下来执行parkAndCheckInterrupt();

```
/**
 * Convenience method to park and then check if interrupted
 *
 * @return {@code true} if interrupted
 */
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

LockSupport.park(this)会挂起当前线程，遇到下列情况之一返回:

```java
1. **其他某个线程将当前线程作为目标调用 unpark**；
2. **其他某个线程中断当前线程**；
3. 该调用不合逻辑，返回。
```


好，我们再回到acquireQueued 方法中，如果当前现在在阻塞后返回，如果在是被中断的，那么就会记录（interrupted = true），返回后重新进行执行上面逻辑获取同步状态。
 如果是在中断后成功获取到同步状态那么就会执行selfInterrupt()。
```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

```
/**
 * Convenience method to interrupt current thread.
 */
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

该方法将会自己中断自己（**中断方法只是设置中断标志位为true**，并不发生异常，便于调用者通过中断标志位来判断是否被中断）。

到这里我们可以知道，同步器的acquire(int arg)方法对中断不敏感，也就是说对线程中断操作时，线程不会从同步队列中移除（只是会再次获取同步状态，如果不符合条件容易会被再次加入队列），但是在成功获取到同步状态后，我们可以查看该线程的中断标志位(isInterrupted())，知道该线程是否被中断过。

上面整个流程如下：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170819183901705.png)



独占模式同步状态的获取我们已经分析完了，
### 小总结
 **（1）等待队列是FIFO先进先出。**

 **（2）在挂起该线程前，会先设置前驱节点的状态为SIGNAL。**

**（3）只有前一个节点的状态为SIGNAL时，当前节点的线程才能被挂起（不就是第二点嘛）**

**（4）将节点加入同步队列时，使用了无限循环，并使用CAS，保证了节点会被线程安全的添加到尾节点上。**

**（5）只有前驱节点是头节点，才能够尝试获取同步状态（说明队列是FIFO）**

**（6）加入同步队列后，并不是立即挂起，而是再次进行获取同步状态, 到挂起之前都是在自旋（无限循环尝试），因为同步状态的变化很快，线程上下文的切换比较耗时，所以用短暂的自旋来换取时间开销，当然如果一直自旋，那么开销反而大于了线程切换。所以把自旋时间（次数）控制在一定范围有利于提高性能。**

下面是整个独占模式获取同步状态的流程，个人觉得流程图有助于把知识点串起来，只是单个分析某个功能，就算弄明白，但是整个还是会感觉有点模糊，而且这样都是碎片段，不容易理清思路

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170819232221089.png)




## 独占式同步状态释放
看完获取我们在来看看如何释放的
```
public void unlock() {
    sync.release(1);
}
```

同样 释放同步状态内部依赖于sync的realease 方法

```
/**
 * Releases in exclusive mode.  Implemented by unblocking one or
 * more threads if {@link #tryRelease} returns true.
 * This method can be used to implement method {@link Lock#unlock}.
 *
 * @param arg the release argument.  This value is conveyed to
 *        {@link #tryRelease} but is otherwise uninterpreted and
 *        can represent anything you like.
 * @return the value returned from {@link #tryRelease}
 */
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

同步器中的release 方法将会调用我们的tryRelease 方法

```
protected boolean tryRelease(int arg)  {
    if (getState()==0)
        throw  new IllegalMonitorStateException();
    setExclusiveOwnerThread(null);
    setState(0);
    return true;

}
```

如果同步状态已经为0了，表示状态错误，不需要再释放，否则设置独占线程为NULL，并将状态重新设置为0，这里我们设置状态为什么没有用CAS呢？，这样是线程安全的吗？，因为这是独占模式，那么就只有一个线程能获取到锁（获取锁的过程是竞争的，所以需要CAS）,所以释放的时候肯定也是只有一个一个线程的释放，不存在竞争，所以不需要用CAS。

如果当前线程释放成功了，那么我们前面说了，需要唤醒后继线程，当线程获取同步状态成功后，会将自己设置为头节点，因此这里只需要从head 开始，如果head的状态<0，则可以进行唤醒工作。

```
/**
 * Wakes up node's successor, if one exists.
 *
 * @param node the node
 */
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

如果head 节点状态小于0，那么恢复成初始值0，设置失败了也没关系，个人理解，当前head节点已经释放了同步状态，唤醒后继状态的过程中了，其等待状态不会影响到后面的行为（如果不对，欢迎指正）。
将head后面的节点进行唤醒即可，但是后面的节点（线程）可能已经被取消了（CANCELLED），因此不需要将等待状态大于0的线程唤醒了，这里是从后往前找离head最近的需要被唤醒的节点（为什么需要从后往前找呢？不直接从前面开始找），然后对它进行unpark。唤醒后就会进行我们上面的分析的那样重新进入同步状态的竞争中。

独占式同步锁的获取和释基本上分析得差不多了，不知道我表述明白了没有，独占式还有其他几个方法：独占式超时获取同步状态，可中断的获取同步锁，其实这个两个就是在上面的获取同步状态上做了一点小手脚，详细分析应该没必要了吧，就简单看一下就可以了。

### 可中断的获取同步锁，独占式超时获取同步状态

```
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

这个tryAcquire 的差别在于，如果当前线程被中断，那么会抛出中断异常,在看看doAcquireInterruptibly 和acquireQueued差别如下：

```
if (shouldParkAfterFailedAcquire(p, node) &&parkAndCheckInterrupt())
        throw new InterruptedException();
```

想想acquireQueued 里面怎么做的？ acquireQueued中如果当前线程被中断，那么会标记该线程被中断，但是会继续进行同步状态竞争，如果失败又会加入到同步队列中，也就是只是记录是否被中断，不会告诉用户哪个时候中断的。而doAcquireInterruptibly 里面一旦被中断，那么就会抛出中断异常，不会再进行同步状态的竞争了，so 就是这样了，明白了吧。

至于超时中断可以被视作响应中断获取同步状态过程的加强版，针对超时获取，先计算最后期限deadline，deadline=系统当前时间+nanosTimeout（超时时间），当线程唤醒后用deadline-系统当前时间，如果小于0，那么超时，否则还需要睡眠nanosTimeout = deadline - 系统当前时间。

```
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```


回到自己实现的WriteLock中，我们看到在Syn内部都是设置状态为0或者1，这个是因为这个是独占式同步器组件，如果我们状态不止0,1呢，那岂不是、就不是独占了，是的，其实我们可以控制状态，控制并发线程数，类似Semaphore(不知道？没关系)。

```java
public class ReadLock implements Lock {

    private    Sync sync;

    private static int count;

    public  ReadLock(int count){
        this.count=count;
        sync=new Sync(count);
    }
    private  static   class  Sync extends AbstractQueuedSynchronizer {

        public Sync(int count){
            setState(count);
        }
        @Override
        protected boolean tryAcquire(int acquires) {
            int c=getState();
            if (c==0)
                return false;
            if (compareAndSetState(c, c-acquires)) {
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int arg)  {
            int c=getState();
            if (c==count)
                throw  new IllegalMonitorStateException();
            if (compareAndSetState(c, c+arg)) {
                return true;
            }
            return false;

        }
        public Condition newCondition() {
            return new ConditionObject();
        }
    }
    public void lock() {
        sync.acquire(1);
    }

    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    public boolean tryLock() {
        return  sync.tryAcquire(1);
    }

    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    public void unlock() {
        sync.release(1);
    }

    public Condition newCondition() {
        return sync.newCondition();
    }

}
```
在上面代码中，我们不在是独占式获取同步状态了，我们可以多个线程获取同步状态，具体有好多个线程可以获取到呢（count个）。其实就是把state初始值设置成了count ,每次有线程获取同步状态时，就把state-1，如果state=0，则不能再继续获取了，获取失败。释放的时候则把state+1即可。仔细观察，我们发现在WriteLock和ReadLock里面的tryRelease设置state是不一样的，在WriteLock因为是独占式的，所以不需要用CAS来保证线程安全，但是在ReadLock中，因为可能多个线程会同时释放，因此我们需要用CAS来保证线程安全。

```java
public class TestMyLock {

    static  WriteLock writeLock=new WriteLock();
    static ReadLock readLock=new ReadLock(4);

    static  class  Task implements  Runnable{

        @Override
        public void run() {
            readLock.lock();
            System.out.println(Thread.currentThread().getName()+" 进入lock");
            try {

                Thread.sleep(1500);
            }catch (Exception e){
                e.printStackTrace();
            }
            readLock.unlock();

            System.out.println(Thread.currentThread().getName()+" 释放lock");

        }
    }

    public static void main(String[] args) throws  Exception{


        List<Thread>list=new ArrayList<>();
        Task task=new Task();
        for (int i=0;i<10;i++){
            list.add(new Thread(task,"thread"+i));
        }
        for (int i=0;i<10;i++){
            list.get(i).start();
        }
        for (int i=0;i<10;i++){
            list.get(i).join();
        }
    }

}
```
简单测试了一下，设置了并发数为4，模拟了10个线程去获取锁的过程。

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170819224036418.png)



其实在同步器队列中有共享式同步状态获取与释放，其实现方式和独占式大同小异，下篇博客将分析共享式与基于Condition的等待/通知机制实现。

最后，以上分析因为知识有限，不保证完全正确，欢迎指出问题。

 [Java 并发 ---原子操作的实现原理](http://blog.ztgreat.cn/article/9)

[Java 并发---浅析volatile synchronized](http://blog.ztgreat.cn/article/10)