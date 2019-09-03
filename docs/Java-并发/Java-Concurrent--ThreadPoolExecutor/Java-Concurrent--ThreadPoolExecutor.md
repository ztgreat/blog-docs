在多线程编程中，或多或少都听过或者使用过线程池，合理利用线程池能够带来三个好处。

1、降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。

2、提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。

3、提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

但是要做到合理的利用线程池，必须对其原理比较了解，今天我们就来简要的分析线程池的实现原理---源码面前无秘密
### 线程池的基础架构(jdk 1.8)
为了更好的理解线程池，还是先来梳理一下它的架构，有助于我们的分析，不然很混乱。
#### 继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180118201110795.png)
#### Executor
Executor，任务的执行器，线程池框架中几乎所有类都直接或者间接实现Executor接口，它是线程池框架的基础。它将任务的提交与任务的执行分离开来（生产者--消费者模式），它仅提供了一个Execute()方法用来执行已经提交的Runnable任务。
```
/**
 * Executes the given command at some time in the future.  The command
 * may execute in a new thread, in a pooled thread, or in the calling
 * thread, at the discretion of the {@code Executor} implementation.
 *
 * @param command the runnable task
 * @throws RejectedExecutionException if this task cannot be
 * accepted for execution
 * @throws NullPointerException if command is null
 */
void execute(Runnable command);
```
#### ExcutorService
ExcutorService也是接口，继承Executor接口，通过名称我们知道它提供的是一种服务，ExecutorService提供了 将任务提交给执行者的接口(submit方法)以及 让执行者执行任务(invokeAll, invokeAny方法) 的接口等等。

```
public interface ExecutorService extends Executor {

    /**
     * 关闭线程池，已经提交的任务会得到执行，但不接受新任务
     * 该方法不会等待已经提交的任务执行完成
     */
    void shutdown();

    /**
     * 试图停止所有正在执行的活动任务，暂停处理正在等待的任务
     * 返回等待执行的任务列表
     * 该方法不会等待正在执行的任务终止。
     */
    List<Runnable> shutdownNow();

    /**
     * 如果执行器已关闭，则返回 true。
     */
    boolean isShutdown();

    /**
     * 如果关闭后所有任务都已完成，则返回 true
     * 注意：只有shutdown或者shutdownNow先被调用，isTerminated才可能返回true
     */
    boolean isTerminated();

    /**
     * shutdown request、or the timeout occurs, or the current thread is
     * interrupted，都将导致阻塞，直到所有任务执行完成
     */
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
     * 提交任务用于执行，返回Future,可以通过Future 获得任务执行结果
     */
    <T> Future<T> submit(Callable<T> task);

    /**
     * 提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future
     */
    <T> Future<T> submit(Runnable task, T result);

    /**
     * 提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future
     */
    Future<?> submit(Runnable task);

    /**
     * 执行给定的任务，当所有任务完成时，返回保持任务状态和结果的 Future 列表
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    /**
     * 执行给定的任务，当所有任务完成或超时期满时（无论哪个首先发生），返回保持任务状态和结果的 Future 列表
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
     * 执行给定的任务，如果某个任务已成功完成（也就是未抛出异常），则返回其结果
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    /**
     * 执行给定的任务，如果在给定的超时期满前某个任务已成功完成（也就是未抛出异常），则返回其结果
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
源码上有很全的注释，可以很好的帮助理解。

#### AbstractExecutorService
AbstractExecutorService 抽象类，实现ExecutorService接口，提供了默认实现，下面看看几个我们常使用到的方法。

```
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
    return new FutureTask<T>(runnable, value);
}
```
将Callable 对象包装成FutureTask 对象，在前一篇文章中，我们分析了FutureTask，因此这里就不在分析了，可以参考：[Java 多线程 --- FutureTask 源码分析](http://blog.ztgreat.cn/article/38)

```
// 提交Runnable 任务
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    //执行，execute方法交给具体的子类来实现
    execute(ftask);
    return ftask;
}

