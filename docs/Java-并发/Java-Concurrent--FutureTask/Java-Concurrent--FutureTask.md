在多线程编程中，创建线程可以直接继承Thread，也可以实现Runnable接口。但是这2种方式都有一个缺陷就是：在执行完任务之后无法获取执行结果。

如果需要获取执行结果，就必须通过共享变量或者使用线程通信的方式来达到效果，这样使用起来就比较麻烦。

而自从Java 1.5开始，就提供了Callable和Future，FutureTask，通过它们可以在任务执行完毕之后得到任务执行结果，今天我们就来看看FutureTask 是如何实现的。

### FutureTask （jdk 1.8）
从名字上我们可以知道，FutureTask 是一个可执行的task,同时也拥有Future 的特性，可以获取任务执行的结果，首先我们先看看如何使用FutureTask,这里会简单的涉及线程池的概念。

### FutureTask  使用
```
public class FutureTaskDemo {

    // 异步任务
    static class Task implements Callable<Integer> {
        @Override
        public Integer call() throws Exception {
            System.out.println(Thread.currentThread().getName());
            return 100;
        }
    }
    public static void main(String[] args) {

        Task task = new Task();

        //使用FutureTask
        FutureTask<Integer> futureTask = new FutureTask<Integer>(task);

        // 创建线程池
        ExecutorService executor = Executors.newCachedThreadPool();
        //异步执行任务
        executor.execute(futureTask);

        //使用Future
        Future<Integer> future = executor.submit(task);
        try {
            // 阻塞,等待异步任务执行完毕,获取异步任务的返回值
            System.out.println("FutureTask result:" + futureTask.get());
            System.out.println("Future result:" + future.get());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180117203106535.png)

在上面我们分别使用了FutureTask 和Future来获取返回值，表面看似乎是不一样，实际上本质都是一样，这里通过submit 提交的Callable 任务，返回的Future 其实是FutureTask，这个可以在 AbstractExecutorService源码中找到。
```
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    // 包装任务
    RunnableFuture<T> ftask = newTaskFor(task);
    // 执行任务
    execute(ftask);
    return ftask;
}
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
    return new FutureTask<T>(runnable, value);
}
```
这个是线程池中的细节，这里就展开了，回到我们的FutureTask 上面来。

###  继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180117203854419.png)

通过继承关系我们知道 FutureTask 实际上就是Runnable和Future的合体。

### 数据结构

```
/**
 * The run state of this task, initially NEW.  The run state
 * transitions to a terminal state only in methods set,
 * setException, and cancel.  During completion, state may take on
 * transient values of COMPLETING (while outcome is being set) or
 * INTERRUPTING (only while interrupting the runner to satisfy a
 * cancel(true)). Transitions from these intermediate to final
 * states use cheaper ordered/lazy writes because values are unique
 * and cannot be further modified.
 *
 * Possible state transitions:  //任务状态转移
 * NEW -> COMPLETING -> NORMAL    //正常完成的流程
 * NEW -> COMPLETING -> EXCEPTIONAL //出现异常的流程
 * NEW -> CANCELLED                 //被取消
 * NEW -> INTERRUPTING -> INTERRUPTED //被中断
 */
private volatile int state;  // 任务状态
private static final int NEW          = 0; //初始化状态
private static final int COMPLETING   = 1; 
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5; //正在进行中断
private static final int INTERRUPTED  = 6; // 中断完成

/** The underlying callable; nulled out after running */
//Callable 任务
private Callable<V> callable;
/** The result to return or exception to throw from get() */
// 任务执行结果
private Object outcome; 
/** The thread running the callable; CASed during run() */
// 执行线程
private volatile Thread runner;
/** Treiber stack of waiting threads */
// 调用get() 阻塞等待的线程节点
private volatile WaitNode waiters;
```
主要包含了任务执行状态，Callable 任务以及任务执行结果，文档注释中对任务状态进行了很详细的描述。

### FutureTask 重要方法
1、get 
```
public V get() throws InterruptedException, ExecutionException
```
获取计算的结果, 若计算没完成, 进行阻塞等待, 直到 计算结束或线程中断

2、带超时功能的get 
```
V get (long timeout, TimeUnit unit) throwsInterruptedException, ExecutionException, TimeoutException;
```
获取计算的结果, 若计算没完成, 进行超时等待, 如果发生超时则抛出TimeoutException 异常。

3、isDone
```
boolean isDone();
```
返回计算是否完成 , 若任务完成则返回true。

4、awaitDone
```
int awaitDone(boolean timed, long nanos) throws InterruptedException
```
超时等待任务完成, 返回值是 future 的state的状态。

5、取消任务 cancel
```
public boolean cancel(boolean mayInterruptIfRunning)
```
当任务处于未启动状态时，该方法将导致此任务永远不会被执行；当任务处于已经启动状态时，cancle(true)将以中断执行此任务线程的方式来尝试停止任务,cancle(false)将不会对正在执行此任务的线程产生影响（让正在执行的任务运行完成）;当任务处于以完成状态时，该方法将返回false.

6、任务执行完成回调
```
protected void done()
```
当任务执行完成后将会调用该方法，在FutureTask 中是空实现，如果有需要可以重写该方法。

### 构造方法
1、指定Callable任务
```
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       //初始化状态
}
```
2、指定Runnable 任务及返回结果
```
/**
 * Creates a {@code FutureTask} that will, upon running, execute the
 * given {@code Runnable}, and arrange that {@code get} will return the
 * given result on successful completion.
 *
 * @param runnable the runnable task
 * @param result the result to return on successful completion. If
 * you don't need a particular result, consider using
 * constructions of the form:
 * {@code Future<?> f = new FutureTask<Void>(runnable, null)}
 */
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

