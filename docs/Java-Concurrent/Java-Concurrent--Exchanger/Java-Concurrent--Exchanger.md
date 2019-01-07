### Exchanger 介绍（jdk 1.8）
前面分别介绍了CyclicBarrier、CountDownLatch、Semaphore，现在介绍并发工具类中的最后一个Exchange。

Exchanger 是一个用于线程间协作的工具类，Exchanger用于进行线程间的数据交换，它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。这两个线程通过exchange 方法交换数据，如果第一个线程先执行exchange 方法，它会一直等待第二个线程也执行exchange 方法，当两个线程都到达同步点时，这两个线程就可以交换数据。

### 使用
Exchanger 使用是非常简单的，但是实现原理和前面几种工具比较确实最难的，前面几种工具都是通过同步器或者锁来实现，而Exchanger 是一种无锁算法，和前面SynchronousQueue 一样，都是通过循环 cas 来实现线程安全，因此这种方式就会显得比较抽象和麻烦。
```
public class ExchangerDemo {

    static Exchanger<String>exchanger=new Exchanger<String>();
    static class Task implements Runnable{
        @Override
        public void run() {
            try {
                String result=exchanger.exchange(Thread.currentThread().getName());
                System.out.println("this is "+Thread.currentThread().getName()+" receive data:"+result);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args)throws  Exception{

        Thread t1=new Thread(new Task(),"thread1");
        Thread t2=new Thread(new Task(),"thread2");
        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171205191544015.png)
### 单槽 Exchanger
Exchanger 有单槽位和多槽位之分，单个槽位在同一时刻只能用于两个线程交换数据，这样在竞争比较激烈的时候，会影响到性能，多个槽位就是多个线程可以同时进行两个的数据交换，彼此之间不受影响，这样可以很好的提高吞吐量。
单槽 Exchanger相对要简单许多，我们就先从这里开始分析吧。
#### 数据结构
槽位定义：
```
@sun.misc.Contended static final class Node {
    int index;              //arena的下标，多个槽位的时候利用
    int bound;              // 上一次记录的Exchanger.bound；
    int collides;           // 在当前bound下CAS失败的次数；
    int hash;               // 用于自旋；
    Object item;            // 这个线程的当前项，也就是需要交换的数据；
    volatile Object match;  // 交换的数据
    volatile Thread parked; // 线程
}
/**
 * Value representing null arguments/returns from public
 * methods. Needed because the API originally didn't disallow null
 * arguments, which it should have.
 * 如果交换的数据为 null,则用NULL_ITEM  代替
 */
private static final Object NULL_ITEM = new Object();
```
Node 定义中，index，bound，collides 这些都是用于多槽位的，这些可以暂时不用管，item 是本线程需要交换的数据，match 是和其它线程交换后的数据，开始是为null,交换数据成功后，就是我们需要的数据了，parked记录线程，用于阻塞和唤醒线程。

```
/** The number of CPUs, for sizing and spin control */
private static final int NCPU = Runtime.getRuntime().availableProcessors();
/**
 * The bound for spins while waiting for a match. The actual
 * number of iterations will on average be about twice this value
 * due to randomization. Note: Spinning is disabled when NCPU==1.
 */
private static final int SPINS = 1 << 10; // 自旋次数
/**
 * Slot used until contention detected.
 */
private volatile Node slot; // 用于交换数据的槽位
```
Node是每个线程自身用于数据交换的节点，相当于每个Node就代表了每个线程，为了保证线程安全，把线程的Node 节点 放在哪里呢，当然是ThreadLocal咯。

```
/**
 * Per-thread state  每个线程的数据，ThreadLocal 子类
 */
private final Participant participant;

/** The corresponding thread local class */
 static final class Participant extends ThreadLocal<Node> {
     // 初始值返回Node
     public Node initialValue() { return new Node(); }
 }

