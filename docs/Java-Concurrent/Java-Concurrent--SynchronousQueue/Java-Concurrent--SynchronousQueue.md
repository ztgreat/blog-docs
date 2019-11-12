## SynchronousQueue 介绍（jdk 1.8）

### 前戏
与前面分析的几个阻塞队列不一样(ArrayBlockingQueue,PriorityBlockingQueuey)，SynchronousQueue 很抽象，前面我们看到的阻塞队列都是真真实实的队列，而且从队列的数据结构上，我们都不难进行分析，只要知道了其存储的数据结构，然后再从方法上很容易进行分析，同时使用了可重入锁来控制数据的安全，锁使得线程安全实现起来如此的简单，而今天我们分析的SynchronousQueue 却和前面大大的不同，比前面的阻塞队列都要来得抽象，同时更加的复杂，SynchronousQueue 并没有使用锁来保证线程的安全，使用的是CAS方法，因此其实现过程就要复杂许多.
前面的队列中主要的功能还是存储和传递数据，而在SynchronousQueue  中更多的传递信息的作用。
网上很多都说SynchronousQueue  是一个不存储元素的BlockingQueue，但是我自己从debug 源码的过程中，发现其实是存储了元素的，也许很多朋友表达的意思是SynchronousQueue  的主要功能不是维护元素数据，但是不存储元素和主要功能不是存储元素这两个还是很大区别的，这给很多开始打算了解SynchronousQueue  的朋友们会造成困扰。
SynchronousQueue  代码确实很抽象，其它阻塞队列基本上一看就大概知道是怎么回事了，但是当我看SynchronousQueue  时，发现连方法都看不懂，其数据结构完全不清楚，同时我看到很多网上的朋友的分析也不太通俗，一开始讲的不是SynchronousQueue  的数据结构，也不是讲SynchronousQueue  的大致思想，而是直接就开始分析，对于最开始看SynchronousQueue  源码的朋友来说基本上是看不懂。因此这里我总结自己的理解和网上朋友们的思路 尽量通俗的分析SynchronousQueue  ，力争做到完全不了解SynchronousQueue  的朋友也能大致看明白。
个人建议 自己先动手debug 一下源码，跟着流程走一遍，知道个大概情况，然后在看网上的博客，这样有助于自己理解，同时也能提高自己分析代码的能力。

### 功能
SynchronousQueue中的队列不是针对数据的，而是针对操作，也就是入队不一定就是入队数据，而是入队的操作，操作可以是put,也可以是take，put操作与take操作对应，可以互相匹配，put和put，take和take则是相同的操作（模式）。

SynchronousQueue  不是一个真正的队列，其主要功能不是存储元素，而且维护一个排队的线程清单，这些线程等待把元素加入或者移除队列。每一个线程的入队（出队）操作必须等待另一个线程的出队（入队）操作。
在传统的队列中，线程直接把数据放到队列中就可以了，然后去执行其他任务，但是在SynchronousQueue   中却行不通，当一个线程（A）尝试对SynchronousQueue进行入队（出队）操作，  那么它将会进行阻塞，直到另一个线程(B)来进行出队（入队）操作，如果没有其他线程(B)来进行对应的出队（入队）操作，那么线程（A）将一直阻塞。

当许多线程进行入队时，并且没有线程来取数据，那么这些入队线程都会阻塞，SynchronousQueue  也会形成一个队列，队列中的每个节点有存储的数据，同时也有阻塞的线程（方便后续唤醒），因此SynchronousQueue   中即存储的数据，也存储了线程。把SynchronousQueue  当做普通的队列来用肯定是不行的，其SynchronousQueue  的主要意义在于适合传递性的工作，充当一个传球手的角色，只有两边同时准备好了的情况下才了交接“数据”。

###  继承体系

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171102194648382.png)
这个我们前面分析的几个队列是一样的，这里就不多说了。

## 公平SynchronousQueue
这里我们先从公平的SynchronousQueue开始分析，因为这个相对非公平的SynchronousQueue要简单一些
### 数据结构
对于任何算法的分析，个人觉得都需要先分析数据结构，这样会对算法有个比较大体的认识，当然对于比较抽象的东西也没必要一开始就死磕其数据结构，先了解数据结构，然后在接下来分析的过程中提炼和总结数据结构，最后通过其存储数据结构把各个知识点串联起来。
SynchronousQueue有两种模式：

1、公平模式

所谓公平就是遵循先来先服务的原则，因此其内部使用了一个FIFO队列 来实现其功能。
2、非公平模式

最开始我以为非公平性和前面的重入锁可能会差不多，结果后来才知道，差别大了去了。
SynchronousQueue 中的非公平模式是默认的模式，其内部使用栈来实现其功能，也就是 后来的先服务,这个似乎也太不公平了，这个我们后面再来具体分析。

这里我们先分析公平的SynchronousQueue，因为这个简单许多。

```
/**
 * Shared internal API for dual stacks and queues.
 */
abstract static class Transferer<E> {
    /**
     * Performs a put or take.
     *
     * @param e if non-null, the item to be handed to a consumer;
     *          if null, requests that transfer return an item
     *          offered by producer.
     * @param timed if this operation should timeout
     * @param nanos the timeout, in nanoseconds
     * @return if non-null, the item provided or received; if null,
     *         the operation failed due to timeout or interrupt --
     *         the caller can distinguish which of these occurred
     *         by checking Thread.interrupted.
     */
    abstract E transfer(E e, boolean timed, long nanos);
}
```

