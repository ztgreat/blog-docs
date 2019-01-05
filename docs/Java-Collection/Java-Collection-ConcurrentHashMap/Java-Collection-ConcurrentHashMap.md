注：本文源码是JDK8的版本
### ConcurrentHashMap 介绍（jdk 1.8）
ConcurrentHashMap是HashMap的升级版，HashMap是非线程安全的集合，ConcurrentHashMap则可以支持并发操作，

HashMap是我们平时开发过程中用的比较多的集合，ConcurrentHashMap就算用得少，但是听过的肯定不少。

在jdk1.8 中HashMap是通过数组+链表+红黑树实现的，ConcurrentHashMap 也是如此，在前面接触的众多算法中，我们知道，要实现线程安全，要么显示的使用锁来控制，这样代码会很简单，要么使用无锁CAS算法来实现，这样代码比较复杂，并发性能较好，很明显ConcurrentHashMap 为了更好的并发的性能，选择了CAS来实现，因此代码相对就比较复杂了。

因为ConcurrentHashMap  和HashMap都是采用相同的数据结构，因此在分析ConcurrentHashMap 之前，最好是比较了解HashMap,这样要容易理解一些，关于HashMap可以参考前面写的内容：
#### [Java集合之HashMap源码分析](http://blog.ztgreat.cn/article/20)
### 数据结构
![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171213204025683.png)

上面就是ConcurrentHashMap 的逻辑存储结构，hash 表（table）用于散列，table的位置上可以存储的是一个链表，也可能是一颗红黑树。

从这个逻辑结构中，如果基于简单的分析，我们可以知道，操作table 中不同位置上的数据（链表或者树），是互相独立的，也就是线程安全的，同时操作table 中同一个位置上的数据这个是需要同步，此外对table的大小调整也是需要进行同步处理的，这只是一个简单的分析，并不完全准确，下面我们一步一步来看ConcurrentHashMap  是如何做的。

#### 链表节点--Node
```
//节点定义
static class Node<K,V> implements Map.Entry<K,V> {
     final int hash;
     final K key;
     volatile V val; //volatile修饰 ，保证可见性
     volatile Node<K,V> next;
     Node(int hash, K key, V val, Node<K,V> next) {
         this.hash = hash;
         this.key = key;
         this.val = val;
         this.next = next;
     }
    //不允许直接改变value的值  
    public final V setValue(V value) {  
        throw new UnsupportedOperationException();  
    }
     ... // 省略部分方法
}
```
Node 为ConcurrentHashMap 的存储单元，这个HashMap 中的Node是差不多的，只是这里val，next都被volatile 修饰，保证了在多线程中的可见性。