```
单个槽位需要准备的知识就这么多，具体部分字段的理解，在代码中再来细看。

#### exchange 方法
1、没有设定超时时间的exchange 方法

```
public V exchange(V x) throws InterruptedException {
    Object v;
    Object item = (x == null) ? NULL_ITEM : x; // translate null args
    if ((arena != null ||
         (v = slotExchange(item, false, 0L)) == null) &&
        ((Thread.interrupted() || // disambiguates null return
          (v = arenaExchange(item, false, 0L)) == null)))
        throw new InterruptedException();
    return (v == NULL_ITEM) ? null : (V)v;
}
```
2、具有超时功能的exchange 方法
```
public V exchange(V x, long timeout, TimeUnit unit)
    throws InterruptedException, TimeoutException {
    Object v;
    Object item = (x == null) ? NULL_ITEM : x;
    long ns = unit.toNanos(timeout);
    if ((arena != null ||
         (v = slotExchange(item, true, ns)) == null) &&
        ((Thread.interrupted() ||
          (v = arenaExchange(item, true, ns)) == null)))
        throw new InterruptedException();
    if (v == TIMED_OUT)
        throw new TimeoutException();
    return (v == NULL_ITEM) ? null : (V)v;
}
```
exchange 把 执行 单槽位交换还是多槽位交换，同时如果发生中断，则返回前会重设中断标志位  这几个操作通过一个语句来实现，因此看的时候可能需要仔细一点。
两个方法，主要的不同还是在于是否有超时时间设置，如果有超时时间设置，那么如果在指定的时间内没有交换到数据，那么就会返回（抛出超时异常），不会一直等待。
接下来我们就来看单槽位交换的核心方法slotExchange。

#### slotExchange 方法
```
private final Object slotExchange(Object item, boolean timed, long ns) {
    // 得到一个初试的Node
    Node p = participant.get();
    // 当前线程
    Thread t = Thread.currentThread();
    // 如果发生中断，返回null,会重设中断标志位，并没有直接抛异常
    if (t.isInterrupted()) // preserve interrupt status so caller can recheck
        return null;
    
    for (Node q;;) {
        // 槽位 solt不为null,则说明已经有线程在这里等待交换数据了
        if ((q = slot) != null) {
            // 重置槽位
            if (U.compareAndSwapObject(this, SLOT, q, null)) {
                //获取交换的数据
                Object v = q.item;
                //等待线程需要的数据
                q.match = item;
                //等待线程
                Thread w = q.parked;
                //唤醒等待的线程
                if (w != null)
                    U.unpark(w);
                return v; // 返回拿到的数据，交换完成
            }
            // create arena on contention, but continue until slot null
            //存在竞争，其它线程抢先了一步该线程，因此需要采用多槽位模式，这个后面再分析
            if (NCPU > 1 && bound == 0 &&
                U.compareAndSwapInt(this, BOUND, 0, SEQ))
                arena = new Node[(FULL + 2) << ASHIFT];
        }
        else if (arena != null) //多槽位不为空，需要执行多槽位交换
            return null; // caller must reroute to arenaExchange
        else { //还没有其他线程来占据槽位
            p.item = item;
            // 设置槽位为p(也就是槽位被当前线程占据)
            if (U.compareAndSwapObject(this, SLOT, null, p))
                break; // 退出无限循环
            p.item = null; // 如果设置槽位失败，则有可能其他线程抢先了，重置item,重新循环
        }
    }

    //当前线程占据槽位，等待其它线程来交换数据
    int h = p.hash;
    long end = timed ? System.nanoTime() + ns : 0L;
    int spins = (NCPU > 1) ? SPINS : 1;
    Object v;
    // 直到成功交换到数据
    while ((v = p.match) == null) {
        if (spins > 0) { // 自旋
            h ^= h << 1; h ^= h >>> 3; h ^= h << 10;
            if (h == 0)
                h = SPINS | (int)t.getId();
            else if (h < 0 && (--spins & ((SPINS >>> 1) - 1)) == 0)
                // 主动让出cpu,这样可以提供cpu利用率（反正当前线程也自旋等待，还不如让其它任务占用cpu）
                Thread.yield(); 
        }
        else if (slot != p) //其它线程来交换数据了，修改了solt,但是还没有设置match,再稍等一会
            spins = SPINS;
        //需要阻塞等待其它线程来交换数据
        //没发生中断，并且是单槽交换，没有设置超时或者超时时间未到 则继续执行
        else if (!t.isInterrupted() && arena == null &&
                 (!timed || (ns = end - System.nanoTime()) > 0L)) {
            // cas 设置BLOCKER，可以参考Thread 中的parkBlocker
            U.putObject(t, BLOCKER, this);
            // 需要挂起当前线程
            p.parked = t;
            if (slot == p)
                U.park(false, ns); // 阻塞当前线程
            // 被唤醒后    
            p.parked = null;
			// 清空 BLOCKER
            U.putObject(t, BLOCKER, null);
        }
        // 不满足前面 else if 条件，交换失败，需要重置solt
        else if (U.compareAndSwapObject(this, SLOT, p, null)) {
            v = timed && ns <= 0L && !t.isInterrupted() ? TIMED_OUT : null;
            break;
        }
    }
    //清空match
    U.putOrderedObject(p, MATCH, null);
    p.item = null;
    p.hash = h;
    // 返回交换得到的数据（失败则为null）
    return v;
}
```
理解slotExchange 应该不难吧，相比前面的SynchronousQueue  之类的应该要简单多了，还是再来简述一遍：
当一个线程来交换数据时，如果发现槽位（solt）有数据时，说明其它线程已经占据了槽位，等待交换数据，那么当前线程就和该槽位进行数据交换，设置相应字段，如果交换失败，则说明其它线程抢先了该线程一步和槽位交换了数据，那么这个时候就存在竞争了，这个时候就会生成多槽位（area）,后面就会进行多槽位交换了。
如果来交换的线程发现槽位没有被占据，啊哈，这个时候自己就把槽位占据了，如果占据失败，则有可能其他线程抢先了占据了槽位，重头开始循环。
当来交换的线程占据了槽位后，就需要等待其它线程来进行交换数据了，首先自己需要进行一定时间的自旋，因为自旋期间有可能其它线程就来了，那么这个时候就可以进行数据交换工作，而不用阻塞等待了，如果不幸，进行了一定自旋后，没有其他线程到来，那么还是避免不了需要阻塞（如果设置了超时等待，发生了超时或中断异常，则退出，不阻塞等待），当准备阻塞线程的时候，发现槽位值变了，那么说明其它线程来交换数据了，但是还没有完全准备好数据，这个时候就不阻塞了，再稍微等那么一会，如果始终没有等到其它线程来交换，那么就挂起当前线程。
当其它线程到来并成功交换数据后，会唤醒被阻塞的线程，阻塞的线程被唤醒后，拿到数据（如果是超时，或中断，则数据为null）返回，结束。

单槽位的交换就结束了，整个过程应该不难，如果竞争激烈，那么一个槽位显然就成了性能瓶颈了，因此就衍生出了多槽位交换，各自交换各自的，互不影响。

### 多槽 Exchanger
多槽位呢，实际就是一个Node 数组，代表了很多的槽位，对于多槽位，我个人对代码的部分理解得不是完全透彻，有些地方有点不知其所以然，在前面分析SynchronousQueue  的时候也曾遇到，但是SynchronousQueue 调试方便，跟几遍就可以很好的理解了，但是这个多槽位有点不好调试，而且一两个线程观察不出来，因此有些地方可能会说不清楚，或者有误，因此如果朋友发现有问题的，欢迎指出来，共同学习。
#### 数据结构
```
@sun.misc.Contended static final class Node {
    int index;              //arena的下标，多个槽位的时候利用
    int bound;              // 上一次记录的Exchanger.bound；
    int collides;           // 在当前bound下CAS失败的次数；
    int hash;               // 用于自旋；
    Object item;            // 这个线程的当前项，也就是需要交换的数据；
    volatile Object match;  // 交换的数据
    volatile Thread parked; // 线程
}

