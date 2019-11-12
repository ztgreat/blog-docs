**并发这块反复研究挺久了，源码也研究了一部分，总觉得缺乏一点认识，想了许久还是总结出来，这样也有助于自己理解**

**原子操作意思为不可被中断的一个或一系列操作**

在了解原子操作的实现原理前，先要了解一下相关的术语

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170725133015014.png)

 - 注:图片来自Java并发编程的艺术

## 一、处理器保证复杂内存操作的原子性


### 使用总线锁保证原子性
​        所谓总线锁就是使用处理器提供的一个LOCK#信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞，那么该处理器可以独占共享内存。

### 使用缓存锁保证原子性
​        在同一时刻，我们只需要保证对某个内存地址的操作是原子性即可，频繁使用的内存会缓存到处理器的L1,L2和L3高速缓存里，那么原子操作就可以直接在处理器内部缓存中进行，并不需要声明总线锁。所谓“缓存锁定”是指内存区域如果被缓存在处理器的缓存行中，并且在Lock操作期间被锁定（锁定缓存行不锁定总线），那么当它执行锁操作回写到内存时，处理器不在总线上发出LOCK#信号，而是修改内部的内存地址，并允许它的缓存一致性机制来保证操作的原子性，因为缓存一致性机制会阻止同时修改由两个以上处理器缓存的内存区域数据，当其他处理器回写被锁定的缓存行的数据时，会使缓存行无效。
## 二、Java如何实现原子操作

​        CAS，Compare And Swap，即比较并交换，整个AQS同步组件、Atomic原子类操作等等都是以CAS实现的。
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170725140450740.png)

我们学习学习路线也会从下至上的学习。
​        Java无法直接访问底层操作系统，而是通过本地（native）方法来访问。不过，JDK中有一个类Unsafe，它提供了硬件级别的原子操作，例如获取数据的内存地址偏移等等，所以这个类对开发人员来说是不安全的。而这个CAS就在这个Unsafe中。
到这里只需要知道java底层通过Unsafe这个工具类来完成原子操作，UnSafe中大部分方法都是本地方法，与操作系统关系密切。

JVM中的CAS操作正是利用了**处理器提供的指令**来实现的（处理器级别，不是依靠软件），其CAS实现的基本思路就是循环进行比较，如果**符合条件就设置新的值**，设置不成功再尝试，操作直到成功为止，其循环的过程也叫自旋。

**CAS有三个操作数：内存值V、旧的预期值A、要修改的值B，当且仅当预期值A和内存值V相同时（条件），将内存值修改为B并返回true，否则条件不符合返回false。**条件不符合说明该变量已经被其它线程更新了。



### AtomicInteger源码分析

​       java.util.concurrent.atomic包下的原子操作类都是基于CAS实现的，接下去我们通过AtomicInteger来看看是如何通过CAS实现原子操作的：

```
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    //获得unsafe实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    //存放变量value的内存偏移
    private static final long valueOffset;

    static {
        try {
            //通过unsafe获得value的内存偏移
            valueOffset = unsafe.objectFieldOffset
                    (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    //volatile修饰的，保证了多线程之间看到的value值是同一份,后面会分析
    private volatile int value;
    
    //省略后续代码
}
```

接下来看看设置新的值是如何完成的：

```
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

这里很简单，直接调用unsafe的cas方法就可以了，通过value的内存偏移，以及期望的值来设置新的值。接下来就是本地方法调用了。

```
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
```

同样原子增加的操作就是通过unsafe来完成，只是每次都是增加1而已。
可以去看看其他几个原子操作类：AtomicLong（原子更新长整型）AtomicBoolean（原子更新布尔类型，用int来实现的），AtomicIntegerArray(原子更新整型数组里的元素)，AtomicReference（原子更新引用类型）等，其核心思想都是用unsafe提供的原子操作来完成的。
在**Java并发编程**的艺术中，作者实现了一个基于CAS线程安全的计数器和一个非线程安全的计数器，本质就是用原子操作类代替一般的int或者Long数据类型，通过原子操作类来完成原子操作，保证了计数的线程安全，但是这里提一下，**原子操作类不是什么时候都是线程安全的，当由原子操作类来完成复合操作是，此时就不一定是线程安全的了**。

```
AtomicInteger a=new AtomicInteger(10);
//假设下面会出现线程竞争
int b=a.get();
    if (b==10){
    a.compareAndSet(10,100);
}
```

在多线程中，这样就不是线程安全的了，因为先取出某个值，然后在判断，这整个操作不是原子操作。


## CAS实现原子操作的三大问题

### ABA问题
​        因为CAS需要在操作值的时候，检查值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A,变成了B,又变成了A,那么使用CAS进行检查时会发现它的值没有发生变化，但实际上却变化了。ABA问题的解决思路就是使用版本号，在变量前面追加版本号，这样每次变量更新的时候把版本号也进行改变，这样这算值相同，那么也可以通过版本号区分。JDK提供了一个类**AtomicStampedReference**来解决ABA问题，这个类的compareAndSet方法的作用是首先检查当前引用是否等于预期引用，并检查当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。



```
public class AtomicStampedReference<V> {

    private static class Pair<T> {
        final T reference;  //值
        final int stamp;   //标志

        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }

        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<T>(reference, stamp);
        }
    }

    private volatile Pair<V> pair;
    //省略后续代码
}
```

内部使用了pair来包裹值和标志。

```
public boolean compareAndSet(V   expectedReference,  //预期值
                             V   newReference,   //新值
                             int expectedStamp,  //预期标志
                             int newStamp) {    //新标志
    Pair<V> current = pair;
    return
            expectedReference == current.reference &&
                    expectedStamp == current.stamp &&
                    ((newReference == current.reference &&
                            newStamp == current.stamp) ||
                            casPair(current, Pair.of(newReference, newStamp)));
}
```

### 循环时间长开销大
​        回过头来想想，为什么要使用CAS,难道不可以直接用锁之类的嘛，用锁实现肯定是可以的 ，但是锁的代价太大，需要上下文切换，如果执行代码很短，但是切换频繁，那么cpu的有效利用率其实很低，用CAS就是用自旋来代替上下文切换，因为很有可能这次不成功，下次检查的时候就成功了，当然如果一些自旋，那么消耗的cpu也比锁代价大了，得不偿失，所有后面我们会看到有些基于CAS实现的会控制自旋的次数，在要求次数内如果还没成功，则切换阻塞或者进行其他操作。

 - **只能保证一个共享变量的原子操作**
    如果需要对多个共享变量操作时，循环CAS就无法保证整体操作的原子性了。

**不知道描述清楚了没有，基本先这样认识一下，欢迎拍砖**

 ## 参考
java 并发编程的艺术



