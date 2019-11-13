



>本文 转载  RateLimiter 源码分析(Guava 和 Sentinel 实现)（作者javadoop）
>
>原文链接https://www.javadoop.com/post/rate-limiter
>
>本文 在其基础 上 进行了个人的补充



在开发高并发系统时有三把利器用来保护系统：**缓存、降级和限流**

**缓存**：缓存的目的是提升系统访问速度和增大系统处理容量
**降级**：降级是当服务器压力剧增的情况下，根据当前业务情况及流量对一些服务和页面有策略的降级，以此释放服务器资源以保证核心任务的正常运行
**限流**：限流的目的是通过对并发访问/请求进行限速，或者对一个时间窗口内的请求进行限速来保护系统，一旦达到限制速率则可以拒绝服务、排队或等待、降级等处理

本文主要介绍关于Guava 中 流控的部分内容，阿里 的Sentinel 也具有流控的功能，而且根据灵活和丰富，这个在后面进行分析。

常用的限流算法有漏桶算法和令牌桶算法，

## 漏桶算法

漏桶算法思路很简单，请求先进入到漏桶里，漏桶以固定的速度出水，也就是处理请求，当水加的过快，则会直接溢出，也就是拒绝请求，可以看出漏桶算法能强行限制数据的传输速率，下图来自网上：

