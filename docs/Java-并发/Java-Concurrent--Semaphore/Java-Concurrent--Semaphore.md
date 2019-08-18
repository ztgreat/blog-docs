### Semaphore 介绍（jdk 1.8）
Semaphore（信号量）是用来控制同时访问特定资源的线程数量,可以用于做流量控制，特别是公共资源有限的应用场景，如果熟悉操作系统的概念，那么肯定对这么名词不陌生，当初在学习Linux进程通信中，也简单的学习过，今天再次接触到Semaphore。


### 使用
Semaphore作为一种同步工具，使用是非常简单的。
```
public class SemaphoreDemo {

   private static Semaphore semaphore = new Semaphore(2);
   static class Task implements Runnable {
       private String name;
       public Task(String name) {
           this.name = name;
       }
       @Override
       public void run() {
           try {
               semaphore.acquire();
               System.out.println(this.name+" start...");
               Thread.sleep(2000);
           } catch (InterruptedException e) {
               e.printStackTrace();
           } finally {
               System.out.println(this.name+" end...");
               semaphore.release();
           }
       }
   }
   public static void main(String[] args) {
       ExecutorService pool = Executors.newCachedThreadPool();
       pool.submit(new Task("thread1"));
       pool.submit(new Task("thread2"));
       pool.submit(new Task("thread3"));
       pool.submit(new Task("thread4"));
       pool.submit(new Task("thread5"));
       pool.submit(new Task("thread6"));
       pool.submit(new Task("thread7"));
       pool.submit(new Task("thread8"));
       pool.shutdown();
   }
}
```
Semaphore的使用和锁很类似，这是锁一般都是独占的，而Semaphore则是运行一定数量的线程同时访问，程序中我们设置的能同时访问的最大线程数是2,如果将该值设置为1,那么这个时候Semaphore 就退化成了锁了。
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171203144224213.png)

### 数据结构
如果对同步器比较熟悉，那么看到acquire，release 方法已经可以猜测Semaphore 是利用同步器来实现的了，既然用了同步器来实现，那么应该就不难分析了。