/**
 * The byte distance (as a shift value) between any two used slots
 * in the arena.  1 << ASHIFT should be at least cacheline size.
 * CacheLine填充
 */
private static final int ASHIFT = 7;

/**
 * The maximum supported arena index. The maximum allocatable
 * arena size is MMASK + 1. Must be a power of two minus one, less
 * than (1<<(31-ASHIFT)). The cap of 255 (0xff) more than suffices
 * for the expected scaling limits of the main algorithms.
 */
private static final int MMASK = 0xff;

/**
 * Unit for sequence/version bits of bound field. Each successful
 * change to the bound also adds SEQ.
 * bound的"版本号"
 */
private static final int SEQ = MMASK + 1;

/**
 * The maximum slot index of the arena: The number of slots that
 * can in principle hold all threads without contention, or at
 * most the maximum indexable value.
 */
static final int FULL = (NCPU >= (MMASK << 1)) ? MMASK : NCPU >>> 1;

/**
 * Elimination array; null until enabled (within slotExchange).
 * Element accesses use emulation of volatile gets and CAS.
 */
private volatile Node[] arena;

/**
 * The index of the largest valid arena position, OR'ed with SEQ
 * number in high bits, incremented on each update.  The initial
 * update from 0 to SEQ is used to ensure that the arena array is
 * constructed only once.
 */
