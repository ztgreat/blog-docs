## 前言

前面 分析了Netty整体的流程 和 Channel的结构，我们看到 Netty 中有很多的异步调用，所以在介绍更多 NIO 相关的内容之前，我们来看看它的异步接口是怎么实现的。

## 回顾

前面我们在介绍 Echo 例子的时候，已经用过了 ChannelFuture 这个接口了，接下来我们就来看看 Netty 中的异步调用是如何实现的。

### 客户端

```
// Start the client.
ChannelFuture future = b.connect(HOST, PORT);
future.sync();

// Wait until the connection is closed.
Channel channel= future.channel();
ChannelFuture closeFuture=channel.closeFuture();
closeFuture.sync();
```

### 服务端

```
// Start the server.
ChannelFuture future = b.bind(PORT);
future.sync();

// Wait until the server socket is closed.
Channel channel = future.channel();
ChannelFuture closeFuture = channel.closeFuture();
closeFuture.sync();
```

特意把代码拆开，方便理解，可以看到 其实客户端和服务端 结构是差不多的，相信分析了Future 结构后，我们对上面的代码理解会更加的深刻。

## JDK 中的 Future

关于 Future 接口，常用的就是在使用 Java 的线程池 ThreadPoolExecutor 的时候了。在 submit 一个任务到线程池中的时候，返回的就是一个 **Future** 实例，通过它来获取提交的任务的执行状态和最终的执行结果。

下面是 JDK 中的 Future 接口 `java.util.concurrent.Future`：

```
public interface Future<V> {
    // 取消该任务
    boolean cancel(boolean mayInterruptIfRunning);
    // 任务是否已取消
    boolean isCancelled();
    // 任务是否已完成
    boolean isDone();
    // 阻塞获取任务执行结果
    V get() throws InterruptedException, ExecutionException;
    // 带超时参数的获取任务执行结果
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

Netty 中的 Future 接口继承了 JDK 中的 Future 接口，然后添加了一些方法：

## Netty 中的Future

` io.netty.util.concurrent.Future`:

```
public interface Future<V> extends java.util.concurrent.Future<V> {

    // 是否成功
    boolean isSuccess();

    // 是否可取消
    boolean isCancellable();

    // 如果任务执行失败，这个方法返回异常信息
    Throwable cause();

    // 添加 Listener 来进行回调
    Future<V> addListener(GenericFutureListener<? extends Future<? super V>> listener);
    Future<V> addListeners(GenericFutureListener<? extends Future<? super V>>... listeners);
    // 移除 Listener
    Future<V> removeListener(GenericFutureListener<? extends Future<? super V>> listener);
    Future<V> removeListeners(GenericFutureListener<? extends Future<? super V>>... listeners);

    // 阻塞等待任务结束，如果任务失败，将“导致失败的异常”重新抛出来
    Future<V> sync() throws InterruptedException;
    // 不响应中断的 sync()
    Future<V> syncUninterruptibly();

    // 阻塞等待任务结束，和 sync() 功能是一样的，不过如果任务失败，它不会抛出执行过程中的异常
    Future<V> await() throws InterruptedException;
    Future<V> awaitUninterruptibly();
    boolean await(long timeout, TimeUnit unit) throws InterruptedException;
    boolean await(long timeoutMillis) throws InterruptedException;
    boolean awaitUninterruptibly(long timeout, TimeUnit unit);
    boolean awaitUninterruptibly(long timeoutMillis);

    // 获取执行结果，不阻塞,如果没数据，返回 NULL。
    V getNow();

    // 取消任务执行
    @Override
    boolean cancel(boolean mayInterruptIfRunning);
}

```

我们可以发现， Netty 的 Future 接口 扩展了 JDK 中 Future 接口，它加了 sync() 和 await() 用于阻塞等待，还加了 Listeners，只要任务结束去回调 Listener 们就可以了。

这里顺便说下 sync() 和 await() 的区别：sync() 内部会先调用 await() 方法，等 await() 方法返回后，会检查下**这个任务是否失败**，如果失败，重新将导致失败的异常抛出来，这个我们将在后面的实现类中看到。

### Future 体系

![Future 体系](http://img.blog.ztgreat.cn/document/netty/20190112153742.png)

上面罗列了Future的部分结构，未全部列出来，稍后我们将分析 ChannelFuture,已经Promise。

###  ChannelFuture

 Future 接口的子接口 ChannelFuture，它将和 IO 操作中的 Channel 关联在一起了，用于异步处理 Channel 中的事件。

```
public interface ChannelFuture extends Future<Void> {

    // ChannelFuture 关联的 Channel
    Channel channel();