// 提交Runnable 任务，并指定结果
public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}

// 提交Callable任务
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

大致过程就是根据提交的任务封装成FutureTask 对象，然后再执行任务，execute 方法交由具体的子类来实现。

#### ThreadPoolExecutor
这个我们稍后将会详细介绍
#### ScheduledThreadPoolExecutor
ScheduledThreadPoolExecutor继承ThreadPoolExecutor来重用线程池的功能

ScheduledThreadPoolExecutor用来支持周期性任务的调度，这个后面我们再分析。


### ThreadPoolExecutor
Java的线程池支持主要通过ThreadPoolExecutor来实现，我们使用的ExecutorService的各种线程池策略都是基于ThreadPoolExecutor来实现的，所以ThreadPoolExecutor十分重要，要弄明白各种线程池策略，必须先弄明白ThreadPoolExecutor。
#### 数据结构
```
/**
 * 整个线程池的控制状态，包含了两个属性：有效线程的数量、线程池的状态（runState）。
 *  workerCount,有效线程的数量(低29位)
 *  runState,   线程池的状态（高3位）
 *  ctl 包含32位数据，低29位存线程数，高3位存runState,runState有5个值：
 *  RUNNING:  接受新任务，处理任务队列中的任务
 *  SHUTDOWN: 不接受新任务，处理任务队列中的任务
 *  STOP:    不接受新任务，不处理任务队列中的任务
 *  TIDYING:  所有任务完成，线程数为0，然后执行terminated()
 *  TERMINATED: terminated() 已经完成
 * 状态转换:
 *  RUNNING -> SHUTDOWN ：调用shutdown()方法
 * (RUNNING or SHUTDOWN) -> STOP ：调用了shutdownNow()方法
 *  SHUTDOWN -> TIDYING ：队列里面没有任务了，同时线程池没有线程了
 *  STOP -> TIDYING：线程池没有线程了
 *  TIDYING -> TERMINATED ：terminated()方法调用完成
  */
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
 // 29位的偏移量
 private static final int COUNT_BITS = Integer.SIZE - 3;
 // 最大容量（2^29 - 1）
 private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
 // 阻塞队列（任务队列）
 private final BlockingQueue<Runnable> workQueue;
 // 可重入锁
 private final ReentrantLock mainLock = new ReentrantLock();
 // 工作线程集合
 private final HashSet<Worker> workers = new HashSet<Worker>();
 // 终止条件
 private final Condition termination = mainLock.newCondition();
 // 线程池最大容量
 private int largestPoolSize;
 // 已完成的任务数量
 private long completedTaskCount;
 // 线程工厂（产生线程）
 private volatile ThreadFactory threadFactory;
 // 线程池拒绝任务策略
 private volatile RejectedExecutionHandler handler;
 // 线程池空闲时，线程存活的时间
 private volatile long keepAliveTime;
 // 是否运行核心线程超时
 private volatile boolean allowCoreThreadTimeOut;
 // 核心线程池大小
 private volatile int corePoolSize;
 // 最大线程池大小
 private volatile int maximumPoolSize;
 //获取线程池状态
 private static int runStateOf(int c)     { return c & ~CAPACITY; }
 //获取工作线程数
 private static int workerCountOf(int c)  { return c & CAPACITY; }
 //获取ctl
 private static int ctlOf(int rs, int wc) { return rs | wc; }
```
#### ctl 属性
状态变量ctl：包含了两个属性：有效线程的数量（workerCount：低29位）、线程池的状态（runState：高3位）。
runState有5个值：

1、RUNNING:  接受新任务，处理任务队列中的任务

2、SHUTDOWN: 不接受新任务，处理任务队列中的任务

3、STOP:    不接受新任务，不处理任务队列中的任务

4、TIDYING:  所有任务完成，线程数为0，然后执行terminated()

5、TERMINATED: terminated() 已经完成

