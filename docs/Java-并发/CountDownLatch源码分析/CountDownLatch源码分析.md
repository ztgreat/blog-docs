CountDownLatch是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程的操作执行完（也可以说到达一个指定的点，并不是指线程运行结束）后再执行，本文分析的是**jdk 1.8** 中的源码。

实现某个线程等待其余线程完成任务的最简单做法是使用join()方法（join 用于当前执行线程等待被join 线程执行结束），其实现原理就是不停的检查被join 线程是否存活，如果被join 线程存活，则继续让当前线程等待。

CountDownLatch 也可以实现这样的功能，今天我们来看看CountDownLatch 是如何实现这类似的功能的。

### 使用
```
import java.util.concurrent.CountDownLatch;

class CountDownLatchThread extends Thread {

    private CountDownLatch countDownLatch;
    public CountDownLatchThread(String name, CountDownLatch countDownLatch) {
        super(name);
        this.countDownLatch = countDownLatch;
    }
    public void run() {
        System.out.println(Thread.currentThread().getName() + " start doing something");
        try {
            // 休眠2秒
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + " finish");
        //任务结束,调用 countDown 方法
        countDownLatch.countDown();
    }
}
public class CountDownLatchDemo {

    public static void main(String[] args) {

        CountDownLatch countDownLatch = new CountDownLatch(2);
        CountDownLatchThread c1 = new CountDownLatchThread("c1", countDownLatch);
        CountDownLatchThread c2 = new CountDownLatchThread("c2", countDownLatch);
        //c1 开始
        c1.start();
        //c2 开始
        c2.start();
        System.out.println("Waiting for c1 thread and c2 thread to finish");
        try {
            // main 线程等待c1,c2 结束
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + " continue");
    }
}
```
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171127211831256.png)

CountDownLatch 的使用非常简单，通过构造函数指定需要等待的线程数量，等待者调用CountDownLatch  的await() 方法，被等待者调用CountDownLatch 的countDown 方法（都是针对的同一个CountDownLatch  实例哟）。
知道了如何使用，接下来我们看看CountDownLatch 是如何完成这 “**等待--通知**"机制。

### 数据结构
```
public class CountDownLatch {
    //继承同步器
    private static final class Sync extends AbstractQueuedSynchronizer {
        Sync(int count) {
            setState(count);
        }
        int getCount() {
            return getState();
        }
        //重写同步器中的模板方法
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
        //重写同步器中的模板方法
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;
    ...
 }
```
CountDownLatch，其底层是通过队列同步器（AQS）来实现的，一起来回顾一下AQS:
队列同步器（AQS）是一种用于线程同步的一个组件，其内部通过cas操作来实现原子操作，AQS 有两种队列和两种模式

**两种队列**：

同步队列和条件队列，不同的条件会有不同的条件队列。

**两种模式**：

独占模式和共享模式，独占模式 也就是一个线程在使用的时候，其它线程只能等待，共享模式就是可以设置多个线程同时访问。

**同步状态**：

AQS 中维护着一个同步状态（实际就是一个整型变量），独占模式下只有两种状态（占有和空闲），共享模式下可以表示为允许多少个线程同时访问。

**在独占模式中**：如果线程能获取到同步状态（设置同步状态为占有状态），那么就相对于获取到了锁，其它线程再获取同步状态的时候就会失败，这个时候其它线程会进行一定时间的自旋，如果期间仍然没有获取到同步状态，那么就只能放到同步队列中，让其线程阻塞等待。当获取到同步状态的线程执行完后，释放同步状态，同时唤醒同步状态队头的等待线程。

**在共享模式中**：其实和独占模式大同小异，设置同步状态为一个数值，表示最多允许好多个线程同时访问，当一个线程获取到同步状态后，就将同步状态递减，后续线程也是如此，如果当一个线程来获取同步状态时，发现无法获取了，那么就进行自旋，然后再放到同步队列中，只是这个时候节点的是共享模式，而不是上面的独占模式，独占模式某个节点被唤醒之后，它只需要将这个节点设置成head就完事了，而共享模式不一样，某个节点被设置为head之后，还可能需要唤醒后面的节点， 如果它的后继节点是SHARED状态的，那么将继续通过doReleaseShared方法尝试往后唤醒节点，实现了共享状态的向后传播。

AQS这里就简单回顾了一下，想要了解更多可以参考我前面写的内容:

[Java 并发 —AbstractQueuedSynchronizer(同步器)-独占模式](http://blog.ztgreat.cn/article/7)

[Java 并发 ---AbstractQueuedSynchronizer-共享模式与Condition](http://blog.ztgreat.cn/article/9)

###  构造方法
```
public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
```
CountDownLatch 在实例化的时候需要指定count,而这个count的数值就作为了同步器中的同步状态值。
```
Sync(int count) {
    //调用同步器中方法，设置同步器状态的值
     setState(count);
 }
```

### await 方法
该方法会使当前线程在同步状态计数至零之前一直等待，除非线程被中断。
```
public void await() throws InterruptedException {
     sync.acquireSharedInterruptibly(1);
 }
```
这个acquireSharedInterruptibly 是同步器中的方法，我们在来看一遍：

```
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```
如果发生中断，则抛出中断异常，否则调用tryAcquireShared，如果返回小于0，则后面就是自旋和加入阻塞队列操作了，这个里面是同步器里面的内容，tryAcquireShared 是我们需要重写的方法，通过我们来决定再不满足何条件时才放入到同步队列，来看看CountDownLatch 中Sync 的tryAcquireShared 方法。

```
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```
这个很简单，如果同步状态为0，则返回1，否则返回负数，这样也就是说，如果同步状态不为0，那么就可能会把请求await()方法的线程加入到同步队列（这里用的是可能，因此在同步队列器中，在加入到同步队列之前，会再次尝试获取状态（执行tryAcquireShared），因此这个时候可能会满足，而不加入到同步队列中）。

CountDownLatch调用await  的含义是什么：就是等待其它线程完成操作，而其它线程完成操作会设置同步状态的值，这样如果被等待的线程都完成了，那么同步状态的值就为0了，也就完成了等待其它线程完成操作这一任务了。

除此之外，await 还有两一个超时方法：
### await(long timeout, TimeUnit unit) 方法
```
public boolean await(long timeout, TimeUnit unit)
     throws InterruptedException {
     return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
 }
```
这个其实就是调用的同步器中的超时方法，如果在一定时间内未获取到同步状态（执行tryAcquireShared 成功），那么就返回false,而不是一直阻塞等待。

前面说了，被等待的线程可以更新同步状态，那么其它线程如何更新同步状态呢，这个我们继续往下看。

### countDown 方法
```
public void countDown() {
     sync.releaseShared(1); // 释放同步状态
 }
```
releaseShared也是同步器中的方法：
```
public final boolean releaseShared(int arg) {
     if (tryReleaseShared(arg)) {
         doReleaseShared();
         return true;
     }
     return false;
 }
```
同样会调用我们重写的tryReleaseShared方法，如果该方法返回true,那么就会唤醒同步队列中共享模式的的线程。
```
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState(); //得到同步状态
        if (c == 0)
            return false; // 如果同步状态已经为0，返回false
        int nextc = c-1; //递减
        if (compareAndSetState(c, nextc)) //cas 设置同步状态
            return nextc == 0; //如果同步状态为0了，则需要承担唤醒任务
    }
}
```
当被等待的线程 完成任务，调用countDown  方法时，会将同步状态递减，如果同步状态还未到0，那么说明还有其他线程获取了同步状态，那么就不用进行唤醒工作，如果更新后同步状态为0了，说明所有线程都完成任务了，需要唤醒等待者了。如果调用该方法同步状态已经为0了，那么严格来说是有问题的，同步状态已经不需要释放了，返回false.

在CountDownLatch 中可以等待多个线程，被等待的线程 相当于持有同步状态，当自身完成任务时，则释放同步状态, 等待者在同步状态被释放完之前，一直等待，当同步状态释放完后，会唤醒等待者，通过上面利用队列同步器，通过简单的设置同步器状态就能完成这一操作。同步状态不是简单的占有和非占有两种状态，因此CountDownLatch 是一种共享模式。这一点现在通过我们的分析就可以得出，不仅仅只是通过其源代码得出。

### 总结
经过分析CountDownLatch的源码可知，其底层结构是AQS，对其线程所封装的结点是采用共享模式，AQS 使用的是模板设计方法，只需要我们重写几个简单的方法，就可以实现我们自己的同步组件了。

[Java 并发 —AbstractQueuedSynchronizer(同步器)-独占模式](http://blog.ztgreat.cn/article/7)

[Java 并发 ---AbstractQueuedSynchronizer-共享模式与Condition](http://blog.ztgreat.cn/article/9)