private volatile int bound;
```
为了不误导大家，重要的属性保留了源码的注释，这样有助于理解。
在Node 前面有个@sun.misc.Contended 的注解，这个是什么呢，这个是用来避免伪共享的（当然不仅仅是加个注解就ok了），下面的伪共享说明摘自 [大飞](http://brokendreams.iteye.com/blog/2253956)
伪共享说明：假设一个类的两个相互独立的属性a和b在内存地址上是连续的(比如FIFO队列的头尾指针)，那么它们通常会被加载到相同的cpu cache line里面。并发情况下，如果一个线程修改了a，会导致整个cache line失效(包括b)，这时另一个线程来读b，就需要从内存里再次加载了，这种多线程频繁修改ab的情况下，虽然a和b看似独立，但它们会互相干扰，非常影响性能。

ASHIFT 这个字段就是用于避免伪共享的，1 << ASHIFT 可以避免两个Node在同一个共享区，这个可以从后面代码再来具体分析。

在slotExchange 当存在竞争时，会构建area,现在我们再来回顾这个代码：
```
if (NCPU > 1 && bound == 0 &&U.compareAndSwapInt(this, BOUND, 0, SEQ))
      arena = new Node[(FULL + 2) << ASHIFT];
```
初始化arena 时会设置bound为SEQ(SEQ=MMASK + 1)。MMASK  值为255，二进制：8个1
arena的大小为(FULL + 2) << ASHIFT，因为1 << ASHIFT 是用于避免伪共享的，因此实际有效的Node 只有FULL + 2 个，这个我们从后面的代码也可以得出。
#### arenaExchange 方法
多槽的交换大致思想就是：当一个线程来交换的时候，如果"第一个"槽位是空的，那么自己就在那里等待，如果发现"第一个"槽位有等待线程，那么就直接交换，如果交换失败，说明其它线程在进行交换，那么就往后挪一个槽位，如果有数据就交换，没数据就等一会，但是不会阻塞在这里，在这里等了一会，发现还没有其它线程来交换数据，那么就往“第一个”槽位的方向挪，如果反复这样过后，挪到了第一个槽位，没有线程来交换数据了，那么自己就在"第一个"槽位阻塞等待。
简单来说，如果有竞争冲突，那么就寻找后面的槽位，在后面的槽位等待一定时间，没有线程来交换，那么就又往前挪。
```
private final Object arenaExchange(Object item, boolean timed, long ns) {
    // 槽位数组
    Node[] a = arena;
    //代表当前线程的Node
    Node p = participant.get(); // p.index 初始值为 0
    for (int i = p.index;;) {                      // access slot at i
        int b, m, c; long j;                       // j is raw array offset
        //在槽位数组中根据"索引" i 取出数据 j相当于是 "第一个"槽位
        Node q = (Node)U.getObjectVolatile(a, j = (i << ASHIFT) + ABASE);
        // 该位置上有数据(即有线程在这里等待交换数据)
        if (q != null && U.compareAndSwapObject(a, j, q, null)) {
            // 进行数据交换，这里和单槽位的交换是一样的
            Object v = q.item;                     // release
            q.match = item;
            Thread w = q.parked;
            if (w != null)
                U.unpark(w);
            return v;
        }
        // bound 是最大的有效的 位置，和MMASK相与，得到真正的存储数据的索引最大值
        else if (i <= (m = (b = bound) & MMASK) && q == null) {
	        // i 在这个范围内，该槽位也为空
	        
	        //将需要交换的数据 设置给p
            p.item = item;                         // offer
			//设置该槽位数据(在该槽位等待其它线程来交换数据)
            if (U.compareAndSwapObject(a, j, null, p)) {
                long end = (timed && m == 0) ? System.nanoTime() + ns : 0L;
                Thread t = Thread.currentThread(); // wait
                // 进行一定时间的自旋
                for (int h = p.hash, spins = SPINS;;) {
                    Object v = p.match;
                    //在自旋的过程中，有线程来和该线程交换数据
                    if (v != null) {
	                    //交换数据后，清空部分设置，返回交换得到的数据，over
                        U.putOrderedObject(p, MATCH, null);
                        p.item = null;             // clear for next use
                        p.hash = h;
                        return v;
                    }
                    else if (spins > 0) {
                        h ^= h << 1; h ^= h >>> 3; h ^= h << 10; // xorshift
                        if (h == 0)                // initialize hash
                            h = SPINS | (int)t.getId();
                        else if (h < 0 &&          // approx 50% true
                                 (--spins & ((SPINS >>> 1) - 1)) == 0)
                            Thread.yield();        // two yields per wait
                    }
                    // 交换数据的线程到来，但是还没有设置好match，再稍等一会
                    else if (U.getObjectVolatile(a, j) != p)
                        spins = SPINS; 
                    //符合条件，特别注意m==0 这个说明已经到达area 中最小的存储数据槽位了
                    //没有其他线程在槽位等待了，所有当前线程需要阻塞在这里     
                    else if (!t.isInterrupted() && m == 0 &&
                             (!timed ||
                              (ns = end - System.nanoTime()) > 0L)) {
                        U.putObject(t, BLOCKER, this); // emulate LockSupport
                        p.parked = t;              // minimize window
                        // 再次检查槽位，看看在阻塞前，有没有线程来交换数据
                        if (U.getObjectVolatile(a, j) == p) 
                            U.park(false, ns); // 挂起
                        p.parked = null;
                        U.putObject(t, BLOCKER, null);
                    }
                    // 当前这个槽位一直没有线程来交换数据，准备换个槽位试试
                    else if (U.getObjectVolatile(a, j) == p &&
                             U.compareAndSwapObject(a, j, p, null)) {
                        //更新bound
                        if (m != 0)                // try to shrink
                            U.compareAndSwapInt(this, BOUND, b, b + SEQ - 1);
                        p.item = null;
                        p.hash = h;
                        // 减小索引值 往"第一个"槽位的方向挪动
                        i = p.index >>>= 1;        // descend
                        // 发送中断，返回null
                        if (Thread.interrupted())
                            return null;
                        // 超时
                        if (timed && m == 0 && ns <= 0L)
                            return TIMED_OUT;
                        break;                     // expired; restart 继续主循环
                    }
                }
            }
            else
                //占据槽位失败，先清空item,防止成功交换数据后，p.item还引用着item
                p.item = null;                     // clear offer
        }
        else { // i 不在有效范围，或者被其它线程抢先了
            //更新p.bound
            if (p.bound != b) {                    // stale; reset
                p.bound = b;
                //新bound ，重置collides
                p.collides = 0;
                //i如果达到了最大，那么就递减
                i = (i != m || m == 0) ? m : m - 1;
            }
            else if ((c = p.collides) < m || m == FULL ||
                     !U.compareAndSwapInt(this, BOUND, b, b + SEQ + 1)) {
                p.collides = c + 1; // 更新冲突
                // i=0 那么就从m开始，否则递减i
                i = (i == 0) ? m : i - 1;          // cyclically traverse
            }
            else
                //递增，往后挪动
                i = m + 1;                         // grow
            // 更新index
            p.index = i;
        }
    }
}
```
理解起来应该有点抽象，当然我分析的也不一定就正确，不要全相信我的描述，结合自己理解。

```
Node q = (Node)U.getObjectVolatile(a, j = (i << ASHIFT) + ABASE);
```
对于上面这个代码，开始理解可能有点难，i << ASHIFT 是用于避免伪共享的，虽然arena数组很大，但是里面并不是每个位置都利用的，还有一些是没有利用的，其间隔就是1 << ASHIFT。

```
Class<?> ak = Node[].class;
ABASE = U.arrayBaseOffset(ak) + (1 << ASHIFT);
```
ABASE  就是arena的起始位置上加了(1 << ASHIFT) 这个偏移，相当于ABASE作为了   arena数组的其实位置，其前(1 << ASHIFT)的位置没有利用。

当发现槽位q 有数据的时候就交换数据，如果cas 失败，说明存在竞争，那么就会执行后面的else,更新bound 还是递增或者递减索引i。
如果槽位q 没有数据，并且索引 i在有效范围内，如果是"第一个"槽位，那么会占据槽位，进行一定时间的自旋，然后阻塞等待，如果不是"第一个"槽位，那么就会换个槽位等待，清空当前槽位，然后索引值减半。（如果设置了超时，需要进行超时判断，如果发生超时就返回）。

对于 i <= (m = (b = bound) & MMASK) , m 的值肯定在[0,MMASK]之间。
```
U.compareAndSwapInt(this, BOUND, b, b + SEQ + 1
```
上面这行代码则是在递增BOUND,SEQ+1的二进制位位：100000001，当它和MMASK 相与的结果相比原来就增加了1，因此这个是递增BOUND。
```
if (m != 0)                // try to shrink
    U.compareAndSwapInt(this, BOUND, b, b + SEQ - 1);
```
对于上面这行代码，SEQ - 1 就是MMASK,这个新bound和MMASK  相与的结果应该变大了，（注意m！=0 ,则说明bound低8位存在1），那么索引i 就越有可能在这个范围内，如果还是没有其他线程来交换，则会再次减小i，也就是说增大了m,反而减小了i, jdk 源码上的注释（try to shrink）则是尝试缩小,不知是不是指的这个含义。

估计我描述的晕咚咚的，这个部分，我也不是特别明白，因此就不好多阐述了，后面抽空还会进行研究研究，如果朋友有新的理解，可以指教一二。

### 总结
Exchanger 还是很有意思的，用于线程之间两两交换数据，在多线程下，互相交换数据的两个线程是不确定的。
在竞争比较小的时候，采用单槽位进行交换数据，当线程来交换数据时，发现槽位为空，则自己在这里等待，否则就和槽位进行交换数据，同时会唤醒等待的线程。
在竞争比较激烈的情况下，就会转到多槽位的交换，这个多槽位的交换，其实思想还是很好理解，但是局部有些细节确实还没有理解透，同时调试也困难，只有自己脑海里不挺的模拟，但是这个可能模拟出错（哈哈）。但是对多槽位的大致思想应该还是明白了，当一个线程来交换的时候，如果"第一个"槽位是空的，那么自己就在那里等待，如果发现"第一个"槽位有等待线程，那么就直接交换，如果交换失败，说明其它线程在进行交换，那么就往后挪一个槽位，如果有数据就交换，没数据就等一会，但是不会阻塞在这里，在这里等了一会，发现还没有其它线程来交换数据，那么就往“第一个”槽位的方向挪，如果反复这样过后，挪到了第一个槽位，没有线程来交换数据了，那么自己就在"第一个"槽位阻塞等待。
第一个槽位并不是指的数组中的第一个，而是逻辑第一个，因为存在伪共享，多槽位中，部分空间没有被利用。
