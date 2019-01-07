### ThreadLocal是什么
Java中的ThreadLocal类允许我们创建只能被同一个线程读写的变量。因此，如果一段代码含有一个ThreadLocal变量的引用，即使两个线程同时执行这段代码，它们也无法访问到对方的ThreadLocal变量
### 如何创建ThreadLocal
ThreadLocal是一个类，那么可以直接new出来，并且可以指定其类型

```
ThreadLocal<Integer>id=new ThreadLocal<Integer>();
```
### ThreadLocal 常用方法
1、设置值
```
id.set(1);
```
2、读取保存在ThreadLocal变量中的值：
```
Integer value = id.get();
```
3、从ThreadLocal中移除某个变量

```
id.remove();
```
ThreadLocal使用方法很简单，ThreadLocal就是一个变量，可以给这个变量赋值（set），取值（get），删除值（remove），只是这个变量和线程相关，同一段ThreadLocal代码在不同线程里面是不一样的，数据相互独立。
### ThreadLocal 源码浅析
分析源码，个人还是喜欢先大概看一下定义，然后再从方法中进行切入（注意看方法注释）。
#### get 方法

```
/**
 * Returns the value in the current thread's copy of this
 * thread-local variable.  If the variable has no value for the
 * current thread, it is first initialized to the value returned
 * by an invocation of the {@link #initialValue} method.
 *
 * @return the current thread's value of this thread-local
 */
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

来梳理一下逻辑：

（1）获得当前线程

（2）获得ThreadLocalMap

来看看ThreadLocalMap是什么
```
static class ThreadLocalMap
```
是ThreadLocal的静态内部类，大致看一下，似乎还很复杂，在ThreadLocalMap的下还有一个静态内部类

```
static class Entry extends WeakReference<ThreadLocal<?>>
```
不过从名字我们可以看出来 ThreadLocalMap 应该是一个key-value的结构，那我们就来找找key和value是什么

```
/**
 * Construct a new map initially containing (firstKey, firstValue).
 * ThreadLocalMaps are constructed lazily, so we only create
 * one when we have at least one entry to put in it.
 */
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

这是ThreadLocalMap 的一个构造函数，从形参我们可以知道，key就是ThreadLocal，至于value暂时可以不用care,不过可以猜测是我们set时的value。

```
/**
 * The initial capacity -- MUST be a power of two.
 */
private static final int INITIAL_CAPACITY = 16;

/**
 * The table, resized as necessary.
 * table.length MUST always be a power of two.
 */
private Entry[] table;

/**
 * The number of entries in the table.
 */
private int size = 0;
```

table是一个Entry的数组而table的大小是2的倍数，初始大小为16.到这里又要中断去看一下Entry是什么

```
/**
 * The entries in this hash map extend WeakReference, using
 * its main ref field as the key (which is always a
 * ThreadLocal object).  Note that null keys (i.e. entry.get()
 * == null) mean that the key is no longer referenced, so the
 * entry can be expunged from table.  Such entries are referred to
 * as "stale entries" in the code that follows.
 */
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

Entry是ThreadLocalMap的一个静态内部类，继承了弱引用，key作为了WeakReference的引用，value是Entry的value,于WeakReference，在下一次GC的时候，会回收WeakReference关联的对象（如果该关联的对象不再被使用）

ThreadLocalMap的结构大致明白了，ThreadLocalMap确实是key-value结构，而这个结构是通过Entry来实现的，key是ThreadLocal 实例，value到这里还不确定（其实看一下set就很容易看出这个就是我们传入的value）。

回到ThreadLocalMap的构造函数

（1）生成一个Entry （key-value）数组 叫table

（2）通过名称和下一行代码，我们知道 这是通过一个hashcode生成一个索引

（3）然后将key ，value 封装成一个 Entry实例，table[i]指向这个实例。

知道ThreadLocalMap，我们再回到get中，再来看看getMap

```
ThreadLocalMap map = getMap(t);
```
```
/**
 * Get the map associated with a ThreadLocal. Overridden in
 * InheritableThreadLocal.
 *
 * @param  t the current thread
 * @return the map
 */
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