状态转换:

 RUNNING -> SHUTDOWN ：调用shutdown()方法

 (RUNNING or SHUTDOWN) -> STOP ：调用了shutdownNow()方法

 SHUTDOWN -> TIDYING ：队列里面没有任务了，同时线程池没有线程了

 STOP -> TIDYING：线程池没有线程了

 TIDYING -> TERMINATED ：terminated()方法调用完成

#### corePoolSize
线程池中核心线程的数量。当提交一个任务时，线程池会新建一个线程来执行任务，直到当前线程数等于corePoolSize。如果调用了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有基本线程。

#### maximumPoolSize
线程池中允许的最大线程数。线程池的阻塞队列满了之后，如果还有任务提交，如果当前的线程数小于maximumPoolSize，则会新建线程来执行任务。注意，如果使用的是无界队列，该参数也就没有什么效果了。

#### keepAliveTime
线程空闲的时间。线程的创建和销毁是需要代价的。线程执行完任务后不会立即销毁，而是继续存活一段时间：keepAliveTime。默认情况下，该参数只有在线程数大于corePoolSize时才会生效。

#### workQueue
用来保存等待执行的任务的阻塞队列，等待的任务必须实现Runnable接口。

#### RejectedExecutionHandler
RejectedExecutionHandler，线程池的拒绝策略。所谓拒绝策略，是指将任务添加到线程池中时，线程池拒绝该任务所采取的相应策略。当向线程池中提交任务时，如果此时线程池中的线程已经饱和了，而且阻塞队列也已经满了，则线程池会选择一种拒绝策略来处理该任务。

线程池提供了四种拒绝策略：

1、AbortPolicy：直接抛出异常，默认策略；

2、CallerRunsPolicy：用调用者所在的线程来执行任务；

3、DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；

4、DiscardPolicy：直接丢弃任务；

当然我们也可以实现自己的拒绝策略，实现RejectedExecutionHandler接口即可。

#### 线程池处理流程
线程池处理流程如下（来自《Java并发编程的艺术》）：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180120172606803.png)

ThreadPoolExecutor 执行execute 方法分下面4种

1、如果当前运行的线程小于corePoolSize,则创建新线程执行任务（任务无需入阻塞队列，即使此时线程池中存在空闲线程。）。

2、如果当运行的线程大于等于CorePoolSize，那么将任务加入到BlockingQueue,等待线程池中任务调度执行 。

3、如果不能加入BlockingQueue（队列已满），如果线程数小于MaxPoolSize，则创建线程执行任务。

4、如果线程数大于等于MaxPoolSize，那么执行拒绝策略。

5、当线程池中线程数超过corePoolSize，且超过这部分的空闲时间达到keepAliveTime时，回收这些线程。

6、当设置 allowCoreThreadTimeOut(true)时，线程池中corePoolSize范围内的线程空闲时间达到keepAliveTime也将回收。

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180120181611858.png)