```
private final Sync sync;
```
Semaphore 中数据结构很简单，就通过内部类Sync 来实现的，而Sync则是依托同步器来实现的。
```
abstract static class Sync extends AbstractQueuedSynchronizer {
   Sync(int permits) {
       setState(permits);
   }
   ... // 省略其它方法
}
```
在[Java 并发 --- CountDownLatch源码分析](http://blog.ztgreat.cn/article/29)中，我们对同步器做了简单的回顾，这里我们就不在详细阐述同步器的原理了，想要了解更多可以参考下面的内容：

[Java 并发 ---AbstractQueuedSynchronizer(同步器)-独占模式](http://blog.ztgreat.cn/article/7)

[Java 并发 ---AbstractQueuedSynchronizer-共享模式与Condition](http://blog.ztgreat.cn/article/9)

同步器提供了模板方法，我们只需要重写部分方法，就可以实现我们自己的逻辑，因此分析Semaphore 自然而然也落到了这几个方法上。
### 构造方法
1、指定最大访问量
```
public Semaphore(int permits) {
     sync = new NonfairSync(permits);
 }
```
2、指定最大访问量和同步类型
```
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```
通过指定permits 可以设置能同时访问资源的最大线程数，同时可以指定Semaphore 内部使用的同步工具的类型（公平锁或者非公平锁，默认是非公平锁），这个**permits就是同步器的同步状态（也就是信号量）**。
在 ReentrantLock 中就有公平锁和非公平锁之分，在Semaphore 中同样也存在。

### 抽象的Sync
```
abstract static class Sync extends AbstractQueuedSynchronizer {
   
    Sync(int permits) {
        setState(permits); //初始化同步状态
    }
    
    final int getPermits() {
        return getState();
    }
    //非公平的获取同步状态（直接参与同步状态的竞争）
    final int nonfairTryAcquireShared(int acquires) {
        for (;;) {
            //同步状态
            int available = getState();
            //新的同步状态
            int remaining = available - acquires;
			//如果remaining<0 表明超出最大访问数量，失败
			//如果remaining>=0 则cas 设置同步状态，获取同步状态成功。
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
    // 尝试释放同步状态（重写同步器的方法）
    protected final boolean tryReleaseShared(int releases) {
        for (;;) {
            int current = getState();
            // 释放
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next)) // cas 更新同步状态
                return true;
        }
    }
    // 减少信号量
    final void reducePermits(int reductions) {
        for (;;) {
            int current = getState(); //当前同步状态
            int next = current - reductions; //减少
            if (next > current) // underflow
                throw new Error("Permit count underflow");
            if (compareAndSetState(current, next)) //cas 设置同步状态
                return;
        }
    }
    //设置同步状态为0
    final int drainPermits() {
        for (;;) {
            int current = getState();
            if (current == 0 || compareAndSetState(current, 0))
                return current;
        }
    }
}
```
#### 非公平的Sync
```
static final class NonfairSync extends Sync {
    // 初始化信号量（同步状态）
    NonfairSync(int permits) {
        super(permits);
    }
    //重新同步器 尝试获取同步状态 方法
    protected int tryAcquireShared(int acquires) {
        // 调用非公平的获取同步器状态方法
        return nonfairTryAcquireShared(acquires); 
    }
}
```
在同步器中我们已经知道了 只有前驱节点是头节点，才能够尝试获取同步状态，在公平锁中则遵循该规则，在非公平锁中，则不完全遵循该规则，当线程第一次获取同步状态时，不管同步队列中是否有等待线程，照样参与到竞争同步状态中，如果竞争失败，则可能会加入到同步队列中，在后续的获取同步状态过程中遵循FIFO规则，这个可以在同步器（AbstractQueuedSynchronizer）acquireQueued方法中可以得出，这个方法在前面有分析过。
#### 公平的Sync
```
static final class FairSync extends Sync {
    // 初始化 同步状态
    FairSync(int permits) {
        super(permits);
    }
    // 重写同步器 尝试获取同步状态 方法
    protected int tryAcquireShared(int acquires) {
        for (;;) {
            // 判断是否是头结点的后继（也就是同步队列中是否有其它等待线程）
            if (hasQueuedPredecessors())
                return -1;
            // 获取同步状态，判断是否超出可提供量，获取成功则更新
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
```
在同步器中我们已经知道了 只有前驱节点是头节点，才能够尝试获取同步状态，在公平锁中则遵循该规则，因此hasQueuedPredecessors 判断的是当前线程节点是否是头结点的后继节点，只有符合该规则那么才会获取同步状态。

这里的分析都很简单，因为已经很多次接触同步器了，前面也分析过多次这样的代码，因此这里就不再啰嗦了。
### acquire 方法
1、可中断的获取信号量（同步状态）

```
public void acquire() throws InterruptedException {
     //获取一个同步状态（信号量）
     sync.acquireSharedInterruptibly(1);
 }
```
acquireSharedInterruptibly 是同步器中的方法，如果发生中断，会抛出中断异常
```
public final void acquireSharedInterruptibly(int arg)
         throws InterruptedException {
     if (Thread.interrupted())
         throw new InterruptedException();
     // 调用我们重写的尝试获取同步状态（信号量）的方法，如果获取失败，则进行自旋或加入同步队列。
     if (tryAcquireShared(arg) < 0)
         doAcquireSharedInterruptibly(arg);
 }
```
2、 指定获取的信号量个数

```
public void acquire(int permits) throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireSharedInterruptibly(permits);
}
```
在无参acquire中，默认是获取一个信号量，而这个方法，可以指定获取信号量的个数，如果发生中断，则抛出中断异常。
3、不响应中断的acquire
```
public void acquireUninterruptibly(int permits) {
     if (permits < 0) throw new IllegalArgumentException();
     sync.acquireShared(permits);
 }
```
该方法不响应中断，也就是说上层应用没法通过中断的方法取消竞争，就算发生中断，线程同样会参与获取同步状态，获取失败则进行等待，如果发生过中断，会设置中断标志位，这样线程返回时，上层应用可以通过中断标志位来判断是否发生过中断。

### release 方法
####  无参release
```
public void release() {
    sync.releaseShared(1);
}
```
与acquire 相对，release则是释放同步状态（信号量）。releaseShared 是同步器中的方法，该方法会调用我们重写的tryReleaseShared 方法
```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```
如果释放同步状态成功，则会执行doReleaseShared，这个也是同步器中的方法，会进行后续线程的唤醒工作（这个是共享模式的同步器）。

#### 指定释放信号量的数量
```
public void release(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.releaseShared(permits);
}
```
这个acquire 类似，释放指定数量的信号量，本质上和无参的release 是一样的。

### 总结
Semaphore 是一种线程同步工具，使用的同步器来实现，可以控制同时访问特定资源的线程数量。
Semaphore 其实也是相当于一种锁，只是是一种共享锁罢了，我们已经看到了很多基于同步器实现的工具（ReentrantLock，ReentrantReadWriteLock，CountDownLatch），这些都是对同步器的一种应用，因此分析这些工具之前，必须要明白同步器的原理，这样就可以很容易明白这些工具的实现原理了。

[Java 并发 --- CountDownLatch源码分析](http://blog.ztgreat.cn/article/29)

[Java 并发 ---ReentrantLock源码分析](http://blog.ztgreat.cn/article/14)

[Java 并发 ---AbstractQueuedSynchronizer(同步器)-独占模式](http://blog.ztgreat.cn/article/7)

[Java 并发 ---AbstractQueuedSynchronizer-共享模式与Condition](http://blog.ztgreat.cn/article/9)