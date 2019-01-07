在前面我们分析了AbstractQueuedSynchronizer（同步器），通过内部使用同步器可以实现线程之间的同步，今天我们就来看看基于同步器的锁ReentrantLock是如何实现的。
对同步器还不熟悉的，非常有必要先弄清楚同步器，ReentrantLock是基于AbstractQueuedSynchronizer来实现的，里面关于AbstractQueuedSynchronizer的部分不会再次过多分析，因此在看本文之前，必须明白AbstractQueuedSynchronizer的整体流程。

可以参考我前面写的  [Java 并发 ---AbstractQueuedSynchronizer(同步器)-独占模式](http://blog.ztgreat.cn/article/7)

### 介绍
ReentranLock是可重入锁（synchronized也是），重入是指任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞，实现重入锁那么需要在获取锁时，需要去识别获取锁的线程是否会当前占据锁的线程，如果是，则再次获取成功，在锁的释放时，线程重复n次获取了锁，那么在第n次释放该锁后，其它线程才能够获取到该锁。

ReentranLock 有公平锁和非公平锁（默认）之分，公平性与否是针对开始获取锁而言的，如果一个锁是公平的，那么开始锁的获取顺序就应该符合请求的绝对时间顺序，也就是FIFO。

ReentranLock中的公平锁和非公平锁是ReentranLock的内部类，其内部实现都是基于AbstractQueuedSynchronizer来实现的。

### 使用
```
Lock lock = new ReentrantLock();//Lock lock = new ReentrantLock(true);
Condition condition = lock.newCondition();
    lock.lock();
    try {
    while(条件判断表达式) {
        condition.wait();
    }
    // 处理逻辑
} finally {
    lock.unlock();
}
```

使用方式很简单，在产生ReentrantLock 实例时，通过指定参数（true 或 false）来显示使用公平锁或非公平锁。
对于Condition，在我前面博客中有分析（[Java 并发 ---AbstractQueuedSynchronizer-共享模式与Condition](http://blog.ztgreat.cn/article/9)），如果对这个知识点不熟悉，也不会影响我们对ReentrantLock的分析。

下面展示的是ReentrantLock的内部结构。
```
public class ReentrantLock implements Lock, java.io.Serializable {

    // 依赖内部的同步器
    private final Sync sync;
    abstract static class Sync extends AbstractQueuedSynchronizer {
        abstract void lock();
        protected final boolean tryRelease(int releases) {
            ...
        }
    }

    /**
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync {
        final void lock() {
            ...
        }
        protected final boolean tryAcquire(int acquires) {
            ...
        }
    }
    /**
     * Sync object for fair locks
     */
    static final class FairSync extends Sync {
        final void lock() {
            ...
        }
        protected final boolean tryAcquire(int acquires) {
            ...
        }
    }
}
```

### 公平锁实现

先看看公平锁源码：

```
/**
 * Sync object for fair locks
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

接下来我们来分析一下里面的两个方法：

```
final void lock() {
    acquire(1);
}
```

lock 方法很简单，这个是对外的接口，其内部调用的是AbstractQueuedSynchronizer的acquire方法，这个方法我们这里不在跟入分析（在前面我们已经分析过了），acquire方法里面会调用tryAcquire方法，这个方法就至关重要了。

```
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

在同步器中我们已经知道了 **只有前驱节点是头节点，才能够尝试获取同步状态**，在公平锁中也遵循该规则，因此hasQueuedPredecessors 判断的是当前线程节点是否是头结点的后继节点，只有符合该规则那么才会获取同步状态，同时设置当前现在为获取锁的线程。

```
else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0)
        throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
}
```

对于tryAcquire中的这部分代码，是重入锁的实现，如果获取锁的线程再次请求锁，那么会直接获取锁。

**公平锁如果获取锁失败，那么该线程会被加入到同步队列中，在后续的获取同步状态过程中遵循FIFO规则**，这个可以在同步器（AbstractQueuedSynchronizer）acquireQueued方法中可以得出，这个方法在前面有分析过，这里比较重要就再次贴一下：

AbstractQueuedSynchronizer 中的acquireQueued 方法。
```
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

#### 公平锁小结
公平锁我们看到其实很简单，严格按照FIFO规则获取锁，对于每个线程节点来说都是公平的，先请求锁的线程总是会先得到锁。

    1. 公共锁**初次获取锁**的过程遵循FIFO规则
    2. 公平锁初次获取锁过后，后续过程同样遵循FIFO规则。

### 非公平锁实现

ReentrantLock默认产生的实例是非公平锁
```
/**
 * Sync object for non-fair locks
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

```
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

这里我们主要看一下nonfairTryAcquire 方法，公平锁和非公平锁的差异就体现在这里。

非公平锁在**初次获取锁**的时候，不会要求当前线程在同步队列中并头结点是该线程节点的前继结点，会直接获取同步状态，因此我们看到非公平锁在这里不会遵循FIFO规则，如果同步队列中有其它等待线程，那么对于这些等待线程是不公平的。

如果非公平锁在获取锁失败后，会加入同步队列，对于此后的后续过程获取锁都遵循FIFO规则（同步器中 的 acquireQueued方法），因此后续过程都是公平的，只有在最开始可以"插队",以后就不能"插队"了.

#### 非公平锁小结

    1. 非公共锁**初次获取锁**的过程不遵循FIFO规则（插队）
    2. 非公平锁初次获取锁过后，后续过程同样遵循FIFO规则。

### ReentrantLock总结
ReentrantLock的分析和实现都比较简单，因为ReentrantLock都是基于同步器来实现的，ReentrantLock中的公平锁和非公平锁的差异只是在初次获取锁的过程，这个过程是实现同步器的子类需要重写的方法。