我们看到返回的是一个线程中的threadLocals，我们到线程中一探究竟

```
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. 
 * /
ThreadLocal.ThreadLocalMap threadLocals = null;
```

这个threadLocals就是我们开始分析的ThreadLocalMap，到这里我们大致知道了:thread中有一个ThreadLocalMap 它是一个key-value结构，key是我们定义的ThreadLocal，value就是我们设置的值，继续往下看：

```
if (map != null) {
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null) {
        @SuppressWarnings("unchecked")
        T result = (T)e.value;
        return result;
    }
}
```

通过 thread中的ThreadLocalMap获取一个Entry ,参数是this,是谁在调用get方法？是通过我们定义的ThreadLocal变量调用，那么这个this就是我们的ThreadLocal。前面我们知道ThreadLocal就是key,那么现在传入的就是key，那么肯定就是为了获取value

```
/**
 * Get the entry associated with key.  This method
 * itself handles only the fast path: a direct hit of existing
 * key. It otherwise relays to getEntryAfterMiss.  This is
 * designed to maximize performance for direct hits, in part
 * by making this method readily inlinable.
 *
 * @param  key the thread local object
 * @return the entry associated with key, or null if no such
 */
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

getEntry中 通过key计算一个索引，然后根据table查出Entry，如果查出的Entry的key是我们传入的key，那么返回这个Entry。（else 先不看），再对照上面取得Entry中的value，返回，结束。
因为我们是一步一步分析的，到这里是不是有点混乱，那我们来梳理一下：

（1）Thread 里面有一个 ThreadLocalMap, ThreadLocalMap中一个Entry数据结构，它是一个key-value结构，key是ThreadLocal,value是我们需要存的值。由Entry构成了一个table数组，用于存放这些Entry，通过ThreadLocal去存取值，实际就是把自己做为key,把值放到thread中的ThreadLocalMap中，这样就能明白为什么这些数据是线程相关的了，应该这些数据就是存到线程里面，别的线程里面肯定没有呀，画了个图，方便大家理解：
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170910221431823.png)

####  set 方法
有了上面的基础，再来看set就好理解了

```
/**
 * Sets the current thread's copy of this thread-local variable
 * to the specified value.  Most subclasses will have no need to
 * override this method, relying solely on the {@link #initialValue}
 * method to set the values of thread-locals.
 *
 * @param value the value to be stored in the current thread's copy of
 *        this thread-local.
 */
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

（1）首先获取线程里面的ThreadLocalMap

（2）如果map 为空，那么先创建,creat里面就是我们前面分析过的ThreadLocalMap的构造函数，就不再分析了。

（3）如果map不为空，那么就设置值，key就是当前的ThreadLocal(this)

```
/**
 * Set the value associated with key.
 *
 * @param key the thread local object
 * @param value the value to be set
 */
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

（1）根据key 计算得到一个索引值

（2）根据索引值可以得到table 中的Entry

（3）按照指定的条件遍历table，nextIndex从字面意思可以看出是计算下一个索引值（这就是所谓的指定的条件），刚开始table[i]肯定是null的，所以暂时不看那段for循环

（4）将key，value 封装成Entry,将table[i]指向Entry。

ok,set方法我们也大致了解了，基本对ThreadLocal有了比较清楚的认识了，至少是揭开了它神秘的面纱。

### ThreadLocal 源码深入分析
前面我们大致看了一下set 和 get 方法，了解了ThreadLocal的原理，但是还有一些比较细节的地方还没有仔细琢磨，在前面我们看到在get或者set的时候，都会通过key来计算一下索引（table的下标），在回想一下，table是一个数组，存取数据是通过下标来进行的，而这个下标是通过key来计算得到，这个很明显就是一种hash的计算，当然通过名称（key.threadLocalHashCode）我们也很容易的猜到是这样的,在ThreadLocal 类开始处，我们可以看到如下代码

```
/**
 * ThreadLocals rely on per-thread linear-probe hash maps attached
 * to each thread (Thread.threadLocals and
 * inheritableThreadLocals).  The ThreadLocal objects act as keys,
 * searched via threadLocalHashCode.  This is a custom hash code
 * (useful only within ThreadLocalMaps) that eliminates collisions
 * in the common case where consecutively constructed ThreadLocals
 * are used by the same threads, while remaining well-behaved in
 * less common cases.
 */