Transferer是SynchronousQueue中的一个内部类，定义了一个转移方法（取数据或者存数据），后面的（非）公平 队列都实现了该类，**入队和出队 都会调用该方法**（本质上就是转移数据），可以指定超时操作，timed 为true 表示进行超时传递数据，nanos 是超时时间。

```
static final class TransferQueue<E> extends Transferer<E> {
...
}
```
TransferQueue 代表的是公平的SynchronousQueue，实现了Transferer接口，既然是队列，那么必然也有单元节点。

```
static final class TransferQueue<E> extends Transferer<E> {
	static final class QNode {
	...
	}
	 
	//队列头，初始化为指向一个"空"节点
    transient volatile QNode head;
    //队列尾   
    transient volatile QNode tail;
	//清除标记，后面再细说
    transient volatile QNode cleanMe;

    TransferQueue() {
            //会构造一个"空"的头
            QNode h = new QNode(null, false); // initialize to dummy node.
            head = h;
            tail = h;
     }
}
```
QNode是TransferQueue的内部类，代表的是队列中的一个存储节点，详细的定义见下面：

```
//队列节点定义
static final class QNode {
    // 指向队列中的下一个节点
    volatile QNode next;          // next node in queue
    //数据域
    volatile Object item;         // CAS'ed to or from null
    //等待的线程
    volatile Thread waiter;       // to control park/unpark
    //存储的是否是数据
    final boolean isData;

    QNode(Object item, boolean isData) {
        this.item = item;
        this.isData = isData;
    }

    //cas 操作，将next 由cmp 设置为val
    boolean casNext(QNode cmp, QNode val) {
        return next == cmp &&
                UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
    }
    //cas 操作，将 item 由cmp 设置为val
    boolean casItem(Object cmp, Object val) {
        return item == cmp &&
                UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
    }

    /**
     * Tries to cancel by CAS'ing ref to this as item.
     */
    //取消当前的操作 则会把item cas 设置为this 
    void tryCancel(Object cmp) {
        UNSAFE.compareAndSwapObject(this, itemOffset, cmp, this);
    }
    // 判断是否取消该操作，对比tryCancel
    boolean isCancelled() {
        return item == this;
    }

    /**
     * Returns true if this node is known to be off the queue
     * because its next pointer has been forgotten due to
     * an advanceHead operation.
     */
    //判断当前节点是否离开了队列，这个可以从后面的代码进行分析
    boolean isOffList() {
        return next == this;
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long itemOffset;
    private static final long nextOffset;

    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = QNode.class;
            itemOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("item"));
            nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

这个队列节点也不复杂，其casNext，casItem，tryCancel内部都用了cas算法，来实现原子操作。
SynchronousQueue和前面阻塞队列一个显著的差别就是没有使用锁，而是通过cas 算法来完成原子操作。

在SynchronousQueue 类中还有下面几个属性：
```
/** The number of CPUs, for spin control */
//当前cpu的数量
static final int NCPUS = Runtime.getRuntime().availableProcessors();

/**
 * The number of times to spin before blocking in timed waits.
 * The value is empirically derived -- it works well across a
 * variety of processors and OSes. Empirically, the best value
 * seems not to vary with number of CPUs (beyond 2) so is just
 * a constant.
 */
// 自旋最长时间
static final int maxTimedSpins = (NCPUS < 2) ? 0 : 32;

/**
 * The number of times to spin before blocking in untimed waits.
 * This is greater than timed value because untimed waits spin
 * faster since they don't need to check times on each spin.
 */

static final int maxUntimedSpins = maxTimedSpins * 16;

/**
 * The number of nanoseconds for which it is faster to spin
 * rather than to use timed park. A rough estimate suffices.
 */

static final long spinForTimeoutThreshold = 1000L;
```

这几个主要是用来设定自旋时间限的，在前面我们分析的同步器的实现方式中，我也看到过节点的自旋操作，在一点时间内自旋其消耗成本是低于阻塞的。 
首先阻塞是代价非常大的操作，要保存当前线程的很多数据，并且要切换上下文，等线程解释阻塞的时候还有切换回来。所以通常来说在阻塞之前都先自旋，自旋其实就是在一个循环里不停的检测是否有效，当然这要设定时间限。如果在时间限内通过自旋完成了操作。那就不需要去阻塞这也自然是最好的提高了响应速度。但是如果自旋时间限内还是能没能完成操作那就只有阻塞了。


### 构造方法

```
private transient volatile Transferer<E> transferer;

/**
 * Creates a {@code SynchronousQueue} with nonfair access policy.
 */
public SynchronousQueue() {
    //默认使用非公平队列实现
    this(false);
}

/**
 * Creates a {@code SynchronousQueue} with the specified fairness policy.
 *
 * @param fair if true, waiting threads contend in FIFO order for
 *        access; otherwise the order is unspecified.
 */
public SynchronousQueue(boolean fair) {
    //通过 fair 来实例化具体的实现类
    //TransferQueue 公平队列
    //TransferStack 非公平队列，用stack 来实现的，这个后面分析
    transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
}
```

SynchronousQueue 默认会使用非公平队列实现。


### 入队与出队（公平队列）
这是一个非常重要的部分，SynchronousQueue 中将出队和入队都在同一个方法中进行，同时没有用锁来控制，因此代码方面会显得比较复杂，因此需要仔细的分析。

1、入队方法
```
/**
 * Adds the specified element to this queue, waiting if necessary for
 * another thread to receive it.
 *
 * @throws InterruptedException {@inheritDoc}
 * @throws NullPointerException {@inheritDoc}
 */
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    //调用transfer 默认不进行超时等待，如果发生中断则抛出中断异常
    if (transferer.transfer(e, false, 0) == null) {
        Thread.interrupted();
        throw new InterruptedException();
    }
}
```

2、出队方法

```
/**
 * Retrieves and removes the head of this queue, waiting if necessary
 * for another thread to insert it.
 *
 * @return the head of this queue
 * @throws InterruptedException {@inheritDoc}
 */
