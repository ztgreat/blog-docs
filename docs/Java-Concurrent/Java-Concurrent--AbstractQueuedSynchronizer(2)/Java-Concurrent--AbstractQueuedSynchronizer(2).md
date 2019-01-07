[上文](http://blog.ztgreat.cn/article/7)我们分析了AbstractQueuedSynchronizer独占模式的acquire实现流程，接下来继续看一下AbstractQueuedSynchronizer共享模式acquire的实现流程。可以对比独占模式acquire和共享模式acquire的区别，加深对于AbstractQueuedSynchronizer的理解。

### 共享式同步状态的获取
 同样在开始前，先大概看一下怎么使用？，只有先掌握用法，才能进一步理解为什么

```java
public class MyReadLock implements Lock{

    private final  Sync sync;
    public  MyReadLock(int count){
        sync=new Sync(count);
    }
    private  static  final  class Sync extends AbstractQueuedSynchronizer{
        Sync(int count){
            if (count<=0){
                throw  new IllegalArgumentException("count 必须大于0");
            }
            setState(count);
        }

        @Override
        protected int tryAcquireShared (int arg) {

            for (;;){
                int current=getState();
                int newCount=current-arg;
                if (newCount<0 || compareAndSetState(current,newCount)){
                    return  newCount;
                }
            }
        }

        @Override
        protected boolean tryReleaseShared(int arg) {

            for (;;){
                int current=getState();
                int newCount=current+arg;
                if (compareAndSetState(current,newCount)){
                    return true;
                }
            }
        }
        public Condition newCondition() {
            return new ConditionObject();
        }
    }


    @Override
    public void lock() {
        sync.acquireShared(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return  sync.tryReleaseShared(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return  sync.tryAcquireSharedNanos(1,unit.toNanos(time));
    }

    @Override
    public void unlock() {
        sync.releaseShared(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}
```
其实大体和独占式差不多，只是内部调用方法变了，对外的接口名称没有变。
通过传入的count来控制能同时获取同步状态的线程数（控制并发量），这里的count或state可以理解为资源数，也就是最多能多少个线程同时访问。

```
public void lock() {
    sync.acquireShared(1);
}
```

每次获取同步状态时都进行加1操作（当然你可以随意加多少，看你具体使用场景来定），来看看acquireShared

#### acquireShared
```
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

同样的这里会调用我们重写后的tryAcquireShared

```
@Override
protected int tryAcquireShared (int arg) {
    for (;;){
        int current=getState();
        int newCount=current-arg;
        if (newCount<0 || compareAndSetState(current,newCount)){
            return  newCount;
        }
    }
}
```

这是我们重写后的tryAcquireShared，**独占模式acquire的时候子类重写的方法tryAcquire返回的是boolean，即是否tryAcquire成功；共享模式acquire的时候，返回的是一个int型变量，判断是否<0**
tryAcquireShared 里面我们进行无限循环，首先获取当前state,再判断能否成功获取“资源”:
（1）如果newCount 还大于0，说明一定可以获得同步状态，那么就用CAS设置state，如果失败就再次尝试，直到成功，返回newCount，当然我们根据acquireShared 知道，这种情况你返回大于0的数就可以，最好是返回所剩的资源的数，这样更具有实际意义。

（2）如果newCount <0 ,那么说明无法在获取同步状态了，返回负数。

#### doAcquireShared

```
/**
 * Acquires in shared uninterruptible mode.
 * @param arg the acquire argument
 */
private void doAcquireShared(int arg) {
    //这里是共享模式了
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
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

我们来分析一下这段代码做了什么：

（1）addWaiter，这和独占锁是一样的，把所有tryAcquireShared<0的线程实例化出一个Node（共享模式），添加到同步队列尾中。

（2）只有前驱节点是head的节点才能尝试获取同步状态。

（3）判断当前线程是否需要挂起。

共享模式下的acquire和独占模式下的acquire大部分逻辑差不多，最大的差别在于tryAcquireShared成功之后，**独占模式的acquire是直接将当前节点设置为head节点即可，共享模式会执行setHeadAndPropagate方法** 来看看setHeadAndPropagate 做了什么？

##### setHeadAndPropagate

```
/**
 * Sets head of queue, and checks if successor may be waiting
 * in shared mode, if so propagating if either propagate > 0 or
 * PROPAGATE status was set.
 *
 * @param node the node
 * @param propagate the return value from a tryAcquireShared
 */
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

propagate 是什么？ 就是我们tryAcquireShared的返回值，也就是资源所剩的数量。

（1）首先将获取同步状态成功的节点设置为head。

（2）后面就和独占式不一样了，独占锁某个节点被唤醒之后，它只需要将这个节点设置成head就完事了，而共享锁不一样，某个节点被设置为head之后，如果head的状态<0 表明它需要唤醒后面的节点， 如果它的后继节点是SHARED状态的，那么将继续通过doReleaseShared方法尝试往后唤醒节点，实现了共享状态的向后传播。

为了更好的清楚整个流程，我们就先直接分析 doReleaseShared流程。

##### doReleaseShared

```
/**
 * Release action for shared mode -- signals successor and ensures
 * propagation. (Note: For exclusive mode, release just amounts
 * to calling unparkSuccessor of head if it needs signal.)
 */
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                    !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

在独占式里面我们分析了：

（1）在挂起该线程前，会先设置前驱节点的状态为SIGNAL。 
共享式里面也是如此

如果head节点的状态为SIGNAL，那么说明需要唤醒后继节点，那么先设置head节点的状态为初始值0,然后唤醒后继线程，unparkSuccessor和独占式里面都是一样的，就不进入分析了。

**注意**：当进行唤醒后，后继线程就会进入到同步状态的获取中，如果获取成功就会在来到doReleaseShared 方法中，而且，如果其他线程正在释放状态，那么也会进入此方法，因此这个方法会存在并发，这里假设执行线程为A,被唤醒的线程为B
假设唤醒线程后，线程B没获取到同步状态或者获取后还没设置新的head，那么A发现head 没有变化，那么可以退出了，
如果唤醒线程后，B获取到同步状态，并且设置了新的head，那么A发现head 变了，那么就会再次执行上面逻辑，唤醒后面需要唤醒的后继线程。
头节点本身的waitStatus是0的话，或者有线程在唤醒其他线程时，另一个线程在释放状态，发现头节点的状态为0，那么也会尝试将其设置为PROPAGATE状态的，意味着共享状态可以向后传播。
其实这里感觉还是比较复杂的，因为会存在并发，需要考虑会在哪些情况下这个方法会被调用，如果只是简单理逻辑，还是比较简单，但是这样并不能体会到代码后面的精髓，我理解的也许也不透彻，建议自己多去思考一下。
还是上个流程图吧，有助于理解，本来只想把这个复杂的部分用流程图表示出来，但是好像又不是很连贯，最后把整个过程都弄出来了，流程有点复杂，部分地方可能处理得不太好。

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170820192935901.png)

### 共享模式同步状态的释放

似乎我们在上面已经分析的差不多了，不过还是来撸一遍吧，有个整个认识

```
public void unlock() {
    sync.releaseShared(1);
}
```

依然从我们提供的方法入口 入手

```
/**
 * Releases in shared mode.  Implemented by unblocking one or more
 * threads if {@link #tryReleaseShared} returns true.
 *
 * @param arg the release argument.  This value is conveyed to
 *        {@link #tryReleaseShared} but is otherwise uninterpreted
 *        and can represent anything you like.
 * @return the value returned from {@link #tryReleaseShared}
 */
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

同步器器会调用我们重写的tryReleaseShared 方法
```
protected boolean tryReleaseShared(int arg) {

    for (;;){
        int current=getState();
        int newCount=current+arg;
        if (compareAndSetState(current,newCount)){
            return true;
        }
    }
}
```

通过CAS设置成功后，就进入 doReleaseShared() 方法中了，也就是我们开始分析的那样，这里就不再重复分析了。
我们这个tryReleaseShared 方法其实是有问题的，因为可能造成重复释放问题，怎么解决呢，其实很简单，只要释放到原来的count大小，就不让它释放了，和我们独占式里面很类似。
**分析源代码 要注意里面的注释，其实说得很明白的，有利于自己分析。**

### Condition接口
任意一个java对象，都拥有一组监视器方法定义在（java.lang.Object上），主要包括wait(),notify(),notifyAll()方法，这些方法与synchronized同步关键字配合，可以实现等待/通知模式。Condition接口也提供了类似Object的监视器方法，与lock配合可以实现等待/通知模式。

```
public interface Condition {
    
    void await() throws InterruptedException;

    void awaitUninterruptibly();


    long awaitNanos(long nanosTimeout) throws InterruptedException;

    boolean await(long time, TimeUnit unit) throws InterruptedException;

    boolean awaitUntil(Date deadline) throws InterruptedException;

    void signal();

    void signalAll();
}

```
**await()**
当前线程在接收到唤醒信号（signal）之前或被中断之前一直处于等待休眠状态。调用此方法后，当前线程会释放持有的锁。如果当前等待线程从该方法返回，那么在返回之前会重新获取锁。

 **await(long time,TimeUnit unit)**
调用此方法后，会造成当前线程在接收到唤醒信号之前、被中断之前或到达指定等待时间之前一直处于等待状态。如果在从此方法返回前检测到等待时间超时，则返回 false，否则返回true。

**awaitNanos(long nanosTimeout)**
该方法等效于await(long time,TimeUnit unit)方法，只是等待的时间是
nanosTimeout指定的以毫微秒数为单位的等待时间。该方法返回值是所剩毫微秒数的一个估计值，如果超时，则返回一个小于等于0的值。可以根据该返回值来确定是否要再次等待，以及再次等待的时间。

**awaitUninterruptibly()**
当前线程进入等待状态直到被通知，该方法对中断不敏感。

**awaitUntil(Date deadline)**
当前线程进入等待状态直到被通知，中断或者到某个时间，如果没有到指定时间就被通知，返回true,否则表示到了指定时间，返回false.

**signal()**
唤醒一个等待线程，如果所有的线程都在等待此条件，则选择其中的一个唤醒。在从await返回之前，该线程必须重新获取锁。

**signalAll()**
唤醒所有等待线程，如果所有的线程都在等待此条件，则唤醒所有线程。 在从await返回之前，每个线程必须重新获取锁。

来个demo吧，用生产者和消费者来举例是最合适不过的了，在一个有大小的队列中，生产者往队列中放数据，消费者从中取数据，当队列不满时，生产者可以继续生产数据，当队列不空时，消费者可以不停的从里面取数据，如果不符合条件，则等待，知道符合条件为止。

```
public class FoodQueue<T> {

    //队列大小
    private  int size;

    //list 充当队列
    private List<T> food;

    //锁
    private Lock lock=new ReentrantLock();

    //保证队列大小不<0 的condition
    private Condition notEmpty=lock.newCondition();

    //保证队列大小不>size的condition
    private Condition notFull=lock.newCondition();

    public  FoodQueue(int size){
        this.size=size;
        food=new ArrayList<T>();
    }
    public void product(T t) throws  Exception{
        lock.lock();
        try{

            //如果队列满了，就不能生产了，等待消费者消费数据
            while (size==food.size()){
                notFull.await();
            }

            //队列已经有空位置了，放入数据
            food.add(t);

            //队列已经有数据了，也就是不为空了，可以通知消费者消费了
            notEmpty.signal();
        }finally {
            lock.unlock();
        }

    }

    public T consume() throws  Exception{
        lock.lock();
        try{

            //队列为空，需要等待生产者生产数据
            while (food.size()==0){
                notEmpty.await();
            }
            //生产者生产了数据，可以拿掉一个数据
            T t=food.remove(0);

            //通知消费者可以继续生产了
            notFull.signal();
            return t;
        }finally {
            lock.unlock();
        }

    }
}
```

### Condition的实现分析
Condition的实现类是ConditionObject,ConditionObject是同步器AbstractQueuedSynchronizer的内部类，Condition的操作需要获取相关联的锁，因此需要和同步器挂钩。
每个Condition对象都包含着一个队列（等待队列），Condition中也有节点的概念，在将线程放到等待队列中时会构造节点，而这个节点的定义其实是复用了同步器中节点的定义（可以看上一文中节点的定义）。
下面是截取了同步器中的ConditionObject 的部分内容：
```
public class ConditionObject implements Condition, java.io.Serializable {
    /** First node of condition queue. */
    private transient Node firstWaiter;
    /** Last node of condition queue. */
    private transient Node lastWaiter;

    public ConditionObject() { }
    
    private Node addConditionWaiter() {
    }
    
    private void unlinkCancelledWaiters() {

    }
    public final void signal() {

    }
    public final void signalAll() {

    }

    public final void await() throws InterruptedException {

    }
    ...

}
```


#### 等待队列
等待队列是一个FIFO的队列，在队列中的每个节点都包含了一个线程引用，该线程就是在Condition对象上等待的线程，如果一个线程调用了Condition。await（）方法，那么该线程将会释放锁，构造成节点加入等待队列并进入等待状态
一个Condition包含一个等待队列，Condition拥有首节点（firstWaiter）和尾节点（lastWaiter）。
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170830224000162.png)

#### 等待（await）

```
/**
 * Implements interruptible condition wait.
 * <ol>
 * <li> If current thread is interrupted, throw InterruptedException.
 * <li> Save lock state returned by {@link #getState}.
 * <li> Invoke {@link #release} with saved state as argument,
 *      throwing IllegalMonitorStateException if it fails.
 * <li> Block until signalled or interrupted.
 * <li> Reacquire by invoking specialized version of
 *      {@link #acquire} with saved state as argument.
 * <li> If interrupted while blocked in step 4, throw InterruptedException.
 * </ol>
 */
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

（1）如果线程被中断，抛出中断异常

（2）生成一个节点node（与当前线程绑定），并将该节点加入等待队列中

（3）释放该线程的锁（状态）

（4）直到当前节点不在同步队列中，挂起该线程。

（5）线程唤醒后，获取同步状态（锁）

（6）线程唤醒后，如果不是尾节点，那么会检查队列，清除一些取消的节点。

大致流程是这样，现在来仔细看看每个执行环节：

添加节点到等待队列

```
/**
 * Adds a new waiter to wait queue.
 * @return its new wait node
 */
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

如果尾节点状态不是condition,也就是线程任务被取消了，那么需要从等待队列中清除掉。

```
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null;
    while (t != null) {
        Node next = t.nextWaiter;
        if (t.waitStatus != Node.CONDITION) {
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
```

这个很简单，就是遍历一遍等待队列，把取消了的线程节点从队列中移除。

释放线程获得的锁（同步状态）

```
/**
 * Invokes release with current state value; returns saved state.
 * Cancels node and throws exception on failure.
 * @param node the condition node for this wait
 * @return previous sync state
 */
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

获取state，release的时候将整个state传过去，因为某线程可能多次调用（重入）了lock()方法，需要将状态全部释放，这样后面的线程才能重新从state=0开始竞争锁，这也是方法被命名为fullyRelease的原因，release方法前面的文章详细讲解过。
**思考一下这个问题：Condition 支持 共享式获取锁的方式吗？**，我的推断是不支持，为什么了，在fullyRelease我们看到会释放整个状态，也就是把锁归零了，但是如果是共享的，不是把其它线程的共享锁也释放了吗，这样肯定就不对了呀，不过这个我还没有验证。

判断当前节点是否在同步队列中：
```
/**
 * Returns true if a node, always one that was initially placed on
 * a condition queue, is now waiting to reacquire on sync queue.
 * @param node the node
 * @return true if is reacquiring
 */
final boolean isOnSyncQueue(Node node) {
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // If has successor, it must be on queue
        return true;
    /*
     * node.prev can be non-null, but not yet on queue because
     * the CAS to place it on queue can fail. So we have to
     * traverse from tail to make sure it actually made it.  It
     * will always be near the tail in calls to this method, and
     * unless the CAS failed (which is unlikely), it will be
     * there, so we hardly ever traverse much.
     */
    return findNodeFromTail(node);
}
```

（1）如果当前节点的的状态等于CONDITION或者前驱节点pre为空，则不表示不在同步队列中

（2）如果当前节点的next节点不为空，则表示在同步队列中，next和pre在同步队列中使用，等待队列不会使用的。

（3）如果当前节点的前驱节点pre不为空也不能说明该节点就在同步队列中，看下面代码：

```
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

这个是同步队列中添加节点的方法，这里会先设置node.prev = pred，然后再cas设置tail,但是有可能cas设置失败，导致了设置了node的前驱节点，但是node并没有加入同步队列中。所以才有findNodeFromTail方法，从尾节点往前找，如果找到了当前节点，说明在同步队列中，否则就不在。

（4）如果该线程被唤醒后 会执行acquireQueued，重新加入到获取同步状态的竞争中（挂起前释放了锁），这个方法我们在上文中分析过了。

#### 小结
（1）在执行await之前，线程是获取了锁的，所以没有使用cas来保证线程安全。

（2）调用await后，线程会释放全部锁，然后被挂起

（3）线程被唤醒后，会重新加入到锁的竞争中，在线程挂起前，线程是不在同步队列中的，但是后面唤醒后又在获取同步状态，而获取同步状态是要在同步队列中，并且前驱节点是头结点，那么是谁把它又加入到了同步队列中呢？

（4）await返回 表明该线程已经重新获取到了锁。




#### 通知（signal）

```
/**
 * Moves the longest-waiting thread, if one exists, from the
 * wait queue for this condition to the wait queue for the
 * owning lock.
 *
 * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
 *         returns {@code false}
 */
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

（1）这里我们看到如果当前不是独占模式，那么就会抛出异常，也就是说这个Condition对共享模式是不适用的，是不是验证了我们前面的说法呢？
（2）取第一个等待节点，然后进行唤醒工作

```
/**
 * Removes and transfers nodes until hit non-cancelled one or
 * null. Split out from signal in part to encourage compilers
 * to inline the case of no waiters.
 * @param first (non-null) the first node on condition queue
 */
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
            (first = firstWaiter) != null);
}
```

（1）重新设置firstWaiter，指向第一个waiter的nextWaiter

（2）如果第一个waiter的nextWaiter为null，说明当前队列中只有一个waiter，lastWaiter置空
因为firstWaiter是要被signal的，因此它没什么用了，nextWaiter置空，执行transferForSignal方法

```
/**
 * Transfers a node from a condition queue onto sync queue.
 * Returns true if successful.
 * @param node the node
 * @return true if successfully transferred (else the node was
 * cancelled before signal)
 */
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

从注释我们可以看出该方法是将一个节点从Condition队列转换为AbstractQueuedSynchronizer队列（是不是解决了我们前面的疑问呢？）

总结一下方法的实现：

（1）尝试将Node的waitStatus从CONDITION置为0，这一步失败直接返回false

（2）当前节点进入调用enq方法进入AbstractQueuedSynchronizer队列

（3）当前节点通过CAS机制将waitStatus置为SIGNAL，唤醒线程
返回true，代表唤醒成功。

这里我们可以得出一个重要结论：**某个被await()的节点被唤醒之后并不意味着它后面的代码会立即执行，它会被加入到AbstractQueuedSynchronizer队列的尾部**

（4）如果transferForSignal 执行失败，那么会返回false,然后会对下个节点进行唤醒，同时firstWaiter也会被重新设置，也就是唤醒失败的那个节点会被移除等待队列。这里有疑问？如果该线程被取消了，那么唤醒失败，移除该节点是正确的，如果线程没被取消，但是仍然唤醒失败，那么线程从此以后是否就不会被唤醒了呢？除非中断它。

OK，到这里await()和signal()就分析完了，相比独占模式的分析，要简单很多吧，除此之外，Condition还有其他几个方法，超时等待，不响应中断的等待，唤醒所有节点，这几个方法都比较简单，超时和中断问题在独占模式中有分析到，唤醒全部节点，就是遍历一遍等待队列，然后全部唤醒就可以了。

 [Java 并发 ---原子操作的实现原理](http://blog.ztgreat.cn/article/2)

[Java 并发---解读volatile synchronized](http://blog.ztgreat.cn/article/3)

 [Java 并发 ---AbstractQueuedSynchronizer(同步器)-独占模式](http://blog.ztgreat.cn/article/7)