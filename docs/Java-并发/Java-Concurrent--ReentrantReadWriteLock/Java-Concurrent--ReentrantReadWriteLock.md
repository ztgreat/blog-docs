在我们分析了AbstractQueuedSynchronizer（同步器）之后，分析了ReentrantLock，ReentrantLock内部组合了同步器来完成同步操作，从源码中我们知道ReentrantLock是排它锁（独占锁），这些锁在同一时刻只允许一个线程进行访问，今天我们来分析基于同步器实现的另一个同步组件ReentrantReadWriteLock（读写锁）。

**本文需要有同步器知识的基础，同时也了解ReentrantLock 最好**，可以参考前面写的内容：

[Java 并发 ---AbstractQueuedSynchronizer(同步器)-独占模式](http://blog.ztgreat.cn/article/7)

[Java 并发 ---AbstractQueuedSynchronizer-共享模式与Condition](http://blog.ztgreat.cn/article/9)

[Java 并发 ---ReentrantLock源码分析](http://blog.ztgreat.cn/article/14)

 [Java 并发 ---ThreadLocal源码分析](http://blog.ztgreat.cn/article/11)

### 介绍（jdk 1.8）
读写锁维护着一对锁，一个读锁和一个写锁，读写锁在同一时刻允许多个读线程访问，但是在写线程访问时，所有的读线程和其它写线程均被阻塞（独占），在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量。

### 继承体系
ReentrantReadWriteLock有五个内部类，内部类的关系如下图所示
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171013230102876.png)

说明：如上图所示，Sync继承自AQS、NonfairSync继承自Sync类、FairSync继承自Sync类；ReadLock实现了Lock接口、WriteLock也实现了Lock接口
类中的一些属性会在后面讲到，可以最后再回来看这个类图梳理一下结构

### 使用
第一步我们需要知其然，知道如何使用读写锁。

```
public class ReentrantReadWriteLockTest {

    private static  Map<String ,Object>map=new HashMap<String, Object>();

    private static ReentrantReadWriteLock lock=new ReentrantReadWriteLock();

    private  static Lock readLock=lock.readLock();

    private  static Lock writeLock=lock.writeLock();

    public Object get(String key){
        readLock.lock();
        try{
            return map.get(key);
        }finally {
            readLock.unlock();
        }
    }
    public Object put(String key,Object value){
        writeLock.lock();
        try {
            return map.put(key,value);
        }finally {
            writeLock.unlock();
        }
    }
}
```
使用读写锁来保证对非线程安全的HashMap的操作线程安全化。在读操作get(String key)方法中，需要获取读锁，这使得并发访问该方法时不会被阻塞，写操作put(String key,Object value) 时必须获取写锁，当获取写锁后，其他线程对于读锁和写锁的获取都会被阻塞。

### 读写状态的设计
读写锁依赖自定义的同步器来实现同步功能，而读写状态就是其同步器的同步状态，在ReentrantLock 中，其同步状态表示锁被一个线程重复获取的次数，而读写锁的自定义同步器需要在同步状态（一个整型变量）上维护多个读线程和一个写线程的状态，基于这种需求，我们需要对用整型表示的同步状态进行分割，切成两部分，一部分表示读状态，一部分表示写状态。

```
static final int SHARED_SHIFT   = 16;
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);

/**最多支持65535个写锁和65535个读锁；低16位表示写锁计数，高16位表示持有读锁的线程数*/
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;

/**写锁的掩码，用于状态的低16位*/
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

/** Returns the number of shared holds represented in count（读锁,高16位）  */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** Returns the number of exclusive holds represented in count （写锁计数,低16位） */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

```
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171008225424261.png)

### 读锁

#### HoldCounter

```
/**
* A counter for per-thread read hold counts.
* Maintained as a ThreadLocal; cached in cachedHoldCounter
*/
static final class HoldCounter {
	int count = 0;
	// Use id, not reference, to avoid garbage retention
	final long tid = getThreadId(Thread.currentThread());
}
```
HoldCounter 是一个静态内部类，从注释中我们就知道了，它是用来记录每个线程获取读锁的次数的，其内部的count的就是次数，tid用于关联线程，既然与线程相关的数据，那么用ThreadLocal是最简单的方式了

```
/**
* ThreadLocal subclass. Easiest to explicitly define for sake
* of deserialization mechanics.
*/
static final class ThreadLocalHoldCounter extends ThreadLocal<HoldCounter> {
	public HoldCounter initialValue() {
		return new HoldCounter();
	}
}
```
ThreadLocalHoldCounter也是其中的一个静态内部类，通过该类，确实可以看出HoldCounter 通过 ThreadLocal 和线程绑定在了一起，对于ThreadLoal 我们在前面分析过，可以参考：
[Java 并发 ---ThreadLocal源码分析](http://blog.ztgreat.cn/article/11)


除此之外，我们还需要注意一点

```
private transient Thread firstReader = null;
private transient int firstReaderHoldCount;
```
对于第一个线程获取读锁，是不会生成HoldCounter，会用上面的变量进行记录，主要原因我想还是为了效率，通过ThreadLocal把数据和线程绑定在一起，但是这个取数据是需要开销的，通过这个变量直接记录，如果再第一个线程频繁访问的情况下，是可以很好的提高效率的。


#### 读锁的获取
读锁的获取通过ReadLock的lock()方法：
```
/**
* Acquires the read lock.
*
* <p>Acquires the read lock if the write lock is not held by
* another thread and returns immediately.
*
* <p>If the write lock is held by another thread then
* the current thread becomes disabled for thread scheduling
* purposes and lies dormant until the read lock has been acquired.
*/
public void lock() {
	sync.acquireShared(1);
}
```
其内部调用同步器的acquireShared 方法，因为读锁可以并发，因此使用的是同步器的共享模式而不是独占模式。
```
public final void acquireShared(int arg) {
	if (tryAcquireShared(arg) < 0)
		doAcquireShared(arg);
}
```
同步器的acquireShared 会调用tryAcquireShared 方法，而这个方法会在我们自定义同步器中进行重写，实现如何获取同步状态，因此这个方法至关重要，看看

```
protected final int tryAcquireShared(int unused) {
    /*
    * Walkthrough:
    * 1. If write lock held by another thread, fail.
    * 2. Otherwise, this thread is eligible for
    *    lock wrt state, so ask if it should block
    *    because of queue policy. If not, try
    *    to grant by CASing state and updating count.
    *    Note that step does not check for reentrant
    *    acquires, which is postponed to full version
    *    to avoid having to check hold count in
    *    the more typical non-reentrant case.
    * 3. If step 2 fails either because thread
    *    apparently not eligible or CAS fails or count
    *    saturated, chain to version with full retry loop.
    */
    Thread current = Thread.currentThread();
    int c = getState();
    //exclusiveCount(c)计算写锁
    //如果存在写锁，且锁的持有者不是当前线程，直接返回-1
    //存在锁降级问题，后面会讨论
    if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
        return -1;
    //读锁状态
    int r = sharedCount(c);
            /*
             * readerShouldBlock():读锁是否需要等待（公平锁和非公平锁）
             * r < MAX_COUNT：持有线程小于最大数（65535）
             * compareAndSetState(c, c + SHARED_UNIT)：设置读取锁状态
             */
    if (!readerShouldBlock() &&
            r < MAX_COUNT &&
            compareAndSetState(c, c + SHARED_UNIT)) {
        //还没有线程获取过读锁
        if (r == 0) {
            //记录第一次获取读锁的线程
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            //获取的次数
            firstReaderHoldCount++;
        } else {

            HoldCounter rh = cachedHoldCounter;
            //判断上次缓存的线程是否是当前线程
            if (rh == null || rh.tid != getThreadId(current))
                //从ThreadLocal 中获取值
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
```

主要逻辑：

（1）获取同步状态和当前线程

（2）如果写锁状态不为0，并且写锁的占有者不是当前线程那么获取锁失败（这么明明是读锁的操作，为什么会有写锁参与，其实这里有一个锁降级的问题，后面讨论）

```
/** Returns the number of exclusive holds represented in count （写锁计数,低16位） */
static int exclusiveCount(int c) { 
	return c & EXCLUSIVE_MASK; 
}
```
（3）获取读锁状态

```
/** Returns the number of shared holds represented in count（读锁,高16位）  */
static int sharedCount(int c)    { 
	return c >>> SHARED_SHIFT;
}
```
（4）判断读锁是否应该阻塞，如果否并且没有达到最大数量限制情况下，会进行获取锁（cas操作），判断读锁是否该阻塞，因为有公平锁和非公平锁，因此会有两种不同的情况，在公平锁中 同ReentrantLock 中的公平锁一样，需要按照同步器的FIFO规则获取，只有该线程的前继是头结点才能获取锁。

```
/**
 * Fair version of Sync
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = -2274990926593161451L;
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    }
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}
```

在非公平锁中就和ReentrantLock 中的非公平锁不一样了

```
/**
 * Nonfair version of Sync
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -8159625535654395037L;
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
    final boolean readerShouldBlock() {
            /* As a heuristic to avoid indefinite writer starvation,
             * block if the thread that momentarily appears to be head
             * of queue, if one exists, is a waiting writer.  This is
             * only a probabilistic effect since a new reader will not
             * block if there is a waiting writer behind other enabled
             * readers that have not yet drained from the queue.
             */
        return apparentlyFirstQueuedIsExclusive();
    }
}
```

回顾一下 ReentrantLock中，会直接进行抢占式获取同步状态，在来看看同步器中的apparentlyFirstQueuedIsExclusive 方法

```
/**
 * Returns {@code true} if the apparent first queued thread, if one
 * exists, is waiting in exclusive mode.  If this method returns
 * {@code true}, and the current thread is attempting to acquire in
 * shared mode (that is, this method is invoked from {@link
 * #tryAcquireShared}) then it is guaranteed that the current thread
 * is not the first queued thread.  Used only as a heuristic in
 * ReentrantReadWriteLock.
 */
final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&
    (s = h.next)  != null &&
    !s.isShared()         &&
    s.thread != null;
}
```

只有在 头结点后的第一个结点不是Shared 模式（有独占和共享模式）情况下才会抢占获取同步状态。为什么会这样？ 其实是因为 读锁是共享模式不是ReentrantLock 中的独占式，共享式获取锁的时候，会唤醒后面的同样是共享模式的线程，也就是说在读锁被阻塞后，只要有一个读锁线程获取到了锁，其它读锁线程理论也应该会被唤醒获取到锁。
同样的如果有共享模式下的线程处于阻塞状态，那么当前线程也不应该获取到同步状态，因为此时可能写锁被其它线程获取，导致读锁有线程被阻塞了。

（5）如果获取锁成功了，那么就需要记录读锁线程获取读锁的次数了。如果只有一个线程获取读锁，那么设置或更新firstReader，firstReaderHoldCount，否则需要从线程的threadLocal中获取当前线程获取读锁情况（cachedHoldCounter 缓存的是上次获取读锁的线程）

（6）如果在一些情况下读锁获取失败，那么会执行fullTryAcquireShared，这里面就是不断尝试的过程。

```
/**
 * Full version of acquire for reads, that handles CAS misses
 * and reentrant reads not dealt with in tryAcquireShared.
 */
final int fullTryAcquireShared(Thread current) {
        /*
         * This code is in part redundant with that in
         * tryAcquireShared but is simpler overall by not
         * complicating tryAcquireShared with interactions between
         * retries and lazily reading hold counts.
         */
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0) {
            //如果获取写锁的线程不是当前线程，则获取读锁失败（锁降级）
            if (getExclusiveOwnerThread() != current)
                return -1;

        } else if (readerShouldBlock()) {
            // 写锁空闲  且  线程应当被阻塞
            // 如果是已获取读锁的线程重入读锁时，即使公平策略指示应当阻塞也不会阻塞
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            //从ThreadLocal中删除
                            readHolds.remove();
                    }
                }
                // 需要阻塞且是非重入(还未获取读锁的)，获取失败
                if (rh.count == 0)
                    return -1;
            }
        }
        //读锁达到最大
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            //后面 和 tryAcquireShared中类似
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

主要逻辑：

（1）如果写锁被占据，并且如果获取写锁的线程不是当前获取读锁的线程，则获取读锁失败（锁降级），如果是当前获取读锁的线程，那么可以进行后续获取读锁操作。

（2）如果写锁空闲并且应该被阻塞：1、获取读锁的线程是重入的，那么不会被阻塞，会一直循环等待，知道可以获取锁；2、获取读锁的线程是非重入的（还未获取读锁的），获取失败，被阻塞。

（3）可以进行获取锁操作，和tryAcquireShared 中类似。

这里有一个难点：重入的线程，应该被阻塞，但是不会被阻塞，会一直进行自旋，直到能获取读锁。原因在于：重入的线程已经获取到了锁，现在如果阻塞了，那么读锁没有释放，写锁就不能正常获取，逻辑错误。
有没有人认为可以 把重入的锁完全释放再阻塞，等唤醒时在获取锁就可以了(类似 Condition 中的await() ），利用HoldCounter 可以知道重入的次数，但是最主要的是占据的资源该怎么办，释放肯定是应该，那么在唤醒后是不是又该把资源拿到手呢，这样岂不是又要记录资源情况了，而且资源万一出现问题了怎么办，本来操作资源可以成功的，但是因为中途阻塞导致了失败，这样又该如何，似乎有很多问题，不过思考思考总是好的。


ok,到这里 我们就把tryAcquireShared 分析完了，同时读锁也就分析完了，如果tryAcquireShared 中获取锁失败了，那么会执行同步器的doAcquireShared 方法，这个方法我们在分析同步器中已经分析了，这里就不在重复分析了，主要逻辑就是：符合FIFO规则下 再次获取同步状态，如果获取成功，那么会唤醒后续共享模式的线程，否则判断是否阻塞，如果是，则阻塞。


#### 读锁的释放
读锁的释放通过ReadLock的unlock()方法：
```
/**
 * Attempts to release this lock.
 *
 * <p>If the number of readers is now zero then the lock
 * is made available for write lock attempts.
 */
public void unlock() {
    sync.releaseShared(1);
}
```

内部调用 同步器的 releaseShared 方法
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

同获取锁一样，同步器releaseShared 会调用 tryReleaseShared 方法，这个方法会在我们自定义同步器中按照实际需求进行重写。

```
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    //如果是第一个获取读锁的线程
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        //获取缓存的HoldCounter
        HoldCounter rh = cachedHoldCounter;
        //如果缓存的不是当前线程的HoldCounter，那么从ThreadLocal从获取HoldCounter
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;

        //如果count 已经==1 了，那么本次释放后，就可以移除 HoldCounter了
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        //重入线程在释放锁，需要一步一步释放
        --rh.count;
    }
    //无限循环释放锁，直到cas 成功
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```

代码并不复杂，因此这里就不细说了，再最后 return nextc == 0; 在tryReleaseShared 中如果返回true 则会继续执行同步器的doReleaseShared 方法，doReleaseShared 里面主要涉及到了可能会进行唤醒操作，因为如果没有线程占据读锁，那么也许此时写锁的线程正在被阻塞，因此需要去唤醒。

### 写锁

写锁的分析就简单的多了，因为写锁是独占锁，这个和ReentrantLock 中大同小异。
#### 写锁的获取
写锁的获取通过WriteLock的lock()方法：

```
public void lock() {
  sync.acquire(1);
}
```
同样的里面调用同步器的acquire 方法，acquire 方法会调用我们重写的tryAcquire 方法：

```
protected final boolean tryAcquire(int acquires) {
    /*
     * Walkthrough:
     * 1. If read count nonzero or write count nonzero
     *    and owner is a different thread, fail.
     * 2. If count would saturate, fail. (This can only
     *    happen if count is already nonzero.)
     * 3. Otherwise, this thread is eligible for lock if
     *    it is either a reentrant acquire or
     *    queue policy allows it. If so, update state
     *    and set owner.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    //可能有读锁被获取，或者写锁被占据
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        //读锁被占据，或者当前线程非写锁占据线程（重入）
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        //读锁未被获取，写锁线程重入
        setState(c + acquires);
        return true;
    }
    /**
     在公平锁中：遵循FIFO规则
     在非公平锁中：返回false,允许抢占
     */
    if (writerShouldBlock() ||
            !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

如果tryAcquire执行失败（获取锁失败），那么可能会被加入同步队列，具体可以看同步器中的操作。 

#### 写锁的释放
写锁的获取通过WriteLock的unlock()方法：
```
public void unlock() {
   sync.release(1);
}
```
同样的调用同步器的release 方法，同步器中调用重写的tryRelease 方法：

```
/*
 * Note that tryRelease and tryAcquire can be called by
 * Conditions. So it is possible that their arguments contain
 * both read and write holds that are all released during a
 * condition wait and re-established in tryAcquire.
 */
protected final boolean tryRelease(int releases) {
    //如果释放线程不是独占写锁线程，
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```

tryRelease 过程很简单，就是释放写锁，如果写锁被释放完后（同一个线程可能会重入），可能需要唤醒同步队列中阻塞的读锁或者其它写锁线程，这个见同步器中的代码。

#### 锁降级（参考 Java 并发编程的艺术）
锁降级指的是写锁降级成读锁，如果当前线程拥有写锁，然后将其释放，最后在获取读锁，这种不能称之为锁降级，锁降级是指把持住当前拥有的写锁，再获取到读锁，随后再释放写锁的过程。
锁降级中读锁的获取是否有必要呢，答案是必要的。主要是保证数据的可见性，如果当前线程不获取读锁而是直接释放写锁，假设此刻另一个线程（记作线程B）获取了写锁并修改了数据，那么当前线程无法感知到线程B的数据更新，如果当前线程获取读锁，即遵循锁降级的步骤，则线程B将会阻塞，知道当前线程使用数据并释放读锁之后，线程B才能获取写锁进行数据更新。

```
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    //exclusiveCount(c)计算写锁
    //如果存在写锁，且锁的持有者不是当前线程，直接返回-1
    //锁降级问题
    if (exclusiveCount(c) != 0 && getExclusiveOwnerThread() != current)
        return -1;
```

在读锁的获取中，会进行进行判断写锁是否为0,如果不为0,并且当前获取读锁的线程也是写锁的占有者，那么会继续往后执行，这个过程就是锁降级的过程。

ReentrantReadWriteLock 不支持锁升级（把持读锁，获取写锁）目的也是保证数据可见性，如果读锁已经被多个线程获取，其中任意线程成功获取了写锁并更新了数据，则其更新对其他获取到读锁的线程是不可见的 。