    // 覆写以下几个方法，使得它们返回值为 ChannelFuture 类型 
    @Override
    ChannelFuture addListener(GenericFutureListener<? extends Future<? super Void>> listener);
    @Override
    ChannelFuture addListeners(GenericFutureListener<? extends Future<? super Void>>... listeners);
    @Override
    ChannelFuture removeListener(GenericFutureListener<? extends Future<? super Void>> listener);
    @Override
    ChannelFuture removeListeners(GenericFutureListener<? extends Future<? super Void>>... listeners);

    @Override
    ChannelFuture sync() throws InterruptedException;
    @Override
    ChannelFuture syncUninterruptibly();

    @Override
    ChannelFuture await() throws InterruptedException;
    @Override
    ChannelFuture awaitUninterruptibly();

    boolean isVoid();
}

```

我们看到，ChannelFuture 接口相对于 Future 接口，除了将 channel 关联进来，没有增加什么东西，其他几个都是方法覆写，为了让返回值类型变为 ChannelFuture。

### Promise

接下来 我们来介绍下 Promise 接口，它和 ChannelFuture 接口无关，而是和前面的 Future 接口相关，也就是说Promise的和ChannelFuture是独立的，在js 或者 nodejs 中都有一个 Promise的概念。

Promise 这个接口其实可以算作是一个异步任务的抽象，这个我们在后面具体编程的时候，再来看。

Promise 接口和 ChannelFuture 一样，也继承了 Netty 的 Future 接口，然后加了一些 Promise 的内容：

```
public interface Promise<V> extends Future<V> {

    // 设置 该 future 成功及设置其执行结果，然后通知所有的 listeners。
    // 如果该操作失败，将抛出异常
    Promise<V> setSuccess(V result);

    // 和 setSuccess 方法一样，只不过如果失败，它不抛异常，返回 false
    boolean trySuccess(V result);

    // 设置 该 future 失败，及其失败原因。
    // 如果该操作失败，将抛出异常
    Promise<V> setFailure(Throwable cause);

    // 设置 该 future 失败，及其失败原因。
    // 如果该操作失败，返回 false，不抛出异常
    boolean tryFailure(Throwable cause);

    // 设置该 future 不可以被取消
    boolean setUncancellable();

    // 覆写，返回 Promise 类型的实例
    @Override
    Promise<V> addListener(GenericFutureListener<? extends Future<? super V>> listener);
    @Override
    Promise<V> addListeners(GenericFutureListener<? extends Future<? super V>>... listeners);

    @Override
    Promise<V> removeListener(GenericFutureListener<? extends Future<? super V>> listener);
    @Override
    Promise<V> removeListeners(GenericFutureListener<? extends Future<? super V>>... listeners);

    @Override
    Promise<V> await() throws InterruptedException;
    @Override
    Promise<V> awaitUninterruptibly();

    @Override
    Promise<V> sync() throws InterruptedException;
    @Override
    Promise<V> syncUninterruptibly();
}

```

### ChannelPromise

接下来，我们再来看下 **ChannelPromise**，它继承了前面介绍的 ChannelFuture 和 Promise 接口，这样ChannelPromise 就联系上了Chaneel了。

```
/**
 * Special {@link ChannelFuture} which is writable.
 */
public interface ChannelPromise extends ChannelFuture, Promise<Void> {

    //覆写 ChannelFuture 中的 channel() 方法
    @Override
    Channel channel();

    //覆写，目的是为了返回值类型是 ChannelPromise
    @Override
    ChannelPromise setSuccess(Void result);
    ChannelPromise setSuccess();
    boolean trySuccess();
    @Override
    ChannelPromise setFailure(Throwable cause);

    //覆写也是为了得到 ChannelPromise 类型的实例
    @Override
    ChannelPromise addListener(GenericFutureListener<? extends Future<? super Void>> listener);
    @Override
    ChannelPromise addListeners(GenericFutureListener<? extends Future<? super Void>>... listeners);
    @Override
    ChannelPromise removeListener(GenericFutureListener<? extends Future<? super Void>> listener);
    @Override
    ChannelPromise removeListeners(GenericFutureListener<? extends Future<? super Void>>... listeners);

    @Override
    ChannelPromise sync() throws InterruptedException;
    @Override
    ChannelPromise syncUninterruptibly();
    @Override
    ChannelPromise await() throws InterruptedException;
    @Override
    ChannelPromise awaitUninterruptibly();

    /**
     * Returns a new {@link ChannelPromise} if {@link #isVoid()} returns {@code true} otherwise itself.
     */
    ChannelPromise unvoid();
}

