### 中断概述
Java中断机制是一种协作机制，也就是说**通过中断并不能直接终止另一个线程，而需要被中断的线程自己处理中断**，并不是字面意思那样能马上中断操作。
每一个线程都有一个boolean类型的中断状态(不一定就要是Thread类的字段，实际上也的确不是，这几个方法最终都是通过native方法来完成的)，在中断的时候，这个中断状态被设置为true。

java.lang.Thread类提供了几个方法来操作这个中断状态，这些方法包括：

**(1)public static boolean interrupted:**

  测试当前线程是否已经中断。线程的中断状态 由该方法清除。换句话说，如果连续两次调用该方法，则第二次调用将返回 false（在第一次调用已清除了其中断状态之后，且第二次调用检验完中断状态前，当前线程再次中断的情况除外）。

 **(2)public boolean isInterrupted():**

测试线程是否已经中断。线程的中断状态不受该方法的影响。

**(3)public void interrupt():**

中断线程。



其中，**interrupt方法是唯一能将中断状态设置为true的方法**。**静态方法interrupted会将当前线程的中断状态清除（设置为false）,并返回它之前的值，这是清除中断状态唯一的方法**。

先来看一段代码：

```
public class Interrupted {

    //不断睡眠的线程（阻塞）
    static  class SleepRunner implements  Runnable{

        public void run() {
            while(true){
                try {
                    Thread.sleep(10000);
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        }
    }

    // 不停忙碌的线程（非阻塞）
    static  class BusyRunner implements  Runnable{

        public void run() {
            while (true){

            }
        }
    }

    public static void main(String[] args) throws  Exception{
        Thread sleepThread=new Thread(new SleepRunner());
        Thread busyThread=new Thread(new BusyRunner());

        sleepThread.start();
        busyThread.start();

        Thread.sleep(5000);

        //中断睡眠线程
        sleepThread.interrupt();

        //中断忙碌线程
        busyThread.interrupt();
        System.out.println("SleepThread interrupted is "+sleepThread.isInterrupted());
        System.out.println("BusyThread interrupted is "+busyThread.isInterrupted());
    }

}
```
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170906202540513.png)

从结果可以看出，抛出InterruptedException的线程SleepThread,其中断标志位被清除了，而一直忙碌运作的线程BusyThread,中断标志位没有被清除。
下面是sleep 方法，注意看注释：
```
/**
 * Causes the currently executing thread to sleep (temporarily cease
 * execution) for the specified number of milliseconds, subject to
 * the precision and accuracy of system timers and schedulers. The thread
 * does not lose ownership of any monitors.
 *
 * @param  millis
 *         the length of time to sleep in milliseconds
 *
 * @throws  IllegalArgumentException
 *          if the value of {@code millis} is negative
 *
 * @throws  InterruptedException
 *          if any thread has interrupted the current thread. The
 *          <i>interrupted status</i> of the current thread is
 *          cleared when this exception is thrown.
 */
public static native void sleep(long millis) throws InterruptedException;
```

根据sleep方法的说明我们知道，在抛出InterruptedException之前，会先清除中断状态，然后向外层抛出中断异常。
SleepThread 在发生中断后停止运行了，而BusyThread 发生中断后并没有停止，但是中断标志位被设置了，所以我们知道了
**调用interrupt并不意味着必然停止目标线程正在进行的工作，它仅仅传递了请求中断的消息（设置中断标志位）**。
所以说我们**对中断本身最好的理解应该是：它并不会真正中断一个正在运行的线程；它仅仅是发出中断请求，线程自己会在下一个方便的时刻中断自己**。

sleep方法中进行了中断检测，当睡醒了后，检测是否被设置了中断标志位，如果是则抛出中断异常同时清除中断标志位，否则检查是否还应该继续睡眠。


既然如此，那么为什么要抛出中断异常了？，继续看下面


### 中断策略

对于多线程的使用，我们一般将任务封装为一个Runnable，然后用线程池来对任务进行操作（执行，取消，获得结果），任务不会在自己拥有的线程中执行，它们借用属于服务的线程（线程池），调用者通过对任务返回情况进行处理，不用的调用者的处理可能不一样，因此**内部的异常信息应该尽可能的向外传递**，传给调用者，这样上层调用代码就可以进一步行动了。
这就是为什么大多数可阻塞的库函数中，仅仅抛出InterruptedException作为中断响应，它们不能自己处理中断信息，而是应该交给调用者来处理。