public E take() throws InterruptedException {
    //调用transfer， 数据为null ，不超时等待
    E e = transferer.transfer(null, false, 0);
    if (e != null)
        return e;
    Thread.interrupted();
    throw new InterruptedException();
}
```

从上面入队和出队方法参数可以知道，入队和出队都是同一个方法，入队是数据部分是真实数据，出队时数据部分为空，也就是说transfer 内部肯定通过对参数数据进行了相应的判断来确认相关操作的,好下面重点看transfer 方法。
在看transfer 之前需要注意的是：transfer方法中没有使用锁，因此会多线程并发，一定要有这个意识。
```
E transfer(E e, boolean timed, long nanos) {

    QNode s = null; // constructed/reused as needed
    //判断e 是否是数据
    boolean isData = (e != null);

    //死循环
    for (;;) {
        //队列尾
        QNode t = tail;
        //队列头
        QNode h = head;
        //初始化的队头和队尾都是指向的一个"空" 节点，不会是null
        if (t == null || h == null)         // saw uninitialized value
            continue;                       // spin
        //h==t那么队列为空，上次和本次是同样的操作，也就是都是入队或者出队
        //因为每一个入队和出队是互相匹配的，相同的操作就是 入队的概念
        if (h == t || t.isData == isData) { // empty or same-mode
            //和上次一样的操作（put or take），则就是入队操作
            QNode tn = t.next; // 队尾的下一个节点
            // 存在并发，队列被修改了，从头开始
            if (t != tail)                  // inconsistent read
                continue;
            // 队列被修改了，tail 不是队尾，那么把tail 向后 向队尾推进。
            if (tn != null) {               // lagging tail
                // cas 设置队尾，期待值t,设置值tn
                advanceTail(t, tn);
                continue;
            }
            //如果进行了超时等待操作，发生超时则返回NULL
            if (timed && nanos <= 0)        // can't wait
                return null;
            if (s == null)
                s = new QNode(e, isData);  //构造一个新的节点
            // 将新节点 cas 设置成t的后继 ,失败则重来 
            if (!t.casNext(null, s))        // failed to link in
                continue;
            //添加了一个节点，推进队尾指针
            advanceTail(t, s);              // swing tail and wait
            //等待匹配该操作，线程会阻塞，直到有匹配的操作到来
            Object x = awaitFulfill(s, e, timed, nanos);
            //如果操作被取消
            if (x == s) {                   // wait was cancelled
                //清理
                clean(t, s);
                return null;
            }
            //匹配的操作到来，s操作完成，离开队列，如果没离开，使其离开
            if (!s.isOffList()) {           // not already unlinked
                //推进head ,cas将head 由t(是t不是h)，设置为s
                advanceHead(t, s);          // unlink if head
                if (x != null)              // and forget fields
                    s.item = s;
                s.waiter = null;
            }
            return (x != null) ? (E)x : e;

        } else {                            // complementary-mode
            //不同的模式那么是匹配的（put 匹配take ，take匹配put），出队操作
            //队头元素节点
            QNode m = h.next;               // node to fulfill
            //队列发生变化，重来
            if (t != tail || m == null || h != head)
                continue;                   // inconsistent read

            Object x = m.item;

            // 如果isData == (x != null) 为true 那么表示是相同的操作。
            //x==m 则是被取消了的操作
            //m.casItem(x, e) 将m的item设置为本次操作的数据域
            if (isData == (x != null) ||    // m already fulfilled
                    x == m ||                   // m cancelled
                    !m.casItem(x, e)) {         // lost CAS
                //推进head
                advanceHead(h, m);          // dequeue and retry
                continue;
            }

            //推进head
            advanceHead(h, m);              // successfully fulfilled
            LockSupport.unpark(m.waiter);  //唤醒匹配操作的线程
            return (x != null) ? (E)x : e;
        }
    }
}
```

简单看上面注释是不太容易的懂的，先来理一下整个流程，暂时不要太拘泥于细节，先把整体弄明白，然后再去挖细节。
主要逻辑描述：
SynchronousQueue 初始化是一个空队列，队列头尾指针都指向的是一个"空节点"，放进队列的不再是单独的数据，而是操作，这个操作可以是出队（take）操作，也可以是入队（put）操作，其中take和put 是互相匹配的操作，也就是说，如果本次是take(put),上次也是take(put)，那么这是相同的操作，需要放入队列，等待与其相应的匹配操作，如果本次是take(put),上次是put(take)，那么则是互相匹配的操作，那么就相当于出队（put的数据传递给了take,take需要的数据由put来提供）。

假设现在队列为空，来了一个take操作，因为存在线程的并发操作，因此需要检查数据的有效性，现在需要将take 操作入队，如果在入队期间，没有线程修改队列，那么成功将操作入队，同时更新队列尾指针，如果有线程修改了队列，那么在重新开始进行操作，入队后，当前线程经过一定时间的自旋后就阻塞了，等到匹配的操作到来，现在又来了一个take 操作，本次的操作和上次（队尾指向的操作）都是相同的（模式相同），那么进行重复上次的入队操作，线程自旋后阻塞，现在队列中有两个节点，同时有两个线程也阻塞了，现在又来一个put 操作，本次操作和上次是不一样的，因此这两个操作就是匹配的（队列里面放的肯定都是相同操作的“数据”），那么现在就开始从队列头开始进行匹配，如果再匹配过程中，其它现在也进行了匹配，修改了队列，那么尝试更新队列头指针，然后重新再开始匹配，如果队头指针的操作和这次操作匹配，那么就出队（推进头指针），然后唤醒该匹配操作阻塞的线程，两个操作顺利匹配完成，现在队列中还有一个操作等待匹配（take）,继续重复上面的过程。通过这样简单叙述一遍大致过程，不知道有没有帮助大家理解。

#### 更新队尾指针
```
/**
 * Tries to cas nt as new tail.
 */