```

我们可以看到，它综合了 ChannelFuture 和 Promise 中的方法，只不过通过覆写将返回值都变为 ChannelPromise 了而已，没有增加什么新的功能。

我们上面介绍了几个接口，Future 以及它的子接口 ChannelFuture 和 Promise，然后是 ChannelPromise 接口同时继承了 ChannelFuture 和 Promise。

接下来，我们需要来一个实现类，这样才能比较直观地看出它们是怎么使用的，因为上面的这些都是接口定义，具体还得看实现类是怎么工作的。

### **DefaultPromise** 

下面，我们来介绍下 **DefaultPromise** 这个实现类，内容有点多，我们就介绍几个关键的内容。

首先，我们看下它有哪些属性：

```
public class DefaultPromise<V> extends AbstractFuture<V> implements Promise<V> {
    // 任务的执行结果
    private volatile Object result;
    // 执行任务的线程池，promise 持有 executor 的引用
    private final EventExecutor executor;
    // 监听者，回调函数
    private Object listeners;

    // 调用sync()/await()进行等待的线程数量
    private short waiters;

    // 是否正在唤醒等待线程，用于防止重复执行唤醒，不然会重复执行 listeners 的回调方法
    private boolean notifyingListeners;
    ......
}

```

> DefaultPromise 实现了 Promise，但是没有实现 ChannelFuture，所以它还没有和 Channel 联系起来。
>

我们 看到 DefaultPromise 中持有了一个 线程池引用。

在 DefaultPromise  中 我们看到有如下注释：

```
/**
 * Get the executor used to notify listeners when this promise is complete.
 * <p>
 * It is assumed this executor will protect against {@link StackOverflowError} exceptions.
 * The executor may be used to avoid {@link StackOverflowError} by executing a {@link Runnable} if the stack
 * depth exceeds a threshold.
 * @return The executor used to notify listeners when this promise is complete.
 */
protected EventExecutor executor() {
    return executor;
}
```

当任务执行后，会进行 **Listener**  的调用，而**Listener**  的调用逻辑 这个是不清楚的，有可能是同步的，也可能是异步的，因此用线程池去执行，这样将 **回调任务** 和 和 **任务的执行** 进行了分割。

我们来看看 DefaultPromise 中的一些方法：

#### addListener

添加监听者（回调函数）

```
@Override
public Promise<V> addListener(GenericFutureListener<? extends Future<? super V>> listener) {
    checkNotNull(listener, "listener");
    //同步处理
    synchronized (this) {
        addListener0(listener);
    }
    //添加完后，判断任务是否完成了，如果是，则需要进行通知 
    if (isDone()) {
        notifyListeners();
    }
    return this;
}
```

#### isDone

判断任务是否执行完成了，这个判断 结果值类型就可以了，很简单。

```
@Override
public boolean isDone() {
    return isDone0(result);
}
private static boolean isDone0(Object result) {
    return result != null && result != UNCANCELLABLE;
}
```

#### setSuccess

设置 该 future 成功及设置其执行结果，并且会通知所有的 listeners。

如果该操作失败，将抛出异常

```
@Override
public Promise<V> setSuccess(V result) {
    //设置结果
    if (setSuccess0(result)) {
        // 如果设置成功，则开始进行回调处理
        notifyListeners();
        return this;
    }
    throw new IllegalStateException("complete already: " + this);
}

private boolean setSuccess0(V result) {
    return setValue0(result == null ? SUCCESS : result);
}
private boolean setValue0(Object objResult) {
    if (RESULT_UPDATER.compareAndSet(this, null, objResult) ||
        RESULT_UPDATER.compareAndSet(this, UNCANCELLABLE, objResult)) {
        checkNotifyWaiters();
        return true;
    }
    return false;
}

```

#### trySuccess

设置 该 future 成功及设置其执行结果，并且会通知所有的 listeners。

如果该操作失败，返回false，不抛出异常。

```
@Override
public boolean trySuccess(V result) {
    if (setSuccess0(result)) {
        notifyListeners();
        return true;
    }
    return false;
}
```

#### setFailure

设置 该 future 失败，及其失败原因。

如果该操作失败，将抛出异常

```
@Override
public Promise<V> setFailure(Throwable cause) {
    if (setFailure0(cause)) {
        notifyListeners();
        return this;
    }
    throw new IllegalStateException("complete already: " + this, cause);
}
```

#### tryFailure

设置 该 future 失败，及其失败原因。

如果该操作 失败，返回false，不抛出异常。

```
@Override
public boolean tryFailure(Throwable cause) {
    if (setFailure0(cause)) {
        notifyListeners();
        return true;
    }
    return false;
}
```

上面几个方法都非常简单，先设置好值，然后执行监听者们的回调方法。notifyListeners() 方法感兴趣的读者也可以看一看，不过它还涉及到 Netty 线程池的一些内容，我们还没有介绍到线程池，这里就不展开了。上面的代码，在 setSuccess0 或 setFailure0 方法中都会唤醒阻塞在 sync() 或 await() 的线程

另外，就是可以看下 sync() 和 await() 的区别，其他的我觉得随便看看就好了。

#### sync

```
@Override
public Promise<V> sync() throws InterruptedException {
    await();
    // 如果任务是失败的，重新抛出相应的异常
    rethrowIfFailed();
    return this;
}