![漏桶算法](http://img.blog.ztgreat.cn/document/algorithm/5b6e490397691.png)





漏斗有一个进水口 和 一个出水口，出水口以一定速率出水，并且有一个最大出水速率：

在漏斗中没有水的时候，

- 如果进水速率小于等于最大出水速率，那么，出水速率等于进水速率，此时，不会积水
- 如果进水速率大于最大出水速率，那么，漏斗以最大速率出水，此时，多余的水会积在漏斗中

在漏斗中有水的时候

- 出水口以最大速率出水
- 如果漏斗未满，且有进水的话，那么这些水会积在漏斗中
- 如果漏斗已满，且有进水的话，那么这些水会溢出到漏斗之外

## 令牌桶算法

对于很多应用场景来说，除了要求能够限制数据的平均传输速率外，还要求允许某种程度的突发传输。这时候漏桶算法可能就不合适了，令牌桶算法更为适合。

令牌桶算法的原理是系统以恒定的速率产生令牌，然后把令牌放到令牌桶中，令牌桶有一个容量，当令牌桶满了的时候，再向其中放令牌，那么多余的令牌会被丢弃；当想要处理一个请求的时候，需要从令牌桶中取出一个令牌，如果此时令牌桶中没有令牌，那么则拒绝该请求，下图来自网上：



![令牌桶](http://img.blog.ztgreat.cn/document/algorithm/5b6e4903ec371.png)

guava的RateLimiter使用的是令牌桶算法，也就是以固定的频率向桶中放入令牌，例如一秒钟10枚令牌，实际业务在每次响应请求之前都从桶中获取令牌，只有取到令牌的请求才会被成功响应，获取的方式有两种：阻塞等待令牌或者取不到立即返回失败。


## Guava RateLimiter

### RateLimiter继承体系

![RateLimiter](http://img.blog.ztgreat.cn/document/algorithm/115b6e490397691.png)

RateLimiter 下面 有两个具体的 实现类，分别是 `SmoothBursty` ，`SmoothWarmingUp `。

#### SmoothBursty  

SmoothBursty  可以 具有预消费功能，可以应对突发流量，面对瞬时大流量，该算法可以在短时间内请求拿到大量令牌，而且拿令牌的过程并不是消耗很大的事情，但是预消费 也不是没有代价的，预消费 是指的提前消耗 后面的令牌，简单的来说，就是就是透支消费，那么也就说上一次请求获取的令牌数越多，那么下一次再获取授权时更待的时候会更长。

#### SmoothWarmingUp 

SmoothWarmingUp 适用于资源需要预热的场景，如果突发的大量流量到来，在没有预热的情况下，大量请求会打到后台的服务，如果后台服务的缓存陈旧，db和io操作耗时，就有可能会拖垮后台服务，如果有一个预热阶段的话就能够将流量比较平滑的过渡，从而降低后台服务down掉的风险。

### RateLimiter 使用介绍

RateLimiter 的接口非常简单，它有两个静态方法用来实例化，实例化以后，我们只需要关心 `acquire` 就行了

// RateLimiter 部分接口列表：

```java
// 两种方式实例化：
public static RateLimiter create(double permitsPerSecond){}
public static RateLimiter create(double permitsPerSecond,long warmupPeriod,TimeUnit unit) {}

public double acquire() {}
public double acquire(int permits) {}

public boolean tryAcquire() {}
public boolean tryAcquire(int permits) {}
public boolean tryAcquire(long timeout, TimeUnit unit) {}
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {}
```

RateLimiter 的作用是用来限流的，我们知道 java 并发包中提供了 Semaphore(信号量)，它也能够提供对资源使用进行控制，我们看一下下面的代码：

```java
// Semaphore
Semaphore semaphore = new Semaphore(10);
for (int i = 0; i < 100; i++) {
    executor.submit(new Runnable() {
        @Override
        public void run() {
            semaphore.acquireUninterruptibly(1);
            try {
                doSomething();
            } finally {
                semaphore.release();
            }
        }
    });
}
```

Semaphore 用来控制同时访问某个资源的并发数量，如上面的代码，我们设置 100 个线程工作，但是我们能做到最多只有 10 个线程能同时到 `doSomething()` 方法中。Semaphore 控制是 对临界区资源的访问，**它控制的是并发数量**。

而 RateLimiter 是用来控制访问资源的**速率（rate）**的，它强调的是`控制速率`。比如控制每秒只能有 100 个请求通过，比如允许每秒发送 1MB 的数据。

它的构造方法指定一个 `permitsPerSecond` 参数，代表每秒钟产生多少个 permits，这就是我们的速率。

### SmoothRateLimiter 介绍

RateLimiter 有一个抽象子类 `SmoothRateLimiter`，SmoothRateLimiter 有两个实现类，我们先简单介绍下 SmoothRateLimiter。

RateLimiter 作为抽象类，只有两个属性：

```java
private final SleepingStopwatch stopwatch;

private volatile Object mutexDoNotUseDirectly;
```

stopwatch 它是用来"计时"的，RateLimiter 把实例化的时候设置为 `0` 值，后续都是取相对时间，用`微秒`表示。

mutexDoNotUseDirectly 用来做锁，RateLimiter 依赖于 `synchronized` 来控制并发，所以我们之后可以看到，各个属性甚至都没有用 volatile 修饰(`锁 具有可见性 和原子性，volatile  只具有可见性`)。

然后我们来看 SmoothRateLimiter 的属性。

```java
// 当前还剩的 permits(没有被使用的）
double storedPermits;

// 最大允许缓存的 permits 数量
double maxPermits;

// 间隔多少时间产生一个 permit，
// 比如我们构造方法中设置每秒 5 个，也就是每隔 200ms 一个，这里单位是微秒，也就是 200,000
double stableIntervalMicros;

// 下一次可以获取 permits 的时间，这个时间是相对 RateLimiter 的构造时间的，是一个相对时间.
private long nextFreeTicketMicros = 0L; 
```

`nextFreeTicketMicros` 是一个很关键的属性。我们每次获取 permits 的时候，先拿 storedPermits 的值，如果够，storedPermits 减去相应的值就可以了，**如果不够，那么还需要将 nextFreeTicketMicros 往前推，表示我预占了接下来多少的时间量了（透支消费）**。那么下一个请求来的时候，如果还没到 nextFreeTicketMicros 这个时间点，需要 sleep 到这个点再返回。

因为时间是一直往前走的，应该要一直往池中添加 permits，所以 storedPermits 的值需要不断往上添加，难道需要另外开启一个线程来添加 permits？其实实际不是的，只需要在获取的操作中同步一下，重新计算就好了。

### SmoothBursty 分析

我们先从比较简单的 SmoothBursty 出发，来分析 RateLimiter 的源码，之后我们再分析 SmoothWarmingUp。

> SmoothBursty 默认缓存最多 1 秒钟的 permits，不可以修改。

#### create

RateLimiter 的静态构造方法：

```java
public static RateLimiter create(double permitsPerSecond) {
    return create(permitsPerSecond, SleepingStopwatch.createFromSystemTimer());
}
```

构造参数 permitsPerSecond 指定每秒钟可以产生多少个 permits。

```java
static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
    RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
    rateLimiter.setRate(permitsPerSecond);
    return rateLimiter;
}
```

我们看到，这里实例化的是 SmoothBursty 的实例，它的构造方法很简单，而且它只有一个属性 `maxBurstSeconds`，这里就不贴代码了。

构造函数指定了 maxBurstSeconds 为 1.0，也就是说，最多会缓存 1 秒钟，也就是 (`1.0 * permitsPerSecond`) 这么多个 permits 到池中。

> 这个 1.0 秒，关系到 storedPermits 和 maxPermits：
>
> 0 <= storedPermits <= maxPermits = permitsPerSecond

#### setRate 

我们继续往后看 setRate 方法，注意看，这里用了 synchronized 控制并发。

```java
public final void setRate(double permitsPerSecond) {
  checkArgument(
      permitsPerSecond > 0.0 && !Double.isNaN(permitsPerSecond), "rate must be positive");
  synchronized (mutex()) {
    doSetRate(permitsPerSecond, stopwatch.readMicros());
  }
}
```

`setRate` 这个方法是一个 public 方法，它可以用来**调整速率**。

```java
@Override
final void doSetRate(double permitsPerSecond, long nowMicros) {
    // 同步
    resync(nowMicros);
    // 计算属性 stableIntervalMicros
    double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
    this.stableIntervalMicros = stableIntervalMicros;
    doSetRate(permitsPerSecond, stableIntervalMicros);
}
```

#### resync

resync 方法很简单，它用来调整 storedPermits 和 nextFreeTicketMicros。这就是我们说的，在关键的节点，需要先更新一下 storedPermits 到正确的值，在计算出的newPermits 和  maxPermits 中取最小者，也就是 storedPermits  并不会累计。

```java
void resync(long nowMicros) {
  // 如果 nextFreeTicket 已经过掉了，想象一下很长时间都没有再次调用 limiter.acquire() 的场景
  // 需要将 nextFreeTicket 设置为当前时间，重新计算 storedPermits
  if (nowMicros > nextFreeTicketMicros) {
    double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
    storedPermits = min(maxPermits, storedPermits + newPermits);
    nextFreeTicketMicros = nowMicros;
  }
}
```

> coolDownIntervalMicros() 这个方法大家先不用关注，可以看到，在 SmoothBursty 类中的实现是直接返回了 stableIntervalMicros 的值，也就是我们说的，每产生一个 permit 的时间长度。
>

我们回到前面一个方法，resync 同步以后，会设置 stableIntervalMicros 为一个正确的值，然后进入下面的方法：

```java
@Override
void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
  double oldMaxPermits = this.maxPermits;
  // 这里计算了，maxPermits 为 1 秒产生的 permits
  maxPermits = maxBurstSeconds * permitsPerSecond;
  if (oldMaxPermits == Double.POSITIVE_INFINITY) {
    // if we don't special-case this, we would get storedPermits == NaN, below
    storedPermits = maxPermits;
  } else {
    // 因为 storedPermits 的值域变化了，需要等比例缩放
    storedPermits =
        (oldMaxPermits == 0.0)
            ? 0.0 // initial state
            : storedPermits * maxPermits / oldMaxPermits;
  }
}
```

上面这个方法，我们要这么看，原来的 RateLimiter 是用某个 permitsPerSecond 值初始化的，现在我们要调整这个频率。对于 maxPermits 来说，是重新计算，而对于 storedPermits 来说，是做等比例的缩放。

到此，构造方法就完成了，我们得到了一个 RateLimiter 的实现类 SmoothBursty 的实例，可能上面的源码你还是会有一些疑惑，不过也没关系，继续往下看，可能你的很多疑惑就解开了。

#### acquire

接下来，我们来看看 acquire 方法：

```java
@CanIgnoreReturnValue
public double acquire() {
  return acquire(1);
}

@CanIgnoreReturnValue
public double acquire(int permits) {
  // 如果当前不能直接获取到 permits（permists 不够了），则需要等待
  // 返回值代表需要 sleep 多久
  long microsToWait = reserve(permits);
  // sleep
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  // 返回 sleep 的时长
  return 1.0 * microsToWait / SECONDS.toMicros(1L);
}
```

我们来看 reserve 方法：

```java
final long reserve(int permits) {
  checkPermits(permits);
  // 由于涉及并发操作，所以使用synchronized进行并发操作  
  synchronized (mutex()) {
    return reserveAndGetWaitLength(permits, stopwatch.readMicros());
  }
}

final long reserveAndGetWaitLength(int permits, long nowMicros) {
  // 计算从当前时间开始，能够获取到目标数量令牌时的时间
  long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
  // 两个时间相减，获得需要等待的时间
  return max(momentAvailable - nowMicros, 0);
}
```

继续往里看：

```java
@Override
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
  // 刷新令牌数，相当于每次acquire时在根据时间进行令牌的刷新
  resync(nowMicros);
  // 获取当前已有的令牌数和需要获取的目标令牌数进行比较，计算出可以目前即可得到的令牌数。
  long returnValue = nextFreeTicketMicros;
  // storedPermits 中可以使用多少个 permits
  double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
  // storedPermits 中不够的部分
  double freshPermits = requiredPermits - storedPermitsToSpend;
  // 为了这个不够的部分，需要等待多久时间
  long waitMicros =
      storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend) // 这部分固定返回 0
          + (long) (freshPermits * stableIntervalMicros);
  // 将 nextFreeTicketMicros 往前推
  this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
  // storedPermits 减去被拿走的部分
  this.storedPermits -= storedPermitsToSpend;
  // 返回旧的nextFreeTicketMicros数值，无需为预支付的令牌多加等待时间。  
  return returnValue;
}
```

我们可以看到，获取 permits 的时候，其实是获取了两部分，一部分来自于存量 storedPermits，存量不够的话，另一部分来自于预占未来的 freshPermits，并且 无需为预支付的permits 等待。

这里提一个关键点吧，我们看到，返回值是 nextFreeTicketMicros 的旧值，因为只要到这个时间点，就说明当次 acquire 可以成功返回了，而不管 storedPermits 够不够。如果 storedPermits 不够，会将 nextFreeTicketMicros 往前推一定的时间，预占了一定的量。

#### 小结

下面 我们来简单总结一下 前面acquire  的流程：

1. 尝试`resync`操作（尝试调整 storedPermits 和 nextFreeTicketMicros）
2. 尝试 刷新storedPermits 和 nextFreeTicketMicros
3. 计算 实际消耗的Permits：min(请求Permits,存储Permits中的小值) 和 nextFreeTicketMicros
4. 计算 需要 等待的时间，如果 需要等待，则进行等待
5. 返回 最终 等待的时间。

到这里，acquire 方法就分析完了，SmoothBursty 的源码还是非常简单的。

### SmoothWarmingUp 分析

`RateLimiter`的 `SmoothWarmingUp`是带有预热期的平滑限流，它启动后会有一段预热期，逐步将分发频率提升到配置的速率，适用于资源需要预热的场景。

假设我们的业务在稳定状态下，正常可以提供最大 1000 QPS 的访问，但是如果连接池不是很活跃的（可能活跃的连接只有几百个），我们就不能马上让系统达到 1000 个 QPS，要限制住突发流量，因为这会拖垮我们的系统，我们应该有个预热升温的过程。

对应到 SmoothWarmingUp 中，如果系统处于低负载状态，storedPermits 会一直增加，当请求来的时候，我们要从 storedPermits 中取 permits，最关键的点在于，从 storedPermits 中取 permits 的操作是比较耗时的，因为没有预热。

大家先有一些粗的概念，然后我们来看下面这个图：

![smooth-warm-up](http://img.blog.ztgreat.cn/document/algorithm/smooth-warm-up.png)





有几个变量需要说明：

- stableInterval 这个跟之前的文章描述的有点不同，这里考虑是放行令牌桶令牌的速率，类似于漏桶算法
- coldInterval 这个是在冷却情况下发放令牌的速率，目前取值为：`coldInterval = coldFactor * stableInterval`
- thresholdPermits 当令牌桶中存储的令牌数目到达阈值时，发放令牌的速率到达稳定
- maxPermits 令牌桶中存放令牌的最大数目
- warmUpPeriod 预热的时间长度，在这段预热期内令牌桶的存放数目从maxPermits减少到了thresholdPermits



X 轴代表 storedPermits 的数量，Y 轴代表获取一个 permits 需要的时间。

> 假设指定 permitsPerSecond 为 10，那么 stableInterval 为 100ms，而 coldInterval 是 3 倍，也就是 300ms（coldFactor，3 倍是写死的，用户不能修改）。也就是说，当达到 maxPermits 时，此时处于系统最冷的时候，获取一个 permit 需要 300ms，而如果 storedPermits 小于 thresholdPermits 的时候，只需要 100ms。
>

想象有一条垂直线 x=k，它与 X 轴的交点 k 代表当前 storedPermits 的数量：

- 当系统在非常繁忙的时候，这条线停留在 x=0 处，此时 storedPermits 为 0
- 当 limiter 没有被使用的时候，这条线慢慢往右移动，直到 x=maxPermits 处；
- 如果 limiter 被重新使用，那么这条线又慢慢往左移动，直到 x=0 处；

当 【thresholdPermits <= **storedPermits** <= maxPermits】 状态时，我们认为 limiter 中的 permits 是冷的，此时获取一个 permit 需要较多的时间，因为需要预热，有一个关键的分界点是 thresholdPermits。

预热时间是我们在构造的时候指定的，图中梯形的面积就是预热时间，因为预热完成后，我们能进入到一个稳定的速率中（stableInterval），下面我们来根据构造参数计算出 thresholdPermits 和 maxPermits 的值。

有一个关键点，从 thresholdPermits 到 0 的时间，是从 maxPermits 到 thresholdPermits 时间的一半，也就是梯形的面积是长方形面积的 2 倍，梯形的面积是 warmupPeriod。

![12034897227](http://img.blog.ztgreat.cn/document/algorithm/12034897227.png)

> 之所以长方形的面积是 warmupPeriod/2，是因为 coldFactor 是硬编码的 **3**，这个将在下面的代码中体现
>
> // coldFactor 是固定的 3
>  double coldIntervalMicros = stableIntervalMicros * coldFactor;

梯形面积为 warmupPeriod，即：

```java
warmupPeriod = 2 * stableInterval * thresholdPermits
```

由此，我们得出 **thresholdPermits** 的值：

```java
thresholdPermits = 0.5 * warmupPeriod / stableInterval
```

然后我们根据梯形面积的计算公式：

```java
warmupPeriod = 0.5 * (stableInterval + coldInterval) * (maxPermits - thresholdPermits)
```

得出 **maxPermits** 为：

```java
maxPermits = thresholdPermits + 2.0 * warmupPeriod / (stableInterval + coldInterval)
```

这样，我们就得到了 thresholdPermits 和 maxPermits 的值。

接下来，我们来看一下冷却时间间隔，它指的是 storedPermits 中每个 permit 的增长速度，也就是我们前面说的 x=k 这条垂直线往右的移动速度，为了达到从 0 到 maxPermits 花费 warmupPeriodMicros 的时间，我们将其定义为：

```java
@Override
double coolDownIntervalMicros() {
    return warmupPeriodMicros / maxPermits;
}
```
贴一下代码，大家就知道了，在 resync 中用到的这个：
```java
void resync(long nowMicros) {
  if (nowMicros > nextFreeTicketMicros) {
    // coolDownIntervalMicros 在这里使用
    double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
    storedPermits = min(maxPermits, storedPermits + newPermits);
    nextFreeTicketMicros = nowMicros;
  }
}
```

基于上面的分析，我们来看 SmoothWarmingUp 的其他源码。

首先，我们来看它的 doSetRate 方法，有了前面的介绍，这个方法的源码非常简单：

```java
@Override
void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
    double oldMaxPermits = maxPermits;
    // coldFactor 是固定的 3
    double coldIntervalMicros = stableIntervalMicros * coldFactor;
    // 这个公式我们上面已经说了
    thresholdPermits = 0.5 * warmupPeriodMicros / stableIntervalMicros;
    // 这个公式我们上面也已经说了
    maxPermits =
        thresholdPermits + 2.0 * warmupPeriodMicros / (stableIntervalMicros + coldIntervalMicros);
    // 计算那条斜线的斜率。数学知识，对边 / 临边
    slope = (coldIntervalMicros - stableIntervalMicros) / (maxPermits - thresholdPermits);
    if (oldMaxPermits == Double.POSITIVE_INFINITY) {
        // if we don't special-case this, we would get storedPermits == NaN, below
        storedPermits = 0.0;
    } else {
        storedPermits =
            (oldMaxPermits == 0.0)
                ? maxPermits // initial state is cold
                : storedPermits * maxPermits / oldMaxPermits;
    }
}
```

setRate 方法非常简单，接下来，我们要分析的是 storedPermitsToWaitTime 方法，我们回顾一下前面的代码：

```
@Override
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
  // 刷新令牌数，相当于每次acquire时在根据时间进行令牌的刷新
  resync(nowMicros);
  // 获取当前已有的令牌数和需要获取的目标令牌数进行比较，计算出可以目前即可得到的令牌数。
  long returnValue = nextFreeTicketMicros;
  // storedPermits 中可以使用多少个 permits
  double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
  // storedPermits 中不够的部分
  double freshPermits = requiredPermits - storedPermitsToSpend;
  // 为了这个不够的部分，需要等待多久时间
  long waitMicros =
      storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend) // 这部分固定返回 0
          + (long) (freshPermits * stableIntervalMicros);
  // 将 nextFreeTicketMicros 往前推
  this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
  // storedPermits 减去被拿走的部分
  this.storedPermits -= storedPermitsToSpend;
  // 返回旧的nextFreeTicketMicros数值，无需为预支付的令牌多加等待时间。  
  return returnValue;
}
```



这段代码是 acquire 方法的核心，waitMicros 由两部分组成，一部分是从 storedPermits 中获取花费的时间，一部分是等待 freshPermits 产生花费的时间。在 SmoothBursty 的实现中，从 storedPermits 中获取 permits 直接返回 0，不需要等待。

而在 SmoothWarmingUp 的实现中，由于需要预热，所以从 storedPermits 中取 permits 需要花费一定的时间，其实就是要计算下图中，阴影部分的面积。

![12034896227](http://img.blog.ztgreat.cn/document/algorithm/12034896227.png)

```java
@Override
long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {
  double availablePermitsAboveThreshold = storedPermits - thresholdPermits;
  long micros = 0;
  // 如果右边梯形部分有 permits，那么先从右边部分获取permits，计算梯形部分的阴影部分的面积
  if (availablePermitsAboveThreshold > 0.0) {
    // 从右边部分获取的 permits 数量
    double permitsAboveThresholdToTake = min(availablePermitsAboveThreshold, permitsToTake);
    // 梯形面积公式：(上底+下底)*高/2
    double length =
        permitsToTime(availablePermitsAboveThreshold)
            + permitsToTime(availablePermitsAboveThreshold - permitsAboveThresholdToTake);
    micros = (long) (permitsAboveThresholdToTake * length / 2.0);
    permitsToTake -= permitsAboveThresholdToTake;
  }
  // 加上 长方形部分的阴影面积
  micros += (long) (stableIntervalMicros * permitsToTake);
  return micros;
}

// 对于给定的 x 值，计算 y 值
private double permitsToTime(double permits) {
  return stableIntervalMicros + permits * slope;
}
```

到这里，SmoothWarmingUp 也已经说完了。

[Sentinel](https://github.com/alibaba/Sentinel) 是阿里开源的流控、熔断工具，目前 在项目中 我们也在简单使用，但是了解 还不深，后面再学习学习。 