private final int threadLocalHashCode = nextHashCode();

/**
 * The next hash code to be given out. Updated atomically. Starts at
 * zero.
 */
private static AtomicInteger nextHashCode =
        new AtomicInteger();

/**
 * The difference between successively generated hash codes - turns
 * implicit sequential thread-local IDs into near-optimally spread
 * multiplicative hash values for power-of-two-sized tables.
 */
private static final int HASH_INCREMENT = 0x61c88647;

/**
 * Returns the next hash code.
 */
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

可以看到一个静态的原子变量nextHashCode ，在nextHashCode()方法，不断的增加这个原子变量的计数，而这个增量就是HASH_INCREMENT，从此我们可以知道，每当实例一个ThreadLocal，那么其threadLocalHashCode 都在其原来的值上不断增加 HASH_INCREMENT。

在get 方法中的getEntry：

```
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

在计算索引的时候 用到了ThreadLocal实例的threadLocalHashCode ，然后和(table.length-1)相与，那么得到一个在0-table.length-1的数，这个数就用来存数据，所有取数据的时候直接用相同的方法得到索引就可以了。
当然知道hash的都肯定知道hash碰撞这个词，如果在索引i对应的位置已经存过值了，那么就该重新计算另一个位置索引（这个后面再细谈）
如果在计算的索引中不存在指定的ThreadLocal，那么说明可能发生了碰撞，也可能没有个ThreadLocal,那么执行getEntryAfterMiss 方法

```
/**
 * Version of getEntry method for use when key is not found in
 * its direct hash slot.
 *
 * @param  key the thread local object
 * @param  i the table index for key's hash code
 * @param  e the entry at table[i]
 * @return the entry associated with key, or null if no such
 */
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

如果取出的ThreadLocal不为空，但是不是我们指定的ThreadLocal，那么说明发生了碰撞，需要找下一个索引（再hash）

```
/**
 * Increment i modulo len.
 */
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
```

再次hash的方法也很简单，也就是线性递增，依次往后找就可以了，那么说明存数据的时候也是这种方法（很显然嘛，存取数据的hash算法一定要一致）
如果取出的k（ThreadLocal）为空(这里Entry 不为空),那么说明Entry 关联的这个key(ThreadLocal)被回收了（在程序中我们也许不在引用某个ThreadLocal实例，那么后面可能就会被gc回收掉 ），这个时候需要清理hash表（table），执行 expungeStaleEntry 

```
/**
 * Expunge a stale entry by rehashing any possibly colliding entries
 * lying between staleSlot and the next null slot.  This also expunges
 * any other stale entries encountered before the trailing null.  See
 * Knuth, Section 6.4
 *
 * @param staleSlot index of slot known to have null key
 * @return the index of the next null slot after staleSlot
 * (all between staleSlot and this slot will have been checked
 * for expunging).
 */
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

（1）将table中的索引位置的数据清楚掉

（2）重新hash，为什么要重新hash，因为虽然索引i这个位置的数据没有了，但是在i位置上发生了hash碰撞的Entry数据是通过i再次hash计算得到下一个位置的值，既然i的位置数据都没有了，那么那些Entry有什么意义呢（同时table length的大小变化了，所以必须重新hash），所以需要为它们重新计算需要存放的位置。

所以才有了后面的循环，获得不断获得下个索引值，如果该位置关联的ThreadLocal为空，那么清除数据，否则根据当前的length和 ThreadLocal 重新计算存储的位置，如果重新计算出的位置有数据，那么再hash,依次进行，知道找到位置可以放为止。

最后 ，回到get 方法，如果table中get不到值，那么就会执行：

```
return setInitialValue();
```
这里面很简单 就是设置一个初始值，而这个值通过：

```
protected T initialValue() {
    return null;
}
```

这个方法得到，因此我们可以重写该方法，设置我们自己需要的初始值。

看完get方法 ，我们再来回顾一下ThreadLocalmap里面的set 方法：

```
/**
 * Set the value associated with key.
 *
 * @param key the thread local object
 * @param value the value to be set
 */
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

