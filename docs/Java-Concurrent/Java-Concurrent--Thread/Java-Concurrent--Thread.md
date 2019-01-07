

在多线程编程中，如果要使用线程来执行任务，那么最简单的方式就是使用Thread类来创建一个线程，当然也可以使用线程池的方式。

线程是在进程中执行的单位，线程的资源开销相对于进程的开销是相对较少的，所以我们一般创建线程执行，而不是进程执行。

本文不是学习Thread的使用，而是通过Thread类来一探线程从创建到结束的过程。

### 继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180108095108423.png)

Thread类实现了Runnable接口，因此Thread不仅是一个线程类，也是一个特殊的执行任务类。

### 数据结构
```
  //线程名称的字节数组
  private volatile char  name[];
  //线程优先级
  private int            priority;
  private Thread         threadQ;
  private long           eetop;
  //线程是否单步
  private boolean     single_step;
  
  //是否是守护线程
  private boolean     daemon = false;
  
  /* JVM state */
  private boolean     stillborn = false;
  
  // 要执行的run方法的对象
  private Runnable target;
  
  // 这个线程的线程组
  private ThreadGroup group;
  
  /* The context ClassLoader for this thread */
  // 这个线程的上下文类加载器
  private ClassLoader contextClassLoader;
  
  /* The inherited AccessControlContext of this thread */
  private AccessControlContext inheritedAccessControlContext;
  
  //线程编号
  private static int threadInitNumber;

  /* ThreadLocal values pertaining to this thread. This map is maintained
   * by the ThreadLocal class. 
   */
  // ThreadLocal相关 
  ThreadLocal.ThreadLocalMap threadLocals = null;
  /*
   * InheritableThreadLocal values pertaining to this thread. This map is
   * maintained by the InheritableThreadLocal class.
   */
  ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
  /*
   * The requested stack size for this thread, or 0 if the creator did
   * not specify a stack size.  It is up to the VM to do whatever it
   * likes with this number; some VMs will ignore it.
   */
   // 给这个线程设置的栈的大小,默认为0 
  private long stackSize;
  /*
   * JVM-private state that persists after native thread termination.
   */
  private long nativeParkEventPointer;
   
   //线程id
  private long tid;
  
  /* For generating thread ID */
  private static long threadSeqNumber;
  
  /* Java thread status for tools,
   * initialized to indicate thread 'not yet started'
   */
  //初始状态 
  private volatile int threadStatus = 0;
  
  /**
   * The argument supplied to the current call to
   * java.util.concurrent.locks.LockSupport.park.
   * Set by (private) java.util.concurrent.locks.LockSupport.setBlocker
   * Accessed using java.util.concurrent.locks.LockSupport.getBlocker
   */
  volatile Object parkBlocker;
  
  /* The object in which this thread is blocked in an interruptible I/O
   * operation, if any.  The blocker's interrupt method should be invoked
   * after setting this thread's interrupt status.
   */
  private volatile Interruptible blocker;
  
  // 用于blocker 的锁对象
  private final Object blockerLock = new Object();

  /**
   * The minimum priority that a thread can have.
   */
   //最低优先级
  public final static int MIN_PRIORITY = 1;
 /**
   * The default priority that is assigned to a thread.
   */
   // 线程默认的执行优先级为 5
  public final static int NORM_PRIORITY = 5;
  /**
   * The maximum priority that a thread can have.
   */
   // 线程执行的最高的优先级为 10
  public final static int MAX_PRIORITY = 10;
```
上面的属性包含了线程的基本属性，Thread 并不是一个执行任务，而是作为一个线程单位存在，其内部拥有可执行任务 target（Runnable 类型），这个我们在后面的方法中将会更加细致的探讨。
线程的优先级在不同的平台上，对应的系统优先级会不同，可能多个优先级对应同一个系统优先级，优先级高的线程并不一定优先执行，这个由JVM来解释并向系统提供参考。
#### 线程状态（生命周期）

```
public enum State {
    /**
     * Thread state for a thread which has not yet started.
     */
    NEW,

    /**
     * Thread state for a runnable thread.  A thread in the runnable
     * state is executing in the Java virtual machine but it may
     * be waiting for other resources from the operating system
     * such as processor.
     */
    RUNNABLE,

    /**
     * Thread state for a thread blocked waiting for a monitor lock.
     * A thread in the blocked state is waiting for a monitor lock
     * to enter a synchronized block/method or
     * reenter a synchronized block/method after calling
     */
    BLOCKED,

    /**
     * Thread state for a waiting thread.
     */
    WAITING,

    /**
     * Thread state for a waiting thread with a specified waiting time.
     */
    TIMED_WAITING,

    /**
     * Thread state for a terminated thread.
     * The thread has completed execution.
     */
    TERMINATED;
}
```
线程的状态有NEW，RUNNABLE，BLOCKED，WAITING，TIMED_WAITING，TERMINATED，官方文档说得也很详细。

### 构造方法
1、无参构造

```
public Thread() {
    init(null, null, "Thread-" + nextThreadNum(), 0);
}
```
2、指定执行任务
```
public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
}
```
3、指定线程组和执行任务

```
public Thread(ThreadGroup group, Runnable target) {
    init(group, target, "Thread-" + nextThreadNum(), 0);
}
```
还有其它很多的构造方法，这里就不一一罗列出来了，在构造方法中，都调用了init 方法来进行初始化。