void advanceTail(QNode t, QNode nt) {
    if (tail == t)
        UNSAFE.compareAndSwapObject(this, tailOffset, t, nt);
}
```

这个就是cas更新队列尾指针的方法，将队列尾指针，由原来的t,设置为nt，更新头指针的方法类似。

#### 更新队头指针
```
/**
 * Tries to cas nh as new head; if successful, unlink
 * old head's next node to avoid garbage retention.
 */
void advanceHead(QNode h, QNode nh) {
    if (h == head &&
            UNSAFE.compareAndSwapObject(this, headOffset, h, nh))
        h.next = h; // forget old next 注意这里
}
```

更新队头指针后，会将原来的head的next 指向自己，形成一个环状，方便gc回收，同时也表示离开了队列。

#### 阻塞等待（advanceTail）
```
/**
 * Spins/blocks until node s is fulfilled.
 *
 * @param s the waiting node
 * @param e the comparison value for checking match
 * @param timed true if timed wait
 * @param nanos timeout value
 * @return matched item, or s if cancelled
 */
// s 是刚刚入队的操作节点，e是操作（也可说是数据）
Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
        /* Same idea as TransferStack.awaitFulfill */
    // 等待超时时间
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    //当前操作线程
    Thread w = Thread.currentThread();
    //自旋次数
    int spins = ((head.next == s) ?
            (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    for (;;) {
        //如果当前线程发生中断，那么尝试取消该操作
        if (w.isInterrupted())
            s.tryCancel(e); //将item 设置成了s(this)
        Object x = s.item;
        // s.item ！=e则返回，生成s节点的时候，s.item是等于e的，当取消操作或者匹配了操作的时候会进行更改
        if (x != e)
            return x;
        //如果设置了超时等待    
        if (timed) {
            //还剩的等待时间
            nanos = deadline - System.nanoTime();
            //发生了超时，尝试取消该操作
            if (nanos <= 0L) {
                s.tryCancel(e);
                continue;
            }
        }
        //自旋控制
        if (spins > 0)
            --spins;
            //设置等待线程 waiter    
        else if (s.waiter == null)
            s.waiter = w;
        else if (!timed) //如果没设置操作，那么就"永久性"阻塞该线程
            LockSupport.park(this);
        else if (nanos > spinForTimeoutThreshold) //设置了超时，那么就进行超时阻塞
            LockSupport.parkNanos(this, nanos);
    }
}
```

这个方法应该比较好理解，就是等待匹配操作的到来，如果设置了超时等待，那么就等待一定时间，如果发生超时，那么就取消该操作，如果没有设置超时等待，那么就一直等待，直到匹配的操作到来，或者发生中断（取消该操作）。

```
/**
 * Tries to cancel by CAS'ing ref to this as item.
 */
void tryCancel(Object cmp) {
    UNSAFE.compareAndSwapObject(this, itemOffset, cmp, this);
}
```

这个就是取消操作，就是打一个标记，设置item 为this，这样可以区分。
在匹配操作到来后，也会修改item ,那么这样后，返回的条件就是s的item发生了变化（这个变化可以是自己修改的，也可以被其它线程修改的），这个还需要结合后面的匹配操作来分析。

好，到这里，相同的操作进行入队的操作就完成了，真正情况下操作线程需要在这里阻塞，等待匹配操作到来，因此后面的代码暂时就不分析，我们先来看看匹配操作。

下面是transfer方法中的匹配操作
```
//队列第一个操作节点
  QNode m = h.next;               // node to fulfill
  //如果队列发生变化，重新来
if (t != tail || m == null || h != head)
          continue;                   // inconsistent read

  //item 域
  Object x = m.item;
if (isData == (x != null) ||    // m already fulfilled
  x == m ||                   // m cancelled
          !m.casItem(x, e)) {         // lost CAS
      advanceHead(h, m);          // dequeue and retry
      continue;
  }

  //匹配成功，推进head
  advanceHead(h, m);              // successfully fulfilled
  // 唤醒m节点操作的线程
LockSupport.unpark(m.waiter);
return (x != null) ? (E)x : e;
```

这里需要讲的应该就是中间那段代码了，现在假设队列中有一系列相同的操作节点，现在有一个不一样操作到来，也就是匹配操作到来了，因为现在是公平队列，因此需要从队列头开始，进行出队操作，那么此时的操作和队列中第一个节点的操作（m）肯定是匹配（不一样的操作），因为这里存在并发情况，因此当前线程在匹配的时候，也许其它线程把线程修改了，因此需要进行判断，如果当前的操作和m的操作是一致的，那么m就是已经被其它线程匹配了（假设当前操作是take，那么isData则为false,那么m操作则是put，x!=null 是true的），可以这样想象一下，假设当前是take，队列里面有一个put操作，线程这个take和队列里面的put是匹配的，但是假设当前线程执行缓慢，被其它线程的take操作抢先了，现在队列就空了，然后还有其它的线程往队列里面放了take操作，那么现在这个线程判断的结果就是isData == (x != null)则为true，现在不是匹配的操作，也就是m已经被其它线程匹配了，那么就尝试更新队列头（失败了也无所谓），然后重新再来（也许此时将会执行入队操作而不是匹配操作了）
同样，如果x == m 这是说明取消了该操作（tryCancel 中设置的），那么同样需要更新头，重新再来。

如果前面两个判断都是false,那么就需要m.casItem(x, e)，将m的item域cas 设置为e（e代表的是本次操作，如果是put，那么就是数据，如果是take，那么就是null）,如果设置失败，则m的item 被其它线程设置了，也就是m被其它线程匹配了，因此也需要更新头，重新再来。

前面这些操作对当前操作线程来说都是噩梦，因为竞争太激烈了，如果很顺利，那么当前操作就可以直接匹配上队列的第一个操作节点，那么直接推进head，返回数据就可以了。

现在匹配完成，阻塞的线程被唤醒，现在我又再次回到开始那里，还剩一点没有分析。

```
Object x = awaitFulfill(s, e, timed, nanos);
if (x == s) {                   // wait was cancelled
       clean(t, s); //清除操作
       return null;
   }

if (!s.isOffList()) {           // not already unlinked
       advanceHead(t, s);          // unlink if head
       if (x != null)              // and forget fields
           s.item = s;
       s.waiter = null;
   }
return (x != null) ? (E)x : e;
```

还记得awaitFulfill 返回的是什么的，x就是s的item，只是这里是被修改的item（在匹配操作中被修改的），如果当前是put操作，那么匹配的是take操作，take将其item设置为null,如果当前是take，那么匹配的是put，那么put操作把item设置为了数据。
如果当前操作取消了，那么x就是this（也就是这里的s），那么就需要进行清除操作（这个稍后再分析）。

在前面匹配的代码中我们知道，如果匹配成功后，需要推进head，也就是把那个节点出队，因此在这里阻塞线程返回后，操作节点s应该都离开了队列，如果还没有离开，那么帮忙使其离开，然后将item设置成自己 this，表示操作完成。

```
boolean isOffList() {
    return next == this;
}
```

如果节点的next 等于自身，也就是成环，那么就表示离开了队列，这个在前面advanceHead 里面有设置的哟，如果忘了，可以回去看看。

**注意：这里 advanceHead(t, s);，参数是t而不是h,t代码的是队尾指针，那么怎么是队尾呢，不是队头指针呢？**
这个其实很好理解，t开始是队尾，然后将操作s入队后，那么s就是t的后继了，现在队列阻塞，线程数据保存（此时 t不是队尾，t也不会变了，队尾会变），当其他线程的匹配操作到来时，是从头开始匹配的，而head 始终是一个"空"节点，其next域才是真正的队头，当head的next等于s的时候，那么此时说明匹配到了s了，那么s的线程才会被唤醒，因为t的next是s,因此head和t是同一个，因此这里的参数才是t。

基本上到这里基本就对公平的SynchronousQueue比较了解了，现在我们还有一个clean 方法没有分析了。
在前面我们知道，有些时候某个操作可能会被取消，那么这个操作就需要从队列中移除，但是这个移除并不简单，因为这里面没有使用锁，因此竞争会比较激烈，如果需要移除的操作不是队尾，那么可以直接移除掉，如果是队尾，那么就不太安全了，因为可能有其它线程正在执行入队操作，而入队是放在队尾的，因此直接删除队尾，可能会导致其它线程入队操作不能正确的加入队列，因此这里有一个标记的过程，我们一起来看看。

```
Object x = awaitFulfill(s, e, timed, nanos);
if (x == s) {                   // wait was cancelled
	clean(t, s);
	return null;
}
```

```
/**
 * Gets rid of cancelled node s with original predecessor pred.
 */
// pred 是s的前驱
void clean(QNode pred, QNode s) {

    s.waiter = null; // forget thread
    while (pred.next == s) { // Return early if already unlinked
        QNode h = head;
        QNode hn = h.next;   // Absorb cancelled first node as head
        //操作取消了，那么推进head
        if (hn != null && hn.isCancelled()) {
            advanceHead(h, hn);
            continue;
        }
        QNode t = tail;      // Ensure consistent read for tail
        //t==h 那么队列为空了
        if (t == h)
            return;
        QNode tn = t.next;
        //t！=tail ，队列被修改了
        if (t != tail)
            continue;
        //tn 理应为null ,如果不为空，说明其它线程进行了入队操作，更新tail    
        if (tn != null) {
            advanceTail(t, tn);
            continue;
        }
        // s!=t ,则s不是尾节点了，本来最开始是尾节点，其它线程进行了入队操作
        if (s != t) {        // If not tail, try to unsplice
            QNode sn = s.next;
            // 如果s.next ==s ,则已经离开队列
            //设置pred的后继为s的后继，将s从队列中删除
            if (sn == s || pred.casNext(s, sn))
                return;
        }
        // 到这里，那么s就是队尾，那么暂时不能删除
        //cleanMe标识的是需要删除节点的前驱
        QNode dp = cleanMe;
        //有需要删除的节点
        if (dp != null) {    // Try unlinking previous cancelled node
            QNode d = dp.next;
            QNode dn;
            if (d == null ||               // d is gone or
                    d == dp ||                 // d is off list or
                    !d.isCancelled() ||        // d not cancelled or
                    (d != t &&                 // d not tail and
                            (dn = d.next) != null &&  //   has successor
                            dn != d &&                //   that is on list
                            dp.casNext(d, dn)))       // d unspliced
                //将cleanMe标记设为null
                casCleanMe(dp, null);
            //dp==pred 表示已经被设置过了
            if (dp == pred)
                return;      // s is already saved node
        } else if (casCleanMe(null, pred)) //将cleanMe 设置成pred
            return;          // Postpone cleaning s
    }
}
```

先大致说一下这里的逻辑：
回想一下链表的删除，链表的删除是不是很简单，只需要设置节点指针的关系就可以了，这个clean的方法就在链表基础上增加了一个逻辑：就是如果删除的节点不是尾节点，那么可以直接进行删除，如果删除的节点是尾节点，那么用cleanMe标记需要删除的节点的前驱（为什么需要这样，在前面分析过），这样在**下一轮的clean**的过程将会清除打了标记的节点。
1、pred.next == s 表示节点s还在队列中
2、从队头开始，如果节点被取消，那么则推进head。
3、如果队列发生变化，则重新开始或者推进tail.
4、如果删除的节点不是尾节点，那么进行cas 删除操作
5、删除的节点是尾节点，那么需要先检查cleanMe是否已经被标记了
6、如果cleanMe已经被标记了，那么检查标记是否还有效
7、cleamMe 失效的情况有：(1)cleanMe的后继而空（cleanMe 标记的是需要删除节点的前驱），(2)cleanMe的后继等于自身（这个前面有分析过）,(3)需要删除节点的操作没有被取消，(4)被删除的节点不是尾节点且其后继节点有效。
8、如果cleanMe没有被标记，那么就标记为被删除节点的前驱。

### 公平队列SynchronousQueue 总结

SynchronousQueue  公平队列我们就分析完了，文字描述比较多，不知道阐述清楚没有。
公平的SynchronousQueue  使用的队列来实现的，其内部没有使用传统的锁来控制，而是通过CAS来完成，因此需要反复检查状态是否有效，SynchronousQueue  将**相同的操作（这个操作可以是put也可以是take）入队，不同的操作（put 对应take,take 对应put）**那么就是互相匹配的操作，当请求的操作和队尾的操作相同时入队，否则进行出队匹配，如果有超时设置，那么就进行超时操作，入队后的线程被阻塞，直到匹配的操作到来，如果有取消操作需要被清除，如果该操作节点不是尾节点，那么执行删除操作，否则用cleanMe标记其父节点，在下一轮的clean过程中再根据情况进行删除。
现在回过头来再看，是不是也不是那么难了，先弄清楚其大致流程，再去抓细节，公平的SynchronousQueue 基本就这样了，我们再接着看非公平的SynchronousQueue，这个比公平的SynchronousQueue还要难理解一点。


## 非公平SynchronousQueue
非公平的SynchronousQueue是用栈来实现的，我们知道传统的队列，队尾放数据，队头取数据，但是栈总是在栈顶操作数据，这就带来一个很麻烦的问题，这里我先把这个问题抛出，这样后面可能会好理解一点，这个问题和在前面队列的clean 中的方法是一个道理，现在操作栈顶的可能不止一个线程，既要在栈顶放数据，又要取数据，那么如何才能保证其线程安全?,在前面如果要删除队列尾的节点，那么暂时是不会删的（原因见前面分析），先标记，在该节点为非队尾的情况下再进行删除。
非公平的SynchronousQueue中的栈也有这个道理，其内部的做法大致为：**将一个操作和栈顶进行匹配，如果和栈顶是相同的操作，那么就直接入栈，如果和栈顶不是相同的操作（也就是匹配的操作，take匹配put,put匹配take）,那么现在先不急出栈，因为此时可能有线程真正入栈，为了避免出现操作错误，这里加了一个环节，如果操作是匹配的(即需要出栈)，那么入栈一个节点，并标记是真正匹配状态，表示的是栈顶操作节点真正匹配，如果其他线程发现这个过程，那么就会帮助其匹配（使其顺序完成出栈工作），完成匹配过后，再进行自身的操作**。这是一个非常重要的技巧。
不使用锁（阻塞式算法）来保证数据结构的完整性，还要确保其他线程不仅能够判断出第一个线程已经完成了更新还是处在更新的中途，还能够判断出如果第一个线程操作状态，完成更新还需要什么操作。如果线程发现了处在更新中途的数据结构，它就可以 “帮助” 正在执行更新的线程完成更新，然后再进行自己的操作。当第一个线程回来试图完成自己的更新时，会发现不再需要了，返回即可。
### 数据结构
前面说了一大堆，现在正是开始分析非公平的SynchronousQueue。
下面的TransferStack 代码中只贴出来域的定义，有些cas的操作和前面队列是差不多的，还有些重要的方法我们后面分析再给出。
```
static final class TransferStack<E> extends Transferer<E> {
    /*
     * This extends Scherer-Scott dual stack algorithm, differing,
     * among other ways, by using "covering" nodes rather than
     * bit-marked pointers: Fulfilling operations push on marker
     * nodes (with FULFILLING bit set in mode) to reserve a spot
     * to match a waiting node.
     */

    /* Modes for SNodes, ORed together in node fields */
    /** Node represents an unfulfilled consumer */
    //REQUEST表示消费者（take）
    static final int REQUEST    = 0;
    /** Node represents an unfulfilled producer */
    //DATA表示生产者(put 数据)
    static final int DATA       = 1;
    /** Node is fulfilling another unfulfilled DATA or REQUEST */
    //表示该操作节点处于真正匹配状态
    static final int FULFILLING = 2;
    //栈顶，初始为null
    volatile SNode head;

    /** Returns true if m has fulfilling bit set. */
    //是否处于匹配状态
    static boolean isFulfilling(int m) { return (m & FULFILLING) != 0; }

    /** Node class for TransferStacks. */
    static final class SNode {
        volatile SNode next;        // next node in stack 后继指针
        volatile SNode match;       // the node matched to this 匹配节点
        volatile Thread waiter;     // to control park/unpark 操作线程
        Object item;                // data; or null for REQUESTs 数据域
        //模式：REQUEST or DATA or FULFILLING
        int mode;
        // Note: item and mode fields don't need to be volatile
        // since they are always written before, and read after,
        // other volatile/atomic operations.

        SNode(Object item) {
            this.item = item;
        }
        ...

    }

    /**
     * Creates or resets fields of a node. Called only from transfer
     * where the node to push on stack is lazily created and
     * reused when possible to help reduce intervals between reads
     * and CASes of head and to avoid surges of garbage when CASes
     * to push nodes fail due to contention.
     */
    //生成栈节点
    static SNode snode(SNode s, Object e, SNode next, int mode) {
        if (s == null) s = new SNode(e);
        s.mode = mode;
        s.next = next;
        return s;
    }
    ...
}
```

TransferStack中定义了三个状态：REQUEST表示消费者，DATA表示生产者，FULFILLING，表示操作匹配状态。任何线程对TransferStack的操作都属于上述3种状态中的一种（对应着SNode节点的mode）。同时还包含一个head域，表示栈顶。

### 入队与出队
其内部都是调用的同一个方法，注意参数的变换，前面讲过，这里就不细说了。
```
public void put(E e) throws InterruptedException {
     if (e == null) throw new NullPointerException();
     if (transferer.transfer(e, false, 0) == null) {
         Thread.interrupted();
         throw new InterruptedException();
     }
 }

public E take() throws InterruptedException {
    E e = transferer.transfer(null, false, 0);
    if (e != null)
        return e;
    Thread.interrupted();
    throw new InterruptedException();
}
```
#### transfer 方法
transfer 方法 中部分方法和前面公平的SynchronousQueue差不多，因此可能不再分析那么细。

```
E transfer(E e, boolean timed, long nanos) {
    SNode s = null; // constructed/reused as needed
    //操作模式（put or take）
    int mode = (e == null) ? REQUEST : DATA;
    //这里无限循环
    for (;;) {
        SNode h = head;
        //相同的操作模式
        if (h == null || h.mode == mode) {  // empty or same-mode
            if (timed && nanos <= 0) {      // can't wait
                if (h != null && h.isCancelled())  //节点取消，则弹出节点
                    casHead(h, h.next);     // pop cancelled node
                else
                    return null;
            } else if (casHead(h, s = snode(s, e, h, mode))) { //将操作生成节点，入队
                //等待匹配操作
                SNode m = awaitFulfill(s, timed, nanos);
                if (m == s) {               // wait was cancelled
                    clean(s);               //操作取消，清理节点
                    return null;
                }
                //s 还没有离开栈，帮助其离开
                if ((h = head) != null && h.next == s)
                    casHead(h, s.next);     // help s's fulfiller
                return (E) ((mode == REQUEST) ? m.item : s.item);
            }
        } else if (!isFulfilling(h.mode)) { // try to fulfill  不同的模式，并且没有处于正在匹配状态，则进行匹配
            if (h.isCancelled())            // already cancelled 操作节点取消，更新head
                casHead(h, h.next);         // pop and retry
            else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) { //入队一个新节点，并且处于匹配状态（表示h正在匹配）
                for (;;) { // loop until matched or waiters disappear
                    SNode m = s.next;       // m is s's match   s.next 是真正的操作节点
                    if (m == null) {        // all waiters are gone 什么状况下出现这种情况？ 目前还没想到
                        casHead(s, null);   // pop fulfill node
                        s = null;           // use new node next time
                        break;              // restart main loop
                    }
                    SNode mn = m.next;
                    if (m.tryMatch(s)) {    //m和s进行匹配
                        casHead(s, mn);     // pop both s and m 匹配成功，移除匹配节点和操作节点
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    } else                  // lost match
                        s.casNext(m, mn);   // help unlink  没有匹配成功，说明有其他线程已经匹配了，把m移出
                }
            }
        } else {                            // help a fulfiller  头结点是真正匹配的状态，那么就帮助它匹配
            SNode m = h.next;               // m is h's match
            if (m == null)                  // waiter is gone
                casHead(h, null);           // pop fulfilling node
            else {
                SNode mn = m.next;
                if (m.tryMatch(h))          // help match  帮助匹配
                    casHead(h, mn);         // pop both h and m
                else                        // lost match
                    h.casNext(m, mn);       // help unlink
            }
            //帮助完成后，继续回到循环，进行自己的操作
        }
    }
}
```

1、如果当前栈为空获取节点模式与栈顶模式一样，则尝试将节点加入栈内，同时通过阻塞（或自旋一段时间，如果有超时设置，则进行超时等待）等待节点匹配，最后返回匹配的节点或者本身（被取消）

2、 如果栈不为空且节点的模式与首节点模式匹配，则尝试将该节点**打上FULFILLING标记，然后加入栈中**，与相应的节点匹配，成功后将这**两个节点弹出栈**并返回匹配节点的数据

3、如果有节点在匹配，那么**帮助这个节点完成匹配和出栈操作**，然后在主循环中继续执行。


awaitFulfill 方法和前面的大同小异，这个可以自行去看看分析，这里就不再分析了。

#### tryCancel 方法
```
//设置match 为自身，这个在公平的SynchronousQueue设置的是item域
void tryCancel() {
    UNSAFE.compareAndSwapObject(this, matchOffset, null, this);
}

