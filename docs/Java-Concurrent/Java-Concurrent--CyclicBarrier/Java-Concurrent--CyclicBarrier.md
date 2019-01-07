###  CyclicBarrier 介绍（jdk 1.8）
CyclicBarrier的字面意思是可循环使用（Cyclic）的屏障（Barrier）,它的功能是让一组线程到达一个屏障（也就叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。

在前面，我们分析过CountDownLatch，CountDownLatch 主要用于一个线程等待其它线程都完成后，再继续执行，而CyclicBarrier  更多的则是屏障作用，在同一个地方，等待其它线程全部执行到这里，然后再放行，当然CyclicBarrier  也完全可以实现CountDownLatch 的功能。CyclicBarrier  还有一个比较特殊的功能：就是当所有线程到达屏障后，可以执行某个任务。

CountDownLatch  的计数器只能使用一次，而CyclicBarrier的计数器可以重置，得以重新利用。

CountDownLatch 内部使用的队列同步器（AQS），CyclicBarrier 中使用的ReentrantLock 来实现同步，而ReentrantLock 是基于AQS 是实现的，因此其它它们的底层实现都是一样的。
在CountDownLatch  对AQS 做了一个简单的回顾，可以参过下面内容：

[Java 并发 --- CountDownLatch源码分析](http://blog.ztgreat.cn/article/29)

更多的可以参考下面内容：

[Java 并发 --- CountDownLatch源码分析](http://blog.ztgreat.cn/article/29)

[Java 并发 ---ReentrantLock源码分析](http://blog.ztgreat.cn/article/14)

[Java 并发 ---AbstractQueuedSynchronizer(同步器)-独占模式](http://blog.ztgreat.cn/article/7)

[Java 并发 ---AbstractQueuedSynchronizer-共享模式与Condition](http://blog.ztgreat.cn/article/9)

###  使用
```
class MyThread extends Thread {
    private CyclicBarrier cb;
    public MyThread(String name, CyclicBarrier cb) {
        super(name);
        this.cb = cb;
    }
    public void run() {
        System.out.println(Thread.currentThread().getName() + " going to await");
        try {
            cb.await();
            System.out.println(Thread.currentThread().getName() + " continue");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

//最后一个线程到屏障后 执行的任务
class Task implements  Runnable{

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + "execute the barrierCommand");
    }
}
public class CyclicBarrierDemo {
    public static void main(String[] args) throws InterruptedException, BrokenBarrierException {
        CyclicBarrier cb = new CyclicBarrier(2, new Task());
        MyThread thread1 = new MyThread("thread1", cb);
        MyThread thread2 = new MyThread("thread2", cb);
        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();
    }
}
```
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171202135320865.png)
当线程执行到屏障时（cb.await()），如果不是最后一个到来的线程，则进行等待，如果是最后一个到来的线程，则执行指定的任务（可选），然后唤醒所有阻塞在该屏障的线程，继续往下执行。

### 数据结构
```
// 可重入锁
private final ReentrantLock lock = new ReentrantLock();
// 条件队列
private final Condition trip = lock.newCondition();
// 参与的线程数量
private final int parties;
// 由最后一个进入 barrier 的线程执行的操作，也就是所有线程到达屏障后执行的任务
private final Runnable barrierCommand;
// 表示当前的屏障（第x代）
private Generation generation = new Generation();
// 正在等待进入屏障的线程数量，初始值是等于parties值的
private int count;

//屏障控制
private static class Generation {
    boolean broken = false; //表示屏障是否被破坏
}
```
CyclicBarrier 使用一个可重入锁来进行同步操作，通过Condition来进行条件阻塞操作。
 Generation 指的是当前的屏障，因为当所有线程到达该屏障时，该屏障完成任务，同时屏障可以被重设，再次利用，当再次利用时，则会重新生成Generation 来指代当前屏障，这个可以结合后面代码来分析。

### 构造方法
1、指定参与的线程数量
```
public CyclicBarrier(int parties) {
    this(parties, null);
}
```
2、指定参与的线程数量和所有线程到底屏障后执行的任务

```
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties; // 初始值为 parties
    this.barrierCommand = barrierAction;
}
```
### wait 方法
1、无参的wait 方法，如果发生中断会抛出中断异常，如果屏障被破坏会抛出屏障破坏异常
```
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```
2、拥有超时功能的await，达到屏障后，进行一定时间的等待，如果在这段时间内，屏障还没有结束（所有线程到达屏障），则会抛出超时异常。
```
public int await(long timeout, TimeUnit unit)
    throws InterruptedException,
           BrokenBarrierException,
           TimeoutException {
    return dowait(true, unit.toNanos(timeout));
}
```
dowait方法为CyclicBarrier类的核心方法，CyclicBarrier类对外提供的await方法在底层都是调用该了方法。

```
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();  //加锁
    try {
        // 当前屏障（代）
        final Generation g = generation;
         // 屏障被破坏，抛出异常
        if (g.broken)
            throw new BrokenBarrierException();

        // 线程被中断
        if (Thread.interrupted()) {
            // 损坏当前屏障，并且唤醒所有的线程
            breakBarrier();
            throw new InterruptedException();
        }
        // 正在等待进入屏障的线程数量-1
        int index = --count;
        if (index == 0) {  //所有线程都已经进入屏障了，该线程是最后一个进入屏障的
            // 执行任务的标识
            boolean ranAction = false;
            try {
                 //需要执行的任务
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run(); //执行任务
                //任务执行完毕    
                ranAction = true;
                //产生下一个屏障
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction) //执行任务异常
                    breakBarrier(); // 损坏当前屏障
            }
        }
		//不是最后一个到达屏障的线程
        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                // 没有设置等待时间
                if (!timed)
                    trip.await();// 则一直进行等待
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos); //进行超时等待
            } catch (InterruptedException ie) {
                //在等待过程中发生中断异常 ，如果屏障没有结束，并且没有被破坏，那么破坏屏障
                if (g == generation && ! g.broken) {
                    breakBarrier(); // 破坏屏障
                    throw ie;  //抛出异常
                } else {
                    //当前屏障已经结束，或者屏障已经被破坏，则重设中断标识位
                    Thread.currentThread().interrupt();
                }
            }
            //如果屏障被破坏，则抛出异常
            if (g.broken)
                throw new BrokenBarrierException();
			//屏障结束
            if (g != generation)
                return index;
            //设置了时间，并且时间超时了，破坏屏障，抛出异常
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock(); //释放锁
    }
}
```
整个逻辑已经还是比较清楚了，还是再进行简单的描述：
当线程来屏障时（dowait），如果屏障已经被破坏，那么直接抛出异常，如果发现发生了中断异常，则表示上层调用有情况，破坏屏障，抛出异常。
如果自己是最后一个到达屏障的线程，那么需要执行指定的任务（如果有的话），如果执行过程中，发生异常，则破坏屏障（相当于通知了其它线程）。否则结束当前屏障，返回。
如果自己不是最后一个到达屏障的线程，如果设置了超时，则进行超时等待，否则进行"无限"等待，如果在等待过程中，发生了中断异常，如果当前屏障没有结束，并且没有被破坏，那么破坏屏障，抛出异常，否则重新设置中断标志位，告诉上层应用，曾经被中断过。
当从等待中醒来时，如果发现屏障被破坏了，那么抛出异常，如果屏障没有被破坏，并且屏障已经换代了，那么说明最后一个线程已经到来了，当前屏障任务结束，返回来到屏障的相对顺序。
如果屏障还没有结束，同时设置了超时等待，并且已经超时，则抛出超时异常，否则，否则再次从循环开始（检查条件，等待）。

从dowait 我们看到，如果当前线程无法进行执行或者等待，那么就会通过破坏屏障的方式来通知其它线程（也表示了本次屏障失败）。

当最后一个线程到来时，应该负责唤醒其它等待的线程，这个操作是放在的nextGeneration 方法中的，再看这个方法之前，我们来看看breakBarrier 这个方法。

#### breakBarrier 方法
```
private void breakBarrier() {
    generation.broken = true; // 设置屏障破坏标识
    count = parties;  //重设count
    trip.signalAll(); //唤醒所有等待线程
}
```
如果当前线程不能继续等待或者执行，那么就会破坏线程，同时也会唤醒阻塞在屏障中的线程，关于唤醒的工作，我们后面再来分析。

#### nextGeneration 方法
```
private void nextGeneration() {
    // signal completion of last generation
    trip.signalAll();  //唤醒阻塞在屏障上的所有线程
    // set up next generation
    count = parties; //重设count
    generation = new Generation(); // 生成新的屏障
}
```
当最后一个线程到底屏障，并且任务执行成功后（如果有任务的话），那么就表示当前屏障执行完毕，就会唤醒其它线程，继续往后执行，同时重新生成新的屏障，以供下次使用。

#### signalAll 方法
这个signalAll 是同步器中的方法，这个我们这里再来撸一遍吧
```
public final void signalAll() {
    if (!isHeldExclusively()) // 当前线程必须处于独占模式（获取排它锁）
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null) //如果条件队列不为空
        doSignalAll(first); //唤醒所有等待线程
}
```
Condition维护着一个队列，凡是不满足该条件的都会放到该队列进行等待。
signalAll方法会判断头结点是否为空，即条件队列是否为空，然后会调用doSignalAll函数
```
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null; // 置空
    //遍历队列，唤醒每一个节点
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
         // 将first结点从condition队列转移到sync队列
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```
doSignalAll 则是遍历条件队列，将每个节点转移到同步队列中，然后执行再执行唤醒工作，下面看看transferForSignal 方法（该方法在：[Java 并发 ---AbstractQueuedSynchronizer-共享模式与Condition](http://blog.ztgreat.cn/article/9) 有分析，因此下面分析的相对比较简略，如果不是很懂，可以参考这篇文章的内容）
```
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0)) //将节点等待状态设置为0
        return false;
    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    Node p = enq(node); // 将节点加入同步队列中
    int ws = p.waitStatus;
    //当前节点通过CAS机制将waitStatus置为SIGNAL
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread); // 唤醒节点线程
    return true;
}
```
看到这里，应该就明白了CyclicBarrier 的大致流程吧，当屏障执行成功，最后一个线程会唤醒所有等待线程，唤醒的线程会加入到同步队列中，如果屏障执行失败，则也会唤醒所有等待的线程。
**这里还有一个细节**，**把等待的线程从Condition队列移到同步队列中后，线程还没有真正被唤醒**，同步队列中的线程会竞争获取同步状态（锁），只有获取到同步状态后，线程才会真正的唤醒，这个部分可以参考同步器文章的内容，这里就不在详细展开了。

这里还有一个细节，当所有线程到底屏障后，最后一个线程执行指定的任务：

```
//需要执行的任务
final Runnable command = barrierCommand;
if (command != null)
    command.run(); //执行任务
```
这里执行的run 方法，意味着默认这样操作是同步执行的，并不是单独建立一个线程来执行任务，当然我们的run 方法中，可以自己建立线程来异步执行该任务。

### 总结

CyclicBarrier 主要用 ReeantrantLock 与 Condition 来控制线程资源的获取。
CyclicBarrier 是可以被重新利用的，当屏障结束后，屏障会重新被初始化（或者手动重置）。
如果指定了任务，这个任务是由最后一个进入屏障的线程来执行的，调用的是run 方法，也就是默认是进行同步调用执行的。
当CyclicBarrier 执行成功时，会唤醒阻塞在屏障中的线程，等待的线程会移到同步队列中，参与同步状态（锁）的竞争。
当CyclicBarrier 执行失败时（中断，超时，指定的任务执行失败），也会唤醒阻塞在屏障的线程，只是这里会抛出异常。

想要了解更多，可以参考下面的内容：

[Java 并发 --- CountDownLatch源码分析](http://blog.ztgreat.cn/article/29)

[Java 并发 ---ReentrantLock源码分析](http://blog.ztgreat.cn/article/14)

[Java 并发 ---AbstractQueuedSynchronizer(同步器)-独占模式](http://blog.ztgreat.cn/article/7)

[Java 并发 ---AbstractQueuedSynchronizer-共享模式与Condition](http://blog.ztgreat.cn/article/9)