#### init 方法
```
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }
    // 设置线程名称
    this.name = name.toCharArray();
    //当前调用线程为 父线程
    Thread parent = currentThread();
    SecurityManager security = System.getSecurityManager();
    if (g == null) {
        /* Determine if it's an applet or not */

        /* If there is a security manager, ask the security manager
           what to do. */
        if (security != null) {
            g = security.getThreadGroup();
        }

        /* If the security doesn't have a strong opinion of the matter
           use the parent thread group. */
        if (g == null) {
            g = parent.getThreadGroup();
        }
    }

    /* checkAccess regardless of whether or not threadgroup is
       explicitly passed in. */
    g.checkAccess();

    /*
     * Do we have the required permissions?
     */
    if (security != null) {
        if (isCCLOverridden(getClass())) {
            security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
        }
    }

    g.addUnstarted();

    this.group = g;
    // 继承父线程的相关属性
    this.daemon = parent.isDaemon();
    this.priority = parent.getPriority();
    if (security == null || isCCLOverridden(parent.getClass()))
        this.contextClassLoader = parent.getContextClassLoader();
    else
        this.contextClassLoader = parent.contextClassLoader;
    this.inheritedAccessControlContext =
            acc != null ? acc : AccessController.getContext();
    // 设置执行任务        
    this.target = target;
    setPriority(priority);
    if (parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    /* Stash the specified stack size in case the VM cares */
    this.stackSize = stackSize;

    /* Set thread ID */
    // 设置线程id，nextThreadID()进行了同步处理
    tid = nextThreadID();
}
```
构造方法是被调用者线程执行的，因此调用者线程将作为父线程，新创建的线程继承父线程的部分属性。

###  run 方法
Thread继承Runnable 接口，那么就需要实现run 方法。

```
public void run() {
    if (target != null) {
        target.run();
    }
}
```
Thread中的run 方法调用的还是目标任务的run 方法，通过直接调用run 方法并不会创建线程来单独执行任务，这个我们通过Thread中的run 也可以看出来，要创建线程来执行任务，需要调用Thread的start 方法。
### start 方法

```
/**
 * Causes this thread to begin execution; the Java Virtual Machine
 * calls the run method of this thread.
 */
public synchronized void start() {
    /**
     * This method is not invoked for the main method thread or "system"
     * group threads created/set up by the VM. Any new functionality added
     * to this method in the future may have to also be added to the VM.
     *
     * A zero status value corresponds to state "NEW".
     */
    if (threadStatus != 0) // 如果线程不是初始状态，那么会抛出异常
        throw new IllegalThreadStateException();

    /* Notify the group that this thread is about to be started
     * so that it can be added to the group's list of threads
     * and the group's unstarted count can be decremented. 
     */
    //将启动的线程添加到线程组 
    group.add(this);

    boolean started = false;
    try {
        start0(); //调用本地方法
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {
            /* do nothing. If start0 threw a Throwable then
              it will be passed up the call stack */
        }
    }
}
private native void start0();
```
start方法是同步的（synchronized 修饰），在启动线程前会检查线程状态，如果线程不是初始状态，那么会抛出异常，因此多次调用start 是不可行的。

start 方法被调用者线程执行，JVM将会调用这个线程的run方法，这样产生的结果是，两个线程执行着，其中一个是调用start()方法的线程执行，另一个线程是执行run方法的线程。

### sleep()方法
```
 /**
  * Causes the currently executing thread to sleep (temporarily cease
  * execution) for the specified number of milliseconds plus the specified
  * number of nanoseconds, subject to the precision and accuracy of system
  * timers and schedulers. The thread does not lose ownership of any
  * monitors.
  */
 public static void sleep(long millis, int nanos)
 throws InterruptedException {
     if (millis < 0) {
         throw new IllegalArgumentException("timeout value is negative");
     }
     if (nanos < 0 || nanos > 999999) {
         throw new IllegalArgumentException(
                             "nanosecond timeout value out of range");
     }
     if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
         millis++;
     }
     sleep(millis);  // 调用本地sleep 方法
 }
```
 sleep方法的作用使得当前线程休眠一定的时间，值得注意的是这个期间是**不会释放持有的锁**。

 ### join()方法
```
/**
 * Waits at most {@code millis} milliseconds for this thread to
 * die. A timeout of {@code 0} means to wait forever.
 */
public final synchronized void join(long millis)
 throws InterruptedException {
     long base = System.currentTimeMillis();
     long now = 0;

     if (millis < 0) {
         throw new IllegalArgumentException("timeout value is negative");
     }

     if (millis == 0) {
         //一直等待，直到目标线程结束
         while (isAlive()) {
             wait(0);
         }
     } else {
         while (isAlive()) {
             long delay = millis - now;
             if (delay <= 0) {
                 break;
             }
             wait(delay); // 超时等待
             now = System.currentTimeMillis() - base;
         }
     }
 }
```
调用线程通过join方法等待被调用线程任务执行，直到超时或者终止。

### interrupt()方法

```
public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();
    // 同步处理
    synchronized (blockerLock) {
        Interruptible b = blocker;
        if (b != null) {
            interrupt0();           // Just to set the interrupt flag
            b.interrupt(this);
            return;
        }
    }
    interrupt0(); //调用本地方法
}
```
interrupt()方法是中断当前的线程（设置中断标志位），线程当检测到中断标志位被设置后，可能会抛出InteruptedExeption, 同时会清除线程的中断状态，通常中断可以作为取消任务的一种比较安全的方式（前提是能响应中断）。

### 总结
通过对源码的简单分析，了解了Thread一些使用方法和注意事项，整个过程对Thread的分析是比较浅显的，当然对多线程的研究也才刚刚开始，并没有完全深入各个细节，但是通过源码的分析，知道了线程整个创建的大致过程，对于使用Thread也会更加得心应手。