boolean isCancelled() {
    return match == this;
}
```

#### tryMatch 方法
```
/**
 * Tries to match node s to this node, if so, waking up thread.
 * Fulfillers call tryMatch to identify their waiters.
 * Waiters block until they have been matched.
 *
 * @param s the node to match
 * @return true if successfully matched to s
 */
boolean tryMatch(SNode s) {
    //将 match cas设置成s
    if (match == null &&
            UNSAFE.compareAndSwapObject(this, matchOffset, null, s)) {
        Thread w = waiter;
        if (w != null) {    // waiters need at most one unpark
            waiter = null;
            LockSupport.unpark(w); //唤醒阻塞的操作线程
        }
        return true;
    }
    // 有可能已经被其它线程抢先匹配了，因此这里返回match == s
    return match == s;
}
```

如果当前match不为null则说明当前任务完成或者取消了（取消是设置的match 等于操作节点自身），CAS则尝试匹配这两个任务（设置matcht）。如果成功则返回true并用LockSupport.unpark（w）去解锁当前的线程。并返回true。否则需要返回match == s，因为存在并发，可能被其它线程抢先匹配了。

#### clean 方法
```
/**
 * Unlinks s from the stack.
 */
void clean(SNode s) {
    s.item = null;   // forget item
    s.waiter = null; // forget thread
    SNode past = s.next;   //被删除节点的后继
    //后继节点操作被取消，直接移除该节点（感觉加个while 岂不是更好）
    if (past != null && past.isCancelled())
        past = past.next;

    // Absorb cancelled nodes at head
    SNode p;
    //如果栈顶是取消了的操作节点，则移除
    while ((p = head) != null && p != past && p.isCancelled())
        casHead(p, p.next);

    // Unsplice embedded nodes
    //因为是单向链表，因此需要从head 开始，遍历到被删除节点的后继
    while (p != null && p != past) {
        SNode n = p.next;
        if (n != null && n.isCancelled())
            p.casNext(n, n.next); //移除节点n
        else
            p = n;
    }
}
```

单纯来看这个方法还是好理解，被删除节点的后继如果也被取消了，那么就先删除其后继，如果head操作节点也被取消，那么就重新更新头，因为是单链表，因此需要遍历链表，从head 到s的后继中，有被取消了的操作的节点，那么就移除掉。
我们知道**在公平的SynchronousQueue 中的clean 方法会进行打标记，不会立即删除，但是这里为什么都是直接删除了的呢**，其实你可以脑袋了模拟跑几个线程就明白了。
在公平的SynchronousQueue 中，如果删除节点是尾节点，如果在移除尾节点过程中，有操作添加到了尾节点上，那么移除尾节点后，添加的操作节点也丢失了，因此需要打标记。
而在这里添加是从头开始，如果删除的节点不是头，这个很好理解，直接删除可以了（cas 删除），如果是头，那么就cas 设置头，但是此时如果添加了操作，那么都会cas 头，此时就只有一个能成功，因此这个就由cas 来保证了线程的安全。

### TransferStack 总结 
在非公平的SynchronousQueue 中，内部大致是这样做的:
将一个操作和栈顶进行匹配，如果和栈顶是相同的操作，那么就直接入栈，如果和栈顶不是相同的操作（也就是匹配的操作，take匹配put,put匹配take）,那么现在先不急出栈，因为此时可能有线程真正入栈，为了避免出现操作错误，这里加了一个环节，如果操作是匹配的(即需要出栈)，那么入栈一个节点，并标记是真正匹配状态，表示的是栈顶操作节点真正匹配，如果其他线程发现这个过程，那么就会帮助其匹配（使其顺序完成出栈工作），完成匹配过后，再进行自身的操作.
个人觉得这个比公平的SynchronousQueue要难理解一点，尤其是最开始，不懂大致流程的情况下，其主要是在需要出队的操作节点前面打了一个匹配标记，大家肯定会想为什么要这样做，这样就容易产生困惑，因此我这里的分析，都是先说大致流程，最后再来分析代码，免得一来就看代码，大家都晕。

## SynchronousQueue 总结

SynchronousQueue 最初还是看得比较吃力，debug了几次源码，分析了一下数据结构，然后对大体就比较了解了，我选择从公平的SynchronousQueue 开始分析，因为一般公平性的算法比非公平性的算法要简单些，同时队列一般要比栈相对要容易些。

1、SynchronousQueue  有两种实现，一个是公平的，一个是非公平的，公平的是用队列来实现的，非公平的是用栈来实现的，但是不知道大家有没有感觉到，不公平的SynchronousQueue  似乎太不公平的，如果竞争很激烈，同时连续操作相同模式的线程很多，那么最先来的线程，阻塞得最久，同时必须等前面匹配完了，才轮得到自己。

2、SynchronousQueue 中的线程安全是通过CAS 来实现的，没有使用类似前面的可重入锁，因此代码方面就显得比较复杂，需要不断的判断数据的时效性，不过从这里我们也可以学到非阻塞式算法的一些思想。

3、SynchronousQueue  存储了数据，同时维护了操作线程，其最主要的作用是传递信息，及时传递通知，而不是简单的存储数据，因此SynchronousQueue 遍历队列（栈）的方法没意义。

```
public boolean retainAll(Collection<?> c) {
    return false;
}
public E peek() {
    return null;
}
public Iterator<E> iterator() {
    return Collections.emptyIterator();
}
```

4、这篇文章 文字描述应该很多，主要是个人觉得不太好理解，而且网上很多文章也没说清楚（当然，可能在下愚钝了），因此这里我都是先流程后代码，希望看的朋友能明白，后面可能会补一个流程图，帮助理解，文中难免会有理解错误的地方，完全吃透这个现在还达不到，因此如果有错误，还望各位同仁指出。