#### 构造方法
线程池的创建可以通过ThreadPoolExecutor的构造方法实现：
```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
参数的各个含义在上面已经分析过了。

#### 向线程池提交任务
可以使用两个方法向线程池提交任务，execute()和submit()方法。

execute()方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功。

submit()方法用于提交需要返回值的任务。线程池会返回一个future类型的对象，通过这个future对象可以判断任务是否执行成功，并且可以通过future的get()来获取返回值，get()方法会阻塞当前线程知道任务完成，而使用get(long timeout,TimeUnit unit)方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务还没有执行完。
#### submit
submit 方法我们在AbstractExecutorService 中已经分析了，可以参考前面AbstractExecutorService的分析。

#### execute

```
/**
 1. Executes the given task sometime in the future.  The task
 2. may execute in a new thread or in an existing pooled thread.
 3. If the task cannot be submitted for execution, either because this
 4. executor has been shutdown or because its capacity has been reached,
 5. the task is handled by the current {@code RejectedExecutionHandler}.
 */
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     * 1. 如果运行的线程小于corePoolSize,则尝试创建一个新的线程来执行任务，
     * 调用addWorker函数会原子性的检查runState和workCount，保证正确的添加线程
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     * 2. 如果一个任务能够成功入队列，在添加一个线程时仍需要进行双重检查（因为在前一次检查后该线程死亡了），
     * 或者当进入到此方法时，线程池已经shutdown了，所以需要再次检查状态，若有必要，当停止时还需要回滚入队列操作，
     * 或者当线程池没有线程时需要创建一个新线程
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     *3. 如果无法入队列，那么会尝试增加一个新线程，如果此操作失败，
     * 那么就意味着线程池已经shutdown或者已经饱和了，所以拒绝任务
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
     
    //获取状态变量
    int c = ctl.get();
    //通过workerCountOf方法获取当前有效线程数，如果小于corePoolSize，则尝试创建线程来执行任务
    if (workerCountOf(c) < corePoolSize) {
        //创建工作线程来执行任务
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    //线程池处于RUNNING状态则尝试把任务加入任务队列
    if (isRunning(c) && workQueue.offer(command)) {
        //再次检查
        int recheck = ctl.get();
        //线程池不处于RUNNING状态，则将任务从任务队列中移除
        if (! isRunning(recheck) && remove(command))
            reject(command); // 执行拒绝策略
        else if (workerCountOf(recheck) == 0)
            // 线程池处于SHUTDOWN状态，没有活动线程了，但是队列里还有任务没执行这种特殊情况，可能需要添加工作线程来执行任务
            addWorker(null, false);
    }
    else if (!addWorker(command, false))  //则尝试创建新线程来执行任务
        reject(command); // 失败，执行拒绝策略（有效线程数>=maximumPoolSize）
}
```
执行流程大致如下：

1、如果线程池有效线程数小于corePoolSize，则调用addWorker创建新线程执行任务，成功返回true，失败则执行步骤2。

2、如果线程池处于RUNNING状态，则尝试将任务加入队列队列，如果加入队列成功，则尝试进行Double Check，如果加入失败，则执行步骤3。

3、如果线程池不是RUNNING状态或者加入队列失败，则尝试创建新线程直到有效线程达到maxPoolSize，如果失败，则调用reject()方法执行拒绝策略。

Double Check 主要目的是判断加入到队列队列中的线程是否可以被执行。如果线程池不是RUNNING状态，则调用remove()方法从阻塞队列中删除该任务，然后调用reject()方法处理任务。

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180123111903187.png)


#### Worker
在前面，我们多次提到了工作线程，现在我们来揭开它的面纱，Worker是真正的任务，是由任务执行线程完成。每个Worker对象中，包含一个需要立即执行的新任务和已经执行完成的任务数量，Worker本身，是一个Runnable对象，继承了AbstractQueuedSynchronizer，因此Worker本身也是一个同步组件。

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180122171351380.png)



**问题：worker 为什么需要需要做成一个同步组件呢？**

##### 属性
```
//worker持有的线程
final Thread thread;
/** Initial task to run.  Possibly null. */
//具体的任务
Runnable firstTask;
/** Per-thread task counter */
volatile long completedTasks;
Worker(Runnable firstTask) {
    // 初始化同步状态，在这种状态下，无法中断，见interruptIfStarted() 方法
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    //创建线程，把当前worker 传入 当 任务
    this.thread = getThreadFactory().newThread(this);
}
/** Delegates main run loop to outer runWorker  */
public void run() {
    //委托给runWorker，当执行 thread.start()的时候，会调用到这里
    runWorker(this);
}
void interruptIfStarted() {
    Thread t;
    if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
        try {
            t.interrupt();
        } catch (SecurityException ignore) {
        }
    }
}
...// 省略部分方法
```
它内部封装一个Thread对象，用此对象执行本身的run方法，而这个Thread对象则由ThreadPoolExecutor提供的ThreadFactory对象创建新的线程。（将Worker和Thread分离的好处是，如果我们的业务代码，需要对于线程池中的线程，赋予优先级、线程名称、线程执行策略等其他控制时，可以实现自己的ThreadFactory进行扩展，无需继承或改写ThreadPoolExecutor）

#### addWorker

```
private boolean addWorker(Runnable firstTask, boolean core) {
     retry:
     for (;;) {
         int c = ctl.get();
         //线程池状态
         int rs = runStateOf(c);

         /**
          * 如果线程池为SHUTDOWN,则需要判断任务队列中是否还有任务，如果有任务，需要执行
          */
         if (rs >= SHUTDOWN &&
             ! (rs == SHUTDOWN &&
                firstTask == null &&
                ! workQueue.isEmpty()))
             return false;

         for (;;) {
             //有效工作线程数
             int wc = workerCountOf(c);
             //worker数量大于等于最大容量
             if (wc >= CAPACITY ||
                 // worker数量大于等于核心线程池大小或者最大线程池大小
                 wc >= (core ? corePoolSize : maximumPoolSize))
                 return false;
             //递增worker的数量
             if (compareAndIncrementWorkerCount(c))
                 break retry;
             //状态变量    
             c = ctl.get();  // Re-read ctl
             // 此次的状态与上次获取的状态不相同
             if (runStateOf(c) != rs)
                 continue retry;
             // else CAS failed due to workerCount change; retry inner loop
         }
     }

     boolean workerStarted = false;
     boolean workerAdded = false;
     Worker w = null;
     try {
         //把任务封装在Worker中
         w = new Worker(firstTask);
         final Thread t = w.thread;
         if (t != null) {
             final ReentrantLock mainLock = this.mainLock;
             //加锁
             mainLock.lock();
             try {
                 // Recheck while holding lock.
                 // Back out on ThreadFactory failure or if
                 // shut down before lock acquired.
                 //线程池状态
                 int rs = runStateOf(ctl.get());
                 // 如果线程池处于RUNNING状态执行添加任务操作,或线程池处于SHUTDOWN 状态，firstTask 为空（任务队列不为空，需要添加工作线程来执行任务）
                 if (rs < SHUTDOWN ||
                     (rs == SHUTDOWN && firstTask == null)) {
                     if (t.isAlive()) // precheck that t is startable
                         throw new IllegalThreadStateException();
                     // 将worker添加到worker集合    
                     workers.add(w);
                     int s = workers.size();
                     if (s > largestPoolSize)
                         largestPoolSize = s;
                     workerAdded = true;
                 }
             } finally {
                 //释放锁
                 mainLock.unlock();
             }
             if (workerAdded) {
                 //启动线程，执行任务
                 t.start();
                 workerStarted = true;
             }
         }
     } finally {
         //工作线程创建失败
         if (! workerStarted)
             addWorkerFailed(w);
     }
     return workerStarted;
 }