### run  方法
既然FutureTask 实现了Runnable 接口，那么就需要实现其run 方法。
```
public void run() {
    // 判断 state 是否是new, 防止任务重复执行
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                // 调用call方法，返回结果result
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)//如果执行成功，则设置任务运行结果
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING) //如果发生了中断，则等待中断过程完成
            handlePossibleCancellationInterrupt(s);
    }
}
```
整个run 方法不复杂，内部调用callable的call 方法，如果任务执行成功，那么设置运行结果
#### 设置任务运行结果
```
protected void set(V v) {
    // 更新任务状态
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        //设置任务结果
        outcome = v;
        //更新任务状态
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        //收尾工作
        finishCompletion();
    }
}
```
#### 回调及唤醒阻塞线程
当任务还在执行时，调用get 方法会进行阻塞等待，FutureTask 内部维护了一个线程等待的链表，这个我们在分析get 方法的时候再来重点分析。
```
private void finishCompletion() {
    // assert state > COMPLETING;
    //FutureTask 内部维护了一个线程阻塞等待链表（调用get 方法阻塞等待结果）
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    //唤醒线程
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }
    // 执行回调方法 done
    done();

    callable = null;        // to reduce footprint
}
```
### get 方法

```
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    //检查任务状态，如果任务还没有执行完成，则进行等待
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    // 返回结果    
    return report(s);
}
```
get() 方法中涉及到 awaitDone 方法, 将awaitDone的运行结果赋值给state, 最后report方法根据state值进行返回相应的值, 而awaitDone是整个 FutureTask 运行的核心
```
/**
 * Awaits completion or aborts on interrupt or timeout.
 * @param timed true if use timed waits
 * @param nanos time to wait, if timed
 * @return state upon completion
 */
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        //如果当前线程发生中断，则从阻塞等待队列中移除（不等待结果了）
        if (Thread.interrupted()) {
            removeWaiter(q);
            //抛出中断异常
            throw new InterruptedException();
        }

        int s = state;
        //如果任务已经执行完成了，并且相关参数已经设置完毕
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null; //重置
            // 返回状态    
            return s;
        }
        //任务执行完成，但是部分参数还未设置完毕（参考run 方法）
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield(); //让出cpu,等待
        else if (q == null)
            q = new WaitNode(); //创建线程等待节点
        else if (!queued) //如果还没有入队，则进行入队等待
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) { //如果有超时，则进行超时等待
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) { //超时，移除等待线程，返回状态
                removeWaiter(q);
                return state;
            }
            // 超时等待
            LockSupport.parkNanos(this, nanos);
        }
        else
            LockSupport.park(this);
    }
}
```
waitDone就是将当前线程加入等待队列(WaitNode有当前Thread的Thread变量)，然后用LockSupport将自己阻塞，等待超时或者被解除阻塞后，返回状态（可能任务执行完成，也可能没有执行完成），如果发生了中断，则会抛出中断异常。

### cancel 取消任务

```
public boolean cancel(boolean mayInterruptIfRunning) {
    //new 状态包含了任务还没执行和任务正在执行（正在执行：call方法正在运行）
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) { //表示是否中断正在运行的任务
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt(); //中断，并不能实时取消任务，只是设置中断标志位，目标线程需要检测该标志位才知道是否发生了中断
            } finally { // final state
                //更新状态
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        //完成
        finishCompletion();
    }
    return true;
}
```
如果state不是new 那么就退出方法，这时的任务任务是已经完成了 或是被取消了 或是被中断了 
如果state 是new 就设置state 为正在中断 或是取消状态 （new  状态包含了任务正在执行的状态），如果mayInterruptIfRunning 为true 表示允许中断正在运行的任务，则进行中断（**中断，并不能实时取消任务，只是设置中断标志位，目标线程需要检测该标志位才知道是否发生了中断**）。

### report 返回结果
根据任务的状态，返回不同的结果
```
private V report(int s) throws ExecutionException {
    Object x = outcome;
    if (s == NORMAL) //如果任务正常执行完成，则返回结果
        return (V)x;
    if (s >= CANCELLED) //抛出取消任务异常
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}
```
### 总结
FutureTask 内部拥有一个Callable 任务引用，实际的任务是Callable 对象，FutureTask 定义了多种状态，通过对状态的设置和判断可以知道任务的执行情况。

FutureTask 内部维护了一个等待队列（链表），当调用FutureTask 的get 方法时，如果需要进行阻塞等待，则会把请求线程加入到等待队列中，当任务执行完成（正常或者异常），则会唤醒等待队列中的线程，这样等待线程就可以获取任务执行结果了。