再开始，我们没有分析循环，因为第一次放入值，肯定不会执行循环。当通过计算出来的索引值获取Entry 不为空时，那么就执行循环
（1）如果取出的Entry就是我们要的数据，那么很好，结束

（2）如果取出的Entry 的key（ThreadLocal）为空，这就麻烦了，说明该Entry关联的ThreadLocal 被回收了，又需要整理table 了，来看看replaceStaleEntry

```
/**
 * Replace a stale entry encountered during a set operation
 * with an entry for the specified key.  The value passed in
 * the value parameter is stored in the entry, whether or not
 * an entry already exists for the specified key.
 *
 * As a side effect, this method expunges all stale entries in the
 * "run" containing the stale entry.  (A run is a sequence of entries
 * between two null slots.)
 *
 * @param  key the key
 * @param  value the value to be associated with key
 * @param  staleSlot index of the first stale entry encountered while
 *         searching for key.
 */
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // Back up to check for prior stale entry in current run.
    // We clean out whole runs at a time to avoid continual
    // incremental rehashing due to garbage collector freeing
    // up refs in bunches (i.e., whenever the collector runs).
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    // Find either the key or trailing null slot of run, whichever
    // occurs first
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // If we find key, then we need to swap it
        // with the stale entry to maintain hash table order.
        // The newly stale slot, or any other stale slot
        // encountered above it, can then be sent to expungeStaleEntry
        // to remove or rehash all of the other entries in run.
        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // If we didn't find stale entry on backward scan, the
        // first stale entry seen while scanning for key is the
        // first still present in the run.
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // If key not found, put new entry in stale slot
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // If there are any other stale entries in run, expunge them
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

说一下主要逻辑吧，想一下，当前这个Entry的key为空了，那么其前一个Entry 的key 是不是也可能为空了呢（这里的前一个并不是i-1的那个，而是基于hash算法得出的前一个，这个很简单，如果由i算i+1的那么就逆推就好了），当然可能

（1）从此Entry往前找失效的Entry，如果找到了就用slotToExpunge标记

（2）从此Entry（取名 a）开始往后找,如果找到了一个key 和我们当前需要设置数据的key相同，那么说明有这个Entry（取名b），我们直接重新改它的值就好（e.value = value;），因为我们在找需要的key是，发现a的key为空，那么我们就把b和a交换，那么b的查找路径就会变短，接下来我们就需要整理 Entry 中key为空的节点了，如果前面有key为空的那么就从前面开始，否则从当前位置开始，expungeStaleEntry 前面讲过，里面主要就是搬数据（重新hash）,cleanSomeSlots 里面比较简单，就是清理Entry,里面调用了expungeStaleEntry。

（3）如果在前面的查找并整理table中没有找到 我们要设置数据的 ThreadLocal,那么就需要构造一个新的Entry了


#### remove 方法
之所以把remove 放到这里讲，其实remove就是清理table,这个恰好在前面就分析过了

```
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

```
/**
 * Remove the entry for key.
 */
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```

简单贴一下源码就可以了，过程就不再分析了.
ok 分析完了，完了。