```
addWorker大致流程如下：

1、判断线程池状态，如果需要创建工作线程，则原子性的增加workerCount。 

2、将任务封装成为一个worker，并将此worker添加进workers集合中。

3、启动worker对应的线程，并启动该线程，运行worker的run方法。

4、如果失败了回滚worker的创建动作（addWorkerFailed 方法），即将worker从workers集合中删除，并原子性的减少workerCount。

#### runWorker 

```
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    //释放锁（设置state为0，允许中断）
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //任务不为null或者阻塞队列还存在任务
        while (task != null || (task = getTask()) != null) {
            //获取锁
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                //中断wt线程（当前线程）
                wt.interrupt();
            try {
                //beforeExecute方法 没有具体实现,可以根据需要重写这个方法
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //afterExecute方法 没有具体实现,可以根据需要重写这个方法
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                //增加worker完成的任务数量
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        //工作线程退出
        processWorkerExit(w, completedAbruptly);
    }
}
```
此方法会调用用户重写的run方法，并且当给定任务完成后，会继续从阻塞队列中取任务，直到阻塞队列为空（即任务全部完成）。
#### getTask
从阻塞队列中获取任务。
```
/**
 * Performs blocking or timed wait for a task, depending on
 * current configuration settings, or returns null if this worker
 * must exit because of any of:
 * 1. There are more than maximumPoolSize workers (due to
 *    a call to setMaximumPoolSize).
 * 2. The pool is stopped.
 * 3. The pool is shutdown and the queue is empty.
 * 4. This worker timed out waiting for a task, and timed-out
 *    workers are subject to termination (that is,
 *    {@code allowCoreThreadTimeOut || workerCount > corePoolSize})
 *    both before and after the timed wait, and if the queue is
 *    non-empty, this worker is not the last thread in the pool.
 *
 * @return task, or null if the worker must exit, in which case
 *         workerCount is decremented
 */
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        /*
         *当线程池处于STOP及以上状态时，释放该线程。
         *当线程处于SHUTDOWN 状态时，并且workQueue请求队列为空，释放该线程。
         */
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
        //获取当前工作线程数
        int wc = workerCountOf(c);

        // Are workers subject to culling?
        /*
         *如果调用allowCoreThreadTimeOut方法设置为true，则所有线程都有超时时间。
         *如果当前线程数大于核心线程数则该线程有超时时间。
         */
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            //减少工作线程
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            //从阻塞队列中取任务，根据情况选择一直阻塞等待还是超时等待
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) { //不响应中断
            timedOut = false;
        }
    }
}
```
getTask 方法从workerQueue阻塞队列中获取Runnable对象，具有超时等待（poll）和无限等待（take）功能。在该函数中还会响应shutDown和、shutDownNow函数的操作，若检测到线程池处于SHUTDOWN或STOP状态，则会返回null。

#### 关闭线程池
可以通过调用线程池的shutdown或shutdownNow方法来关闭线程池，它们的原理是遍历线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法终止。但是它们存在一定的区别，shutdownNow首先将线程池的状态设置成STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行的任务列表，而shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程。

只要调用了这两个关闭方法中的任意一个，isShutdown方法就会返回true。当所有的任务都已关闭后，才表示线程池关闭成功，这时调用isTerminaed方法才会返回true,至于应该调用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定，通常调用shutdown方法来关闭线程池，如果任务不一定要执行完成，则可以调用shutdownNow方法。

#### shutdown
```
 /**
  * Initiates an orderly shutdown in which previously submitted
  * tasks are executed, but no new tasks will be accepted.
  * Invocation has no additional effect if already shut down.
  *
  * <p>This method does not wait for previously submitted tasks to
  * complete execution.  Use {@link #awaitTermination awaitTermination}
  * to do that.
  */
 public void shutdown() {
     final ReentrantLock mainLock = this.mainLock;
     mainLock.lock();
     try {
         checkShutdownAccess();
         //设置线程池状态为SHUTDOWN（不接受新任务，处理任务队列中的任务）
         advanceRunState(SHUTDOWN);
         //中断所有的空闲线程
         interruptIdleWorkers();
         onShutdown(); // hook for ScheduledThreadPoolExecutor
     } finally {
         mainLock.unlock();
     }
     // 尝试关闭线程池
     tryTerminate();
 }