```

我们看到 sync 内部会调用await 方法，只是如果 await 执行失败，那么会再次抛出异常问题。

#### 实例代码

我们通过一个实例，来了解 Promise的使用，以及监听器的使用：

```
public static void main(String[] args){

   // 构造线程池
   EventExecutor executor = new DefaultEventExecutor();

   // 创建 DefaultPromise 实例
   Promise promise = new DefaultPromise(executor);

   // 添加 一个 listener
    promise.addListener(new GenericFutureListener<Future<String>>() {
         @Override
          public void operationComplete(Future future) throws Exception {
                if (future.isSuccess()) {
                    System.out.println("任务成功,结果：" + future.get());
                } else {
                    System.out.println("任务失败,异常：" + future.cause());
                }
           }
    });
    // 提交任务到线程池
    executor.submit(new Runnable() {
         @Override
          public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                }
                // 设置 promise 的结果
                // promise.setFailure(new RuntimeException());
                promise.setSuccess("success");
           }
     });

     // 主线程线程阻塞等待执行结果
      try {
           promise.sync();
      } catch (InterruptedException e) {

      }
}
```

运行代码， 3 秒后将输出：

```
任务成功,结果：success
```

这里我们也可以试一下 sync() 和 await() 的区别，开启  promise.setFailure(new RuntimeException()) 代码，同时最后调用await 方法，对比区别，这里就不展示了，读者可以自行去尝试。

####  小结一下

对比JDK 中的Future 如果我们需要获取结果，我们需要调用get 方法获取，或者通过isDone 来判断任务是否完成，这都是**主动轮询**的方式。

而Netty中的Future  我们 可以用 await()，等 await() 方法返回后，得到 promise 的执行结果，然后处理它；

另一种就是提供 **Listener** 实例，**我们不太关心任务什么时候会执行完，只要它执行完了以后会去执行 listener 中我们定义的逻辑就可以了**。

在 DefaultPromise 中以及 上面的代码中，我们看到promise  持有了 线程池引用，这个是为什么呢？

在 DefaultPromise  中 我们看到有如下注释：

```
/**
 * Get the executor used to notify listeners when this promise is complete.
 * <p>
 * It is assumed this executor will protect against {@link StackOverflowError} exceptions.
 * The executor may be used to avoid {@link StackOverflowError} by executing a {@link Runnable} if the stack
 * depth exceeds a threshold.
 * @return The executor used to notify listeners when this promise is complete.
 */
protected EventExecutor executor() {
    return executor;
}
```

当任务执行后，会进行 **Listener**  的调用，而**Listener**  的调用逻辑 这个是不清楚的，有可能是同步的，也可能是异步的，因此用线程池去执行，这样将 **回调任务** 和 和 **任务的执行** 进行了分割。

### DefaultChannelPromise

DefaultChannelPromise 基本上都是调用 DefaultPromise的方法 ,实现了ChannelPromise 接口，将Channle 关联了进来。

```
public class DefaultChannelPromise extends DefaultPromise<Void> implements ChannelPromise, FlushCheckpoint {

  private final Channel channel;
  private long checkpoint;

  /**
  * Creates a new instance.
  *
  * @param channel
  *        the {@link Channel} associated with this future
  */
  public DefaultChannelPromise(Channel channel) {
      this.channel = checkNotNull(channel, "channel");
  }
  //... 省略其它方法
}
```

有了上面的认识，下面 我们回过头来再看**客户端中Future**的调用：

```
// Start the client.
ChannelFuture future = b.connect(HOST, PORT);
future.sync();

// Wait until the connection is closed.
Channel channel= future.channel();
ChannelFuture closeFuture=channel.closeFuture();
closeFuture.sync();
```

当客户端进行 connect 后，返回了一个 Future，这个connect 是异步的，因此还不知道任务执行如何，因此这里我们调用 future的sync，等待connect 完成。

当connect 后，我们从future 中获取 Channel, 再从这个 Channle 获取一个 Future,而这个Future 是监听 Channle 是否关闭的，通过sync 方法，我们可以一直等待Channle的关闭，因此如果不进行`closeFuture.sync()`,那么主线程就会直接执行完毕，不会阻塞在最后。