当检查到中断请求时，任务并不需要立即停止所有事情，它可以选择推迟，直到更合适的时机。这就需要记得它已经被请求中断了，完成当前正在进行的任务，然后抛出InterruptedException或者指明中断。这样使得在返回前，可以处理一些结束的事情，使得停止不是那么突然，保证数据结构不被彻底破坏。

### 中断响应


当你调用可中断的阻塞函数时，有两种处理InterruptedException的策略：

#### 1. 传递异常
传递InterruptedException 只需要简单的在方法中添加 throws InterruptedException 这样就可以抛出向上传递异常，
正如sleep方法那样。
#### 2.保存中断状态，上层调用栈中的代码能够对其进行处理
若有时候不太方便在方法上抛出InterruptedException，比如要实现的某个接口中的方法签名上没有throws InterruptedException，这时就可以捕获可中断方法的InterruptedException并通过Thread.currentThread.interrupt()来重新设置中断状态，一般的代码中，尤其是作为一个基础类库时你不应该掩盖InterruptedException（在cat catch块中捕获到异常却什么都不做）因为吞掉中断状态会导致方法调用栈的上层得不到这些信息。
当然，凡事总有例外的时候，当你完全清楚自己的方法会被谁调用，而调用者也不会因为中断被吞掉了而遇到麻烦，就可以随便处理，例如我们自己有一个sleep方法，但是该方法不响应中断，那么我们就可以这样做：

```
//不断睡眠的线程（阻塞）
static  class SleepRunner implements  Runnable{
    public void run() {

        TimeUnit timeUnit=TimeUnit.SECONDS;

        boolean interrupted=sleep(timeUnit.toNanos(10));
        System.out.println("退出睡眠");
        if (interrupted=true)
            System.out.println("我被中断过");
        else
            System.out.println("我没被中断过");

    }
}
```

内部调用封装好了的sleep方法，该方法对中断不响应，但是结束后会返回是否被中断。
```
public static boolean sleep(long time){
    final long deadline = System.nanoTime() + time;
    boolean interrupted=false;
    TimeUnit timeUnit=TimeUnit.NANOSECONDS;
    while(true){

        try{
            Thread.sleep(timeUnit.toMillis(time));
        }catch (InterruptedException e){
            interrupted=true;
            System.out.println("被中断啦");
        }
        time = deadline - System.nanoTime();
        if (time <= 0L)
            return interrupted;
        System.out.println("继续睡眠");
    }
}
```

我们传入一个纳秒时间，然后调用系统sleep 方法，同时catch InterruptedException ，在catch 到中断异常后，设置一个标记，标识被中断过，然后判断是否还应该继续睡觉，如果是则继续睡眠，否则返回结果，这样我们在上层调用者就可以知道在睡眠过程中是否发生过中断了。

此外我们可以通过另一种来响应中断，通过重新设置中断标志位而不是抛出中断异常，这样上层调用者可以通过检查该标志位来判断该线程是否发生过中断。

```
public static void sleepInterrupted(long time){

    TimeUnit timeUnit=TimeUnit.NANOSECONDS;
    try{
        Thread.sleep(timeUnit.toMillis(time));
    }catch (InterruptedException e){
        System.out.println("被中断啦");
        Thread.currentThread().interrupt();
    }
}
```

这种重新设置了中断标志，一般就不在继续任务了（不在调用阻塞库函数），这种不抛出异常

```
//不断睡眠的线程（阻塞）
static  class SleepRunner implements  Runnable{
    public void run() {

        TimeUnit timeUnit=TimeUnit.SECONDS;

        sleepInterrupted(timeUnit.toNanos(10));
        while (true){

        }
    }
}
```

这里被中断后采用死循环，避免在父线程未检测之前就结束了。

```
Thread.sleep(2000);
System.out.println("SleepThread interrupted is "+sleepThread.isInterrupted());
System.out.println("BusyThread interrupted is "+busyThread.isInterrupted());
```

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170906213418196.png)

到此为止，我们真正知道了什么是中断，不能指望中断马上能取消掉任务，但是可以利用中断很优雅的取消任务，对于中断的响应策略，即可以掩盖掉，也可以向上传递，这个在具体环境中会有所不同。