```
interruptIdleWorkers 中断所有空闲线程，参数onlyOne如果为true，则表示仅中断一个线程。
```
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //遍历工作线程
        for (Worker w : workers) {
            Thread t = w.thread;
             // w.tryLock()对Worker加锁，这保证了正在运行执行Task的Worker不会被中断
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt(); //中断
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

#### tryTerminate
```
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        // 线程池处于Running状态
        // 线程池已经终止了
        // 线程池处于ShutDown状态，但是阻塞队列不为空
        if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;

        
        // 如果线程池中还存在线程，则会尝试中断线程
        if (workerCountOf(c) != 0) {
            // /线程池还有线程，但是队列没有任务了，需要中断等待任务的线程
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 尝试终止线程池
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    terminated();
                } finally {
                    // 线程池状态转为TERMINATED
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
    }
}
```

#### shutdownNow

```
/**
 * Attempts to stop all actively executing tasks, halts the
 * processing of waiting tasks, and returns a list of the tasks
 * that were awaiting execution. These tasks are drained (removed)
 * from the task queue upon return from this method.
 *
 * <p>This method does not wait for actively executing tasks to
 * terminate.  Use {@link #awaitTermination awaitTermination} to
 * do that.
 *
 * <p>There are no guarantees beyond best-effort attempts to stop
 * processing actively executing tasks.  This implementation
 * cancels tasks via {@link Thread#interrupt}, so any task that
 * fails to respond to interrupts may never terminate.
 */
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        //设置线程池状态为STOP（不接受新任务，不处理任务队列中的任务）
        advanceRunState(STOP);
        //中断所有工作线程
        interruptWorkers();
        //返回等待执行的任务列表
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```
调用drainQueue()方法返回等待执行到任务列表。
```
/**
  * Drains the task queue into a new list, normally using
  * drainTo. But if the queue is a DelayQueue or any other kind of
  * queue for which poll or drainTo may fail to remove some
  * elements, it deletes them one by one.
  */
 private List<Runnable> drainQueue() {
     BlockingQueue<Runnable> q = workQueue;
     List<Runnable> taskList = new ArrayList<Runnable>();
     q.drainTo(taskList);
     if (!q.isEmpty()) {
         for (Runnable r : q.toArray(new Runnable[0])) {
             if (q.remove(r))
                 taskList.add(r);
         }
     }
     return taskList;
 }
```



### newFixedThreadPool

```
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads,  // corePoolSize
                                      nThreads,  // maximumPoolSize == corePoolSize
                                      0L,        // 空闲时间限制是 0
                                      TimeUnit.MILLISECONDS,
									  // 无界阻塞队列
                                      new LinkedBlockingQueue<Runnable>());
    }
```



### newCacheThreadPool

```
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0,  // corePoolSoze == 0
        							  Integer.MAX_VALUE, // maximumPoolSize 非常大
                                      60L, 			// 空闲判定是60 秒
                                      TimeUnit.SECONDS,
                                      // 神奇的无存储空间阻塞队列，每个 put 必须要等待一个 take
                                      new SynchronousQueue<Runnable>());
}
```



### newSingleThreadPool

```
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 
            						1,
                                    0L, 
                                    TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

可以看到除了多了个 `FinalizableDelegatedExecutorService` 代理，其初始化和 `newFiexdThreadPool` 的 nThreads = 1 的时候是一样的。 

区别就在于：

1、newSingleThreadExecutor返回的ExcutorService在析构函数finalize()处会调用shutdown() 

```
static class FinalizableDelegatedExecutorService
        extends DelegatedExecutorService {
        FinalizableDelegatedExecutorService(ExecutorService executor) {
            super(executor);
        }
        protected void finalize() {
            super.shutdown();
        }
    }
```



2、如果我们没有对它调用shutdown()，那么可以确保它在被回收时调用shutdown()来终止线程。



### 总结

从线程池的简单架构入手，分析了线程池的结构，紧接着分析了线程池的重要实现ThreadPoolExecutor，线程池5种状态:

1、RUNNING:  接受新任务，处理任务队列中的任务

2、SHUTDOWN: 不接受新任务，处理任务队列中的任务

3、STOP:    不接受新任务，不处理任务队列中的任务

4、TIDYING:  所有任务完成，线程数为0，然后执行terminated()

5、TERMINATED: terminated() 已经完成

在ThreadPoolExecutor，真正的任务是Worker对象，Worker 中的任务委托给了runWorker方法来实现。

线程池本身是一种生产者---消费者模式，生产者提交任务，消费者获取任务，然后执行任务，当任务执行完后，再从任务队列中取任务来执行。

通过对ThreadPoolExecutor的参数的控制，可以形成各种不同场景下应用的线程池，这个我们在后面再来分析。


**参考：Java 并发编程的艺术**