下面是ConcurrentHashMap 中部分属性的定义，有些和HashMap 中是一样的。
```
/**
 * The array of bins. Lazily initialized upon first insertion.
 * Size is always a power of two. Accessed directly by iterators.
 */
transient volatile Node<K,V>[] table; // hash 表 

/**
 * The next table to use; non-null only while resizing.
 */
private transient volatile Node<K,V>[] nextTable; // 用于扩容的，过渡表

/**
 * The largest possible table capacity.  This value must be
 * exactly 1<<30 to stay within Java array allocation and indexing
 * bounds for power of two table sizes, and is further required
 * because the top two bits of 32bit hash fields are used for
 * control purposes.
 */
 // hash表的最大的size，必须是2的n次方这种
private static final int MAXIMUM_CAPACITY = 1 << 30; 

/**
 * The default initial table capacity.  Must be a power of 2
 * (i.e., at least 1) and at most MAXIMUM_CAPACITY.
 */
 //hash表的初始默认大小
private static final int DEFAULT_CAPACITY = 16;

/**
 * The load factor for this table. Overrides of this value in
 * constructors affect only the initial table capacity.  The
 * actual floating point value isn't normally used -- it is
 * simpler to use expressions such as {@code n - (n >>> 2)} for
 * the associated resizing threshold.
 */
private static final float LOAD_FACTOR = 0.75f;

/**
 * The bin count threshold for using a tree rather than list for a
 * bin.  Bins are converted to trees when adding an element to a
 * bin with at least this many nodes. The value must be greater
 * than 2, and should be at least 8 to mesh with assumptions in
 * tree removal about conversion back to plain bins upon
 * shrinkage.
 */
 //链表转成红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;

/**
 * The bin count threshold for untreeifying a (split) bin during a
 * resize operation. Should be less than TREEIFY_THRESHOLD, and at
 * most 6 to mesh with shrinkage detection under removal.
 */
//红黑树转为链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;

/**
 * The smallest table capacity for which bins may be treeified.
 * (Otherwise the table is resized if too many nodes in a bin.)
 * The value should be at least 4 * TREEIFY_THRESHOLD to avoid
 * conflicts between resizing and treeification thresholds.
 */
//存储方式由链表转成红黑树 table 容量的最小阈值
static final int MIN_TREEIFY_CAPACITY = 64;

/**
 * Minimum number of rebinnings per transfer step. Ranges are
 * subdivided to allow multiple resizer threads.  This value
 * serves as a lower bound to avoid resizers encountering
 * excessive memory contention.  The value should be at least
 * DEFAULT_CAPACITY.
 */
//用于hash 表扩容后，搬移数据的步长（下面几个属性都是用于扩容或者控制sizeCtl 变量）
private static final int MIN_TRANSFER_STRIDE = 16;

/**
 * The number of bits used for generation stamp in sizeCtl.
 * Must be at least 6 for 32bit arrays.
 */
private static int RESIZE_STAMP_BITS = 16;

/**
 * The maximum number of threads that can help resize.
 * Must fit in 32 - RESIZE_STAMP_BITS bits.
 */
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

/**
 * The bit shift for recording size stamp in sizeCtl.
 */
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

/*
 * Encodings for Node hash fields. See above for explanation.
 */ 
static final int MOVED     = -1; // 表示这是一个forwardNode节点
static final int TREEBIN   = -2; // 表示这时一个TreeBin节点

// 用于生成hash值
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

```
上面的一些属性，会结合后面的代码来分析，这里先有个大致了解就可以了。
#### 红黑树节点--TreeNode
```
static final class TreeNode<K,V> extends Node<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next,
                 TreeNode<K,V> parent) {
            super(hash, key, val, next);
            this.parent = parent;
        }
        ...
    }
```
TreeNode 用于构建红黑树节点，但是ConcurrentHashMap 中的TreeNode和HashMap中的TreeNode用途有点差别，HashMap中hash 表的部分位置上存储的是一颗树，具体存储的就是TreeNode型的树根节点，而ConcurrentHashMap 则不同，其hash 表是存储的被TreeBin 包装过的树，也就是存放的是TreeBin对象，而不是TreeNode对象，同时TreeBin  带有读写锁，当需要调整树时，为了保证线程的安全，必须上锁。

#### TreeBin 对象
```
/**
* TreeNodes used at the heads of bins. TreeBins do not hold user
* keys or values, but instead point to list of TreeNodes and
* their root. They also maintain a parasitic read-write lock
* forcing writers (who hold bin lock) to wait for readers (who do
* not) to complete before tree restructuring operations.
*/
static final class TreeBin<K,V> extends Node<K,V> {
   TreeNode<K,V> root; // 树根
   volatile TreeNode<K,V> first; // 树的链式结构
   volatile Thread waiter; // 等待者
   volatile int lockState; // 锁状态
   // values for lockState
   static final int WRITER = 1; // set while holding write lock
   static final int WAITER = 2; // set when waiting for write lock
   static final int READER = 4; // increment value for setting read lock
...
}

```
具体用法我们后面结合代码再来分析。

#### 过渡节点--ForwardingNode
```
/**
  * A node inserted at head of bins during transfer operations.
  */
static final class ForwardingNode<K,V> extends Node<K,V> {
  final Node<K,V>[] nextTable;
  ForwardingNode(Node<K,V>[] tab) {
      super(MOVED, null, null, null); // hash 值为MOVED 进行标识 
      this.nextTable = tab;
  }
```
ForwardingNode 用于在hash 表扩容过程中的过渡节点，当hash 表进行扩容进行数据转移的时候，其它线程如果还不断的往原hash 表中添加数据，这个肯定是不好的，因此就引入了ForwardingNode 节点，当对原hash 表进行数据转移时，如果hash 表中的位置还没有被占据，那么就存放ForwardingNode 节点，表明现在hash 表正在进行扩容转移数据阶段，这样，其它线程在操作的时候，遇到ForwardingNode 节点，就知道hash 现在的状态了，就可以协助参与hash 表的扩容过程。

到这里，ConcurrentHashMap 中的重要的数据结构基本都了解了，一个是hash 表（table）,一个是链表节点Node,其实呢就是红黑树节点TreeNode.

### 构造方法
1、无参构造

```
/**
  * Creates a new, empty map with the default initial table size (16).
  */
 public ConcurrentHashMap() {
 }
```
里面什么都没有做，hash 表的初始化，是在第一次put 数据的时候初始化的。
2、指定hash 表大小

```
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
/**
 * Returns a power of two table size for the given desired capacity.
 * See Hackers Delight, sec 3.2
 */
 //确保table的大小总是2的幂次方
private static final int tableSizeFor(int c) {
    int n = c - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
3、通过集合初始化
```
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
     this.sizeCtl = DEFAULT_CAPACITY;
     putAll(m);
 }
```
还要其他一些构造方法，这个可以自己去看，这里就不完全列举了。

### put 数据
```
public V put(K key, V value) {
    return putVal(key, value, false);
}
```
```
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    // 计算hash 值
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //将要初始化table
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //通过hash 值计算table 中的索引，如果该位置没有数据，则可以put    
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // cas 将数据设置到table 中，如果设置成功，则本次put 基本完成
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //如果table位置上的节点状态时MOVE,则表明hash 正在进行扩容搬移数据的过程中
        else if ((fh = f.hash) == MOVED)
            //协助扩容
            tab = helpTransfer(tab, f);
        else {// hash 表该位置上有数据，可能是链表，也可能是一颗树
            V oldVal = null;
            synchronized (f) { //将hash 表该位置进行上锁，保证线程安全
                // 上锁后，只有再该位置数据和上锁前一致才进行，否则需要重新循环
                if (tabAt(tab, i) == f) {
                    // hash 值>=0 表明这是一个链表结构
                    if (fh >= 0) {
                        binCount = 1;
                        // 遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 存在相同的key,则覆盖其value
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 不存在该key，将新数据添加到链表尾
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 该位置是红黑树，是TreeBin对象（注意是TreeBin，而不是TreeNode）
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        //通过TreeBin 中的方法，将数据添加到红黑树中
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            //添加了数据，需要进行检查
            if (binCount != 0) {
                //if 成立，说明遍历的是链表结构，并且超过了阀值，需要将链表转换为树
                if (binCount >= TREEIFY_THRESHOLD)
                    //将table 索引i 的位置上的链表转换为红黑树
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // ConcurrentHashMap 容量增加1，检查是否需要扩容
    addCount(1L, binCount);
    return null;
}
```
先来梳理一下大致逻辑：

1、计算key的hash 值

```
int hash = spread(key.hashCode());
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

2、如果table(hash 表)没有被初始化，则执行table的初始化过程。

3、通过hash 值得到table 中的具体位置i,如果该位置没有数据，则直接将该数据存放在该位置。

4、如果table 表中该位置有数据，如果数据的hash 值为MOVED，则表明在进行table表的扩容工作，则辅助进行table的扩容和数据搬移工作。

5、如果table 表中该位置上的数据有效（存储的真正的数据），锁住该位置，然后执行后面的操作。

6、如果该位置上是链表，则遍历链表，如果存在该key,则根据onlyIfAbsent 决定是否覆盖该value,如果不存在该key,则添加到链表的末尾。

7、如果该位置上是树形结构，则执行树的插入操作。

8、如果数据添加到链表中，则需要检查链表的长度是否超过了阀值，如果是则需要将该链表转换为红黑树。

9、如果上面过程在多线程中，执行失败（提前被其它线程改变），则需要从步骤2 重新开始。

10、递增map的容量，并检查是否需要扩容（addCount）。

从整体来看整个过程还是很清楚的，和HashMap有着大致相同的逻辑，因为ConcurrentHashMap 要保证线程安全，因此在存在竞争的操作上采用了cas 或者加锁的方式进行，当执行失败时，则需要重新开始，有了大致的认识后，接下来我们在挨着分析具体的步骤。
### 初始化table
在多线程下，必须要保证table的初始化只能执行一次。

```
/**
 * Initializes table, using the size recorded in sizeCtl.
 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // 如果sizeCtl <0 则说明其它线程在初始化table,自己等待初始化完成即可
        if ((sc = sizeCtl) < 0)
	        // 让出cpu
            Thread.yield(); // lost initialization race; just spin
        //将要执行table初始化，cas 设置 SIZECTL 值为-1   
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2); // 0.75*n
                }
            } finally {
                //这里无需cas ,前面已经保证了只有一个线程能执行初始化工作
                sizeCtl = sc; // sizeCtl 设置为0.75*n
            }
            break;
        }
    }
    return tab;
}
```
sizeCtl默认为0，如果ConcurrentHashMap实例化时有容量参数，那么sizeCtl会是一个2的幂次方的值。所以执行第一次put操作的线程会执行Unsafe.compareAndSwapInt方法修改sizeCtl为-1，有且只有一个线程能够修改成功，其它线程通过Thread.yield()让出CPU时间片等待table初始化完成，整个过程很清晰。

对于table 扩容部分，稍微后面再来分析，先来奠定点基础。
###  添加数据到链表或树
对于 putVal中synchronized 包裹的内容，就是将数据添加到链表或者红黑树中，具体的代码这里就不再贴了，可以看前面的代码。

在前面的属性定义中有这样的定义：如果是一颗红黑树，那么其根（也就是table中的数据）的hash 值为TREEBIN   
```
static final int TREEBIN   = -2; // hash for roots of trees
```
因此分析table 中某个位置上的数据的hash值，如果是负数则说明是一颗红黑树，否则就是链表结构。

对于数据添加到链表中，这个应该很简单，很容易就看懂了，其binCount 用于标识链表长度，其用法还是很巧妙的。

如果该位置是一颗树，那么其table 存放的就是TreeBin 对象，TreeBin 是中有树根的引用同时拥有一系列的红黑树的操作，因此直接调用TreeBin  对象的添加方法（putTreeVal）即可，对于红黑树的操作部分，在本文不会多讲，如果对其感兴趣或者不太清楚的，可以参看另外两篇的内容，里面有对红黑树的详细介绍
#### [亲自动手画红黑树](http://blog.ztgreat.cn/article/12)

#### [Java集合之TreeMap源码分析](http://blog.ztgreat.cn/article/26)

### 将链表转换为红黑树
当添加数据到链表中后，如果链表的长度超过了阀值，那么会将链表转换为红黑树。

```
/**
 * Replaces all linked nodes in bin at given index unless table is
 * too small, in which case resizes instead.
 */
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        // 如果table的容量不满足链表转换为红黑树的阀值要求，则需要对table 进行扩容
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            // 将table 中该位置 锁住
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    // 遍历链表，构造TreeNode链表
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            //利用TreeNode 中的next域，构造TreeNode的链表
                            tl.next = p;　
                        tl = p;
                    }
                    //new TreeBin<K,V>(hd)  是将TreeNode 链表构造成一颗红黑树
                    //将table原位置上的链表TreeNode 对象更改为TreeBin 对象
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```
大致过程：

1、如果table的容量不满足链表转换为红黑树的阀值要求，则需要对table 进行扩容

2、锁住table 表该位置上的数据，遍历链表，将Node 链表转换为TreeNode 链表

3、通过TreeBin对象通过TreeNode 链表构造红黑树结构，树根存储在TreeBin对象中

4、将TreeBin 对象存储在table 原链表Node的位置上，至此链表转红黑树完成。

将链表转换为红黑树之前会构造TreeNode 链表，这样在构造成红黑树后，不仅可以通过树的方式遍历该结构，同时TreeNode 之间也存在一种链式结构，该结构就是最初的链表转换为红黑树时构造的关系,在HashMap也存在这种结构，同时里面有相关的图示（在最后部分），可以参考：

####  [Java集合之HashMap源码分析](http://blog.ztgreat.cn/article/20)

在构造TreeBin 对象时，通过传入的TreeNode 链表，会构造一颗红黑树，这个可以看看TreeBin的构造方法，这里就不贴出来了。

### 扩容
当table容量不足的时候，即table的元素数量达到容量阈值sizeCtl，需要对table进行扩容。
整个扩容分为两部分：

1、构建一个nextTable，大小为table的两倍。
2、把table的数据复制到nextTable中。

扩容在HashMap和ConcurrentHashMap 中都是重头戏，ConcurrentHashMap是支持并发插入的，扩容操可以有两种方式，一种是如同初始化table那样，整个过程都控制只有一个线程进行操作，这样肯定实现比较容易，但是这样会影响到性能，当数据量比较大时，搬移数据将是一个费事操作，追求完美的jdk 当然不是那样实现的，构建nextTable 这个肯定只有一个线程来执行，但是将table 中的数据复制到nextTable 中，这个可以进行并发复制，这样的话，实现就比较复杂了，在无锁的线程安全的算法中，都用到了一种思想：**辅助**

在分析SynchronousQueue 中阐述过下面的一段话：
不使用锁来保证数据结构的完整性，要确保其他线程不仅能够判断出第一个线程已经完成了更新还是处在更新的中途，还能够判断出如果第一个线程操作状态，完成更新还需要什么操作。如果线程发现了处在更新中途的数据结构，它就可以 “**帮助**” 正在执行更新的线程完成更新，然后再进行自己的操作。当第一个线程回来试图完成自己的更新时，会发现不再需要了，返回即可。

因此在table 复制数据的过程中，其它线程是可以参与一同进行复制的，这样可以极大的提高效率，同时也必须要保证数据结构不被破坏，下面我们一步一步来看看是如何实现这一过程的。

分析扩容，我们先从addCount 这个方法入手，当添加数据后，会递增map中的size的计数，同时会检查table 是否需要扩容
```
private final void addCount(long x, int check) {
    long  s;
    ... //中间省略 map 数量增加x 的过程
    
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            // 标记位（这里理解不是很清楚）  
            int rs = resizeStamp(n);
            if (sc < 0) {
                // 这个分支 属于协助扩容部分
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 协助扩容，递增SIZECTL，表明有新线程参与扩容了
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            //设置SIZECTL 标记（rs << RESIZE_STAMP_SHIFT 为负数）
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                //扩容，最先进行扩容的线程执行这里                         
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```
这里相对比较难理解，个人也未完全明白，但是并不妨碍我们掌握整个脉络。

```
/**
 * Returns the stamp bits for resizing a table of size n.
 * Must be negative when shifted left by RESIZE_STAMP_SHIFT.
 */
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```
注意看注释，将会保证其结果左移RESIZE_STAMP_SHIFT 为负数。

最新进行扩容的线程，其sizeCtl 为不为负数，因此将会执行后面的分支，这个时候，会设置SIZECTL的值为(rs << RESIZE_STAMP_SHIFT) + 2，而rs << RESIZE_STAMP_SHIFT 必定是一个负数。
竞争的线程判断如果不需要辅助扩容，则会跳出，否则将会进行辅助扩容，同时递增SIZECTL，在条件判断的地方，个人未完全参透，但是其意图是可以知道的。
#### 辅助扩容
transfer 这个方法便是扩容的核心方法了，在看这个方法前，我们需要先看helpTransfer 方法：
```
else if ((fh = f.hash) == MOVED)
       tab = helpTransfer(tab, f);
```

```
/**
 * Helps transfer if a resize is in progress.
 */
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) { 
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
             //退出  
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
             // 辅助扩容，递增SIZECTL
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```
可以看到这个helpTransfer 方法，和前面我们展示addCount 中的部分是差不多的逻辑，判断是否需要进行辅助扩容，如果参与扩容，则需要递增SIZECTL。

因此现在我们的重点就是transfer 方法了，鉴于transfer 代码有点长，这里先简单的阐述一下过程：

1、获取遍历table的步长

2、最先进行扩容的线程，会初始化nextTable（正常情况下，nextTable 是table的两倍大小）

3、计算table某个位置索引 i,该位置上的数据将会被转移到nextTable 中。

4、如果索引i 所对应的table的位置上没有存放数据，则放在ForwardingNode 数据，表明该table 正在进行扩容处理。（如果有添加数据的线程添加数据到该位置上，将会发现table的状态）

**5**、将索引i 位置上的数据进行转移，**数据分成两部分**，一部分就是数据 在nextTable 中索引没有变（仍然是i），另一部分则是其索引变成i+n的,将这两部分分别添加到nextTable 中。

transfer 中的核心思想大致就是这样，这里有一个细节，就是步骤5 将数据分成两部分，这个其实如果了解jdk 1.8 的HashMap，那么也就知道这里为什么这样了，这个得益于table的大小必须是2的n次幂，这个在HashMap中有详细阐述，这里再次简单描述一下吧：

在ConcurrentHashMap中，扩容后的大小是原来的2倍（这个我们在后面会看到），所以，元素的位置要么是在原位置，要么是在原位置+n的位置（n 为原table的大小）。看下图可以明白这句话的意思，n为table的长度，图（a）表示扩容前的key1和key2两种key确定索引位置的示例，图（b）表示扩容后key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希与高位运算结果。
![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171216160725620.png)

有了这种结果，就可以避免将所有数据再次hash，可以判断hash 值的某个位置上的二进制位，来确定其索引是没有变，还是变成了i+n了。

有了上面的基础，再来看代码，应该就好理解多了
```
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        // 步长
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // 初始化 nextTab    
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        // 设置transferIndex 
        transferIndex = n;
    }
    int nextn = nextTab.length;
    // 转移标识节点
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    // 计算和控制对table 表中位置i 的数据进行转移
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            // 本轮转移完毕    
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 更新TRANSFERINDEX（stride 为步长）
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        //扫描table 本轮完成
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            //如果finished,则设置新tabl,sizeCtl ,返回
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            //利用CAS方法更新这个扩容阈值，sizectl值减一，即将退出扩容
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                //有其它线程在辅助，直接退出
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                //扩容完成，辅助的线程也退出了，设置完成标识，最后再recheck 一下  
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // table 索引i 的位置上没有数据
        else if ((f = tabAt(tab, i)) == null)
            // 设置 为ForwardingNode 节点，这样其它线程可以感知到table 状态
            advance = casTabAt(tab, i, null, fwd);
        //已经设置过了    
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 位置上是数据，需要锁住该位置，进行数据转移
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // 链表数据
                    if (fh >= 0) {
                        // 得到hash 值 高位的二进制值（高位：n的二进制位数）
                        int runBit = fh & n;
                        // 最后添加的数据
                        Node<K,V> lastRun = f;
                        // 遍历链表
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
	                        // 得到hash 值 高位的二进制值
                            int b = p.hash & n;
                            //和前个节点不一致，则更新
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        // 更加runBit  设置相应值
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        // 遍历链表，注意调节p != lastRun
                        // lastRun 之后的数据其高位也是一致
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            // 高位是0,则索引没有变，添加到ln 链表中
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                // 索引变为i+n的数据添加到hn 链表中
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 设置nextTab 索引i 的位置上的数据为链表ln
                        setTabAt(nextTab, i, ln);
                        // 设置nextTab 索引i 的位置上的数据为链表hn
                        setTabAt(nextTab, i + n, hn);
                        //原table  索引i 的位置上的数据转移完成，用ForwardingNode节点标识
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    // 是红黑树
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        //这里是链式遍历，而不是树的方式遍历
                        // 这和我们前面说的一致:树中保存了链式关系，在构造树的时候构建的。
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            // 下面也是分成两部分，重新构建TreeNode 链表
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        // 如果不满足树要求，则转化为链表，否则转换为TreeBin 对象（内部建树）
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        // 设置nextTab 索引i 的位置上的数据为ln（可能是链表，也可能是树）
                        setTabAt(nextTab, i, ln);
                         // 设置nextTab 索引i+n 的位置上的数据为hn
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
						// 重设标识，继续循环，查找下次需要转移数据的索引i
                        advance = true;
                    }
                }
            }
        }
    }
}
```
这代码够长的，但是结合前面的叙述和代码中的注释，我相信掌握其脉络应该不成问题，难理解的应该是table 索引i的控制和计算了，并没有顺序遍历table 进行转移数据，因为多线程下，如果多个线程辅助转移数据，会导致冲突频繁，因此设置了一个步长，相当于不同的线程转移table 中不同 位置段 上的数据，这样可以减小冲突。

在转移数据过程中，如果是链表结构，则需要遍历两次链表，第一次的意义在于减少数据操作，如果链表中后半部分的数据，有着相同的高位，那么第二次遍历就可以缩短，如果不幸，数据的hash值高位情况分布的很乱，那么第一次的意义就不大了，将链表分成两部分，一部分是索引没有变的，一部分是索引值变为i+n,然后设置在新table 中即可。

如果是红黑树节点，同样的操作方式，分成两部分，在遍历树时，采用的是链式遍历，当重新构建树时，也会再次构建链式关系(这个可以看TreeBin的构造方法）,如果发现树的数量不满足最低要求，则把树转换为链表结构。

终于把扩容分析完了，过程有点漫长，接下来要容易多了，不过还有一个难点---**ConcurrentHashMap 的size 统计**。

### size 的统计
ConcurrentHashMap 是可以并发执行的，因此其size 只是一个瞬时值，当你拿到该值时，其大小很有可能已经变化很大了，因此其size 值可以参考，但是不要依赖。

传统算法中，用一个变量计算容器大小即可，当有改变时，更新该变量就可以，但是在ConcurrentHashMap 中是并发操作的，如果用一个变量进行统计的话，那么为了保证正确性，需要锁住，或者循环cas，但是如果在竞争比较激烈的时候，在size的设置上将会冲突很大，反而影响了性能，有点不划算，因此ConcurrentHashMap  采用了分而治之设计思想，什么意思呢，这个我们看一下其中的部分定义：

```
/**
 * Table of counter cells. When non-null, size is a power of 2.
 */
 /**
  * counterCells数组，总数值的分值分别存在每个cell中
  */
private transient volatile CounterCell[] counterCells;

/**
 * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
 */
/**
 * 标识当前cell数组是否在初始化或扩容中的CAS标志位
 */
private transient volatile int cellsBusy;

/**
 * Base counter value, used mainly when there is no contention,
 * but also as a fallback during table initialization
 * races. Updated via CAS.
 */
/**
 * 总和值的获得方式为 base + 每个cell中分值之和
 * 在并发度较低的场景下，所有值都直接累加在base中
 */
private transient volatile long baseCount;
```
我相信看了上面的定义，应该明白分而治之的思想了吧。

```
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
// 计算总和
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        // baseCount+每个cell 的值
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```
现在我们再来看ConcurrentHashMap 中addCount 中 计算size的方法：

```
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    //如果counterCells 为null ,则累计到baseCount 上
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        // 是否冲突标志，默认未冲突
        boolean uncontended = true;
		 // 获取当前线程的probe值，设置到某个cell 中，如果冲突，设置失败，则执行fullAddCount 方法
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        //计算总和    
        s = sumCount();
    }
    ... // 省略后面table 扩容代码
}
```
```
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    // 获取当前线程的probe值
    // 如果为0，则初始化当前线程probe值
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        // 该静态方法会初始化当前线程所持有的随机值
        ThreadLocalRandom.localInit();      // force initialization
        // 获取生成后的probe值，用于选择cells数组下标元素
        h = ThreadLocalRandom.getProbe();
        // 由于重新生成了probe，未冲突标志位设置为true
        wasUncontended = true;
    }
     
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        // cells数组已经被成功初始化
        if ((as = counterCells) != null && (n = as.length) > 0) {
            // 通过该值与当前线程probe求与，获得cells的下标元素，和hash 表获取索引是一样的
            if ((a = as[(n - 1) & h]) == null) {
                // cellsBusy 为0表示cells数组不在初始化或者扩容状态下
                if (cellsBusy == 0) {            // Try to attach new Cell
                    //创建新cell
                    CounterCell r = new CounterCell(x); // Optimistic create
                    // CAS设置cellsBusy，防止其它线程来破坏数据结构
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                //将新创建的cell放入对应下标位置
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            //恢复标识位(未占用)
                            cellsBusy = 0;
                        }
						//操作成功，则退出死循环
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            // 获取了probe对应cells数组中的下标元素，发现不为空
            // 并且调用该函数前，调用方CAS操作也已经失败（已经发生竞争）
            else if (!wasUncontended)       // CAS already known to fail
               // 设置未冲突标志位后，重新生成probe，进入死循环
                wasUncontended = true;      // Continue after rehash
            //CAS 执行累加    
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                // 成功，退出
                break;
            // 如果数组比较大了，则不需要扩容，继续重试，或者已经扩容了，重试
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
            // 过期了，重试    
            else if (!collide)
                collide = true;
            // 对Cell数组进行扩容，CAS设置cellsBusy值    
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == as) {// Expand table unless stale
                        //容量增加1倍
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    // 恢复
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            //重新生成一个probe值
            h = ThreadLocalRandom.advanceProbe(h);
        }
        // 初始化Cell 数组
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {                           // Initialize table
                if (counterCells == as) {
	                // 默认初始容量为2
                    CounterCell[] rs = new CounterCell[2];
                    //将x的值设置到cell 数组中
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    // 初始化成功
                    init = true;
                }
            } finally {
                // 恢复
                cellsBusy = 0;
            }
            // 初始化成功，退出死循环
            if (init)
                break;
        }
        //竞争激烈，其它线程占据cell 数组，直接累加在base变量中
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```
是不是发现这个fullAddCount 方法也不简单，其大致逻辑如下：

1、初始化用于获取counterCells数组的索引值（线程相关）

2、如果存在counterCells数组，如果counterCells 数组中不存在数据，尝试占据counterCells 数组，创建cell,将cell 设置到counterCells 中的某个位置上，退出循环

3、如果counterCells 中该位置上存在数据，如果有冲突，则重新循环，否则尝试累加数据到cell 中，成功则退出循环。

4、如果counterCells 数组扩容，或者不需要扩容，则重新循环

5、对counterCells  进行扩容操作（容量增加1倍），然后重新循环

6、如果不存在counterCells 数组，则尝试进行初始化，并设置数据到某个cell

7、如果尝试初始化失败，则可能其它线程再初始化，那么累计数据到baseCount，成功就退出，否则重新循环。

到这里，应该基本明白是如何更新ConcurrentHashMap的size 大小的吧，接下来的内容就真的很轻松了，同时分析到这里，对ConcurrentHashMap 应该有了大致认识，明白其原理和一些核心思想。

### get 方法
get  方法就简单多了，无非就是遍历链表或者树，进行数据查找工作。

```
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 获得hash 值
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // 如果tbale 中的hash 值就等于需要查找数据的hash 值
        if ((eh = e.hash) == h) {
            // 如果key相等，则返回其value
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 是树结构，这些红黑树的查找
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 链表结构，遍历链表查找数据    
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
### remove 方法
remove 相对要比get 方法麻烦点，设计到容量的改变，但是经历了扩容分析后，还是很简单的。

```
public V remove(Object key) {
    return replaceNode(key, null, null);
}
```

```
final V replaceNode(Object key, V value, Object cv) {
    int hash = spread(key.hashCode());
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0 ||
            (f = tabAt(tab, i = (n - 1) & hash)) == null)
            break;
        // table 正在扩容，协助扩容    
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            boolean validated = false;
            // 锁住数据
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    // 链表结构
                    if (fh >= 0) {
                        validated = true;
                        //遍历链表，从中删除节点
                        for (Node<K,V> e = f, pred = null;;) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                V ev = e.val;
                                if (cv == null || cv == ev ||
                                    (ev != null && cv.equals(ev))) {
                                    oldVal = ev;
                                    if (value != null)
                                        e.val = value;
                                    else if (pred != null) //存在前驱，直接删除节点
                                        pred.next = e.next;
                                    else // 没前驱，设置table 中索引i位置上的数据为其后继节点
                                        setTabAt(tab, i, e.next);
                                }
                                break;
                            }
                            pred = e;
                            if ((e = e.next) == null)
                                break;
                        }
                    }
                    //树结构
                    else if (f instanceof TreeBin) {
                        validated = true;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val;
                            if (cv == null || cv == pv ||
                                (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                else if (t.removeTreeNode(p)) // 从树中删除节点
                                    //如果树太小，需要转换成链表结构
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                }
            }
            if (validated) {
                if (oldVal != null) {
                    if (value == null)
                        //递减容器容量，不需要检查是否扩容
                        addCount(-1L, -1); 
                    return oldVal;
                }
                break;
            }
        }
    }
    return null;
}
```
删除整个过程也不麻烦，也就是设计到链表和红黑树的删除操作，在操作table 表中的某个数据时，需要锁住该位置，在操作红黑树中，如果需要调整树结构，需要锁住树。

### 总结
ConcurrentHashMap 是HashMap的升级版，HashMap是非线程安全的，ConcurrentHashMap 则可以进行并发操作，两者都采用了相同的数据结构：数组+链表+红黑树，因此如果在HashMap的基础上再来分析ConcurrentHashMap ，那么就会容易的很多。

两者在扩容的过程中都相对比较麻烦，ConcurrentHashMap  扩容中，可以其它线程参与来辅助搬移数据，两者在转移数据的过程中，都是将原数据分成两部分，一部分是索引没有变的，另一部分则是索引值为i+n的，这样可以数据放到指定的位置上，而不需要再重新计算索引位置。

ConcurrentHashMap  的size在多线程中，采用的分而治之的思想，类似于ConcurrentHashMap  的hash 表的思想，采用多个数组来保存容器size的值，最后累计所有数组中的值即可，这样可以降低冲突。

ConcurrentHashMap  中还有很多方法，没有一一分析，虽然不能完全理解ConcurrentHashMap  的精髓，但是掌握其脉络还是可以的，最初看ConcurrentHashMap  还是挺难的，但是多花时间，慢慢啃，总会理清思路的，正如当初看HashMap的时候，也觉得很难，但是总算还是啃下来了，当现在回想的时候，感觉其实也没有那么难，先掌握思想，再去研究细节，而不是一来就扎入某个细节而难以自拔，当在不断的看别人代码，分析抽象的代码时，就算不能完全理解，但是能理解一部分也算是成长，站的高度不同，理解程度也有差异，静下心来，慢慢分析代码，本身也是一种提高，如果实在看不懂某个部分，不妨放一放，过段时间再来理解，或者尝试站在写代码的人的角度来思考，也会有不同的收货。

最后，在读本文之前，最后有HashMap的基础，这样能更好的理解，同时看看 SynchronousQueue之类的循环cas 算法，可以更好的帮你理解无锁算法。

####  [Java集合之HashMap源码分析](http://blog.ztgreat.cn/article/20)

####  [亲自动手画红黑树](http://localhost:8000/article/12)

####  [Java集合之TreeMap源码分析](http://blog.ztgreat.cn/article/26)

####  [Java 并发 --- 阻塞队列之SynchronousQueue源码分析](http://localhost:8000/article/23)


