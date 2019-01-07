在学习ConcurrentSkipListMap 之前 我们需要先来学习一种随机化的数据结构--跳跃表(skip list)
对于数组的查找可以有很多方法，如果是有序的，那么可以采用二分查找，二分查找要求元素可以随机访问，所以决定了需要把元素存储在连续内存。这样查找确实很快，但是插入和删除元素的时候，为了保证元素的有序性，就需要大量的移动元素了。 

对于链表而言，不能进行随机访问，也就是说不能单纯的使用二分查找，只能通过顺序遍历的方式，为了使链表也能有很好的查询性能，因此才衍生出跳跃表这种数据结构，而且跳跃表相比平衡树，其实现也相对简单许多。

### 跳跃表介绍
跳跃表（skiplist）是一种随机化的数据结构， 跳跃表以有序的方式在层次化的链表中保存元素， 效率和平衡树媲美 —— 查找、删除、添加等操作都可以在对数期望时间下完成， 并且比起平衡树来说， 跳跃表的实现要简单直观得多，通过下面图示，可以很直观的了解跳跃表。

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180115161001576.png)

上面这张图就是一个跳跃表的实例，先说一下跳跃表的构造特征：

 - 一个跳跃表应该有若干个层（Level）链表组成；
 - 跳跃表中最底层的链表包含所有数据，上层链表存储的数据的引用；
 - 跳跃表中的数据是有序的；
 - 头指针（head）指向最高一层的第一个元素；

通过这样建立多层链表的方式，从顶层开始查找数据，实现了跳跃式的查询数据，利用空间来换取时间，下面就一起来看看跳跃表的建立以及查询过程。

### 跳跃表的查找
SkipListd的查找算法较为简单，对于上面我们我们要查找元素42，其过程如下：

1、比6大，往后找（20），20后面没有数据了，进入20的下一层

2、比20大，继续往后找，找到42.

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180115172158076.png)

### 跳跃表的插入
为什么说跳跃表是一种随机化的数据结构呢，那是因为在每次添加数据后，会随机决定这个数据是否能够攀升到高层次的链表中，也就是同样的数据构造出来的跳跃表可能是不相同的，这个过程我们看看下面的图解：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180115163546220.png)

#### 添加数据
添加数据42：

首先会先查找42应该插入位置的前驱节点，从head 所指节点开始查找（先向右，再向左），当查找节点20时，发现后面没有结点了，那么进行下一层进行查找，那么这样就会查找到节点42的前驱应该是30这个结点，然后将数据插入到最底层链表中:

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180115164834534.png)

然后通过随机过程确定节点42是否会像上进行攀爬，并且确定攀爬的层次，假设这里需要向上攀爬两层，那么其结果如下所示：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180116170117473.png)

#### 连接索引层
当然这样并没有完，现在我们把数据插入到了链表中，同时生成了索引层，那么还有一步，就是**连接索引层**

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180116170332504.png)

### 跳跃表的删除
跳跃表的删除其实和插入差不多，先查找节点，然后移除每一层的链接即可。

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180115173012611.png)

至此对跳跃表应该有了一定的认识了，下面来看看jdk 中ConcurrentSkipListMap 是如何实现跳跃表的，结合具体的分析，也可以加强理解。

### ConcurrentSkipListMap（jdk 1.8）
ConcurrentSkipListMap 一个并发安全, 基于 skip list 实现有序存储的Map,ConcurrentSkipListMap提供了三个内部类来构建这样的链表结构：Node、Index、HeadIndex。其中Node表示最底层的单链表有序节点、Index表示为基于Node的索引层，HeadIndex用来维护索引层次。

###  继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180115202052246.png)
###  数据结构
Node节点定义：
```
static final class Node<K,V> {
    final K key;
    volatile Object value;
    volatile Node<K,V> next;

    /**
     * Creates a new regular node.
     */
    Node(K key, Object value, Node<K,V> next) {
        this.key = key;
        this.value = value;
        this.next = next;
    }
    ... //省略部分方法
}
```

Index 定义：
```
static class Index<K,V> {
      final Node<K,V> node;  // 数据节点 引用
      final Index<K,V> down; //下层 Index
      volatile Index<K,V> right; // 右边Index

      /**
       * Creates index node with given values.
       */
      Index(Node<K,V> node, Index<K,V> down, Index<K,V> right) {
          this.node = node;
          this.down = down;
          this.right = right;
      }
      ...//省略部分代码
}
```
HeadIndex 定义：
```
/**
 * Nodes heading each level keep track of their level.
 */
static final class HeadIndex<K,V> extends Index<K,V> {
    final int level; //索引层，从1开始，Node单链表层为0
    HeadIndex(Node<K,V> node, Index<K,V> down, Index<K,V> right, int level) {
        super(node, down, right);
        this.level = level;
    }
}
```
HeadIndex 增加一个 level 属性用来标示索引层级; **注意**所有的 HeadIndex 都指向同一个 Base_header 节点;
```
 /**
  * Special value used to identify base-level header
  */
 private static final Object BASE_HEADER = new Object();

 /**
  * The topmost head index of the skiplist.
  */
 private transient volatile HeadIndex<K,V> head; // HeadIndex的头指针

 /**
  * The comparator used to maintain order in this map, or null if
  * using natural ordering.  (Non-private to simplify access in
  * nested classes.)
  * @serial
  */
 final Comparator<? super K> comparator;  //元素比较器
```
上面是ConcurrentSkipListMap 中跳跃表的定义，但是仍然比较抽象，还是转换成比较好理解的图示吧：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180115212537964.png)

**注意**：在上面的跳跃表的简单图示中，没有画出HeadIndex结构

### 构造方法
1、默认构造
```
/**
 * Constructs a new, empty map, sorted according to the
 * {@linkplain Comparable natural ordering} of the keys.
 */
public ConcurrentSkipListMap() {
    this.comparator = null;
    initialize();
}
```
2、指定比较器

```
/**
 * Constructs a new, empty map, sorted according to the specified
 * comparator.
 */
public ConcurrentSkipListMap(Comparator<? super K> comparator) {
    this.comparator = comparator;
    initialize();
}
```
3、通过集合初始化
```
/**
 * Constructs a new map containing the same mappings as the given map,
 * sorted according to the {@linkplain Comparable natural ordering} of
 * the keys.
 */
public ConcurrentSkipListMap(Map<? extends K, ? extends V> m) {
    this.comparator = null;
    initialize();
    putAll(m);
}

/**
 * Constructs a new map containing the same mappings and using the
 * same ordering as the specified sorted map.
 */
public ConcurrentSkipListMap(SortedMap<K, ? extends V> m) {
    this.comparator = m.comparator();
    initialize();
    buildFromSorted(m);
}
```
构造方法中都调用了initialize() 方法

```
private void initialize() {
    keySet = null;
    entrySet = null;
    values = null;
    descendingMap = null;
    //初始化head
    head = new HeadIndex<K,V>(new Node<K,V>(null, BASE_HEADER, null),
                              null, null, 1);
}
```
初始创建的跳跃表结构如下：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180116145422248.png)

### 添加数据（put）

```
public V put(K key, V value) {
    if (value == null)
        throw new NullPointerException();
    return doPut(key, value, false);
}
```
要求value 不能为空，内部调用doPut 方法。

#### findPredecessor 方法
findPredecessor 方法的功能是寻找某个key的前驱（如果遇到需要删除的节点，那么进行辅助删除）。从最高层的headIndex开始向右一步一步比较，直到right为null或者右边节点的Node的key大于当前key为止，然后再向下寻找，依次重复该过程，直到down为null为止，即找到了前驱。
```
private Node<K,V> findPredecessor(Object key, Comparator<? super K> cmp) {
    if (key == null)
        throw new NullPointerException(); // don't postpone errors
    for (;;) {
        // 从head 开始遍历
        for (Index<K,V> q = head, r = q.right, d;;) {
             // r != null，表示该节点右边还有节点，需要进行比较
            if (r != null) {
                Node<K,V> n = r.node;
                K k = n.key;
                // value == null，表示该节点已经被删除了
                // 通过unlink()删除该节点
                if (n.value == null) {
                    if (!q.unlink(r))
                        break;           // restart
                    r = q.right;         // reread r
                    continue;
                }
                 // 如果key 大于r节点的key 则继续向后遍历
                if (cpr(cmp, key, k) > 0) {
                    q = r;
                    r = r.right;
                    continue;
                }
            }
            //如果dowm == null，表示指针已经达到最下层了，直接返回该节点
            if ((d = q.down) == null)
                return q.node;
            //否则进入下层查找   
            q = d;
            r = d.right;
        }
    }
}
```
#### doPut 方法
```
private V doPut(K key, V value, boolean onlyIfAbsent) {
    Node<K,V> z;             // added node
    if (key == null)
        throw new NullPointerException();
    Comparator<? super K> cmp = comparator;
    outer: for (;;) {
        // b 为前继节点, n是前继节点的next
        for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
            if (n != null) {
                //b 是链表的最后一个节点
                Object v; int c;
                Node<K,V> f = n.next;
                //多线程下 发生了竞争
                if (n != b.next)               // inconsistent read
                    break;
                //节点n已经逻辑删除了,进行辅助物理删除    
                if ((v = n.value) == null) {   // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                if (b.value == null || v == n) // b is deleted
                    break;
                 // 节点大于，往后继续查找  
                if ((c = cpr(cmp, key, n.key)) > 0) {
                    b = n;
                    n = f;
                    continue;
                }
                //相等，根据参数onlyIfAbsent  决定是否覆盖value
                if (c == 0) {
                    if (onlyIfAbsent || n.casValue(v, value)) {
                        @SuppressWarnings("unchecked") V vv = (V)v;
                        return vv;
                    }
                    // 竞争失败 重来
                    break; // restart if lost race to replace value
                }
                // else c < 0; fall through
            }
            //创建节点
            z = new Node<K,V>(key, value, n);
            //插入节点，如果失败，则重来
            if (!b.casNext(n, z))
                break;         // restart if lost race to append to b
            break outer;
        }
    }
    //随机数
    int rnd = ThreadLocalRandom.nextSecondarySeed();
    // 判断是否需要添加level
    if ((rnd & 0x80000001) == 0) { // test highest and lowest bits
        int level = 1, max;
        //获取 level
        while (((rnd >>>= 1) & 1) != 0)
            ++level;
        Index<K,V> idx = null;
        HeadIndex<K,V> h = head;
        //如果层次level大于最大的层次话则需要新增一层，否则就在相应层次以及小于该level的层次进行节点新增处理。
        // level比最高层次head.level小，直接生成需要的index
        if (level <= (max = h.level)) {
            for (int i = 1; i <= level; ++i) //生成index
                idx = new Index<K,V>(z, idx, null);
        }
        else { // level > max 
            level = max + 1; // hold in array and later pick the one to use
            @SuppressWarnings("unchecked")Index<K,V>[] idxs =
                (Index<K,V>[])new Index<?,?>[level+1];
            for (int i = 1; i <= level; ++i) //生成index
                idxs[i] = idx = new Index<K,V>(z, idx, null);
            for (;;) {
                h = head;
                int oldLevel = h.level;
                // 层次扩大了，需要重新开始（其它线程改变了跳跃表）
                if (level <= oldLevel) // lost race to add level
                    break;
                HeadIndex<K,V> newh = h;
                Node<K,V> oldbase = h.node;
                // 生成新的HeadIndex节点
                for (int j = oldLevel+1; j <= level; ++j)
                    newh = new HeadIndex<K,V>(oldbase, newh, idxs[j], j);
                //更新head    
                if (casHead(h, newh)) {
                    h = newh;
                    idx = idxs[level = oldLevel];
                    break;
                }
            }
        }
        /**
         *前面生成了索引层，但是并没有将这些Index插入到相应的层次当中
         *下面的代码就是将index链接到相对应的层当中
         */
        // 从插入的层次level开始
        splice: for (int insertionLevel = level;;) {
            int j = h.level;
             //从headIndex开始
            for (Index<K,V> q = h, r = q.right, t = idx;;) {
                if (q == null || t == null)
                    break splice;
                // r != null；这里是找到相应层次的插入节点位置，注意这里只横向找    
                if (r != null) {
                    Node<K,V> n = r.node;
                    // compare before deletion check avoids needing recheck
                    int c = cpr(cmp, key, n.key);
                    //需要删除r
                    if (n.value == null) {
                        if (!q.unlink(r))
                            break;
                        r = q.right; //继续向后
                        continue;
                    }
                    //向右进行遍历
                    if (c > 0) {
                        q = r;
                        r = r.right;
                        continue;
                    }
                }
                // 上面找到节点要插入的位置，这里就插入
                if (j == insertionLevel) {
                    //建立链接，失败重试
                    if (!q.link(r, t))
                        break; // restart
                    if (t.node.value == null) {
                        findNode(key);// 查找节点，查找过程中会删除需要删除的节点
                        break splice;
                    }
                    //链接完毕
                    if (--insertionLevel == 0)
                        break splice;
                }
			    //向下继续链接其它index 层
                if (--j >= insertionLevel && j < level)
                    t = t.down;
                q = q.down;
                r = q.right;
            }
        }
    }
    return null;
}
```
doPut方法代码比较长，来梳理一下逻辑：

1、通过 findPredecessor()方法确认key要插入的位置

2、进行数据校验，如果发现需要删除节点，则进行辅助删除，如果其他线程改变了跳跃表，则进行重试或遍历查找合适的位置。

3、如果跳跃表中已经存在该key,则根据onlyIfAbsent 确定是否覆盖旧值。

4、生成节点，插入到最底层的数据链表中。

5、根据随机值确定是否创建索引层，如果不需要则返回，否则执行第6步

6、如果需要创建的索引层超过最大的level,则需要创建HeadIndex 索引层，否则只需要创建Index 索引层即可。

7、从head 开始进行遍历，将每一层的新添加的Index索引层进行连接(这个可以结合上面跳跃表**连接索引层**图示来理解)。

### 查找数据

```
public V get(Object key) {
    return doGet(key);
}
```
```
private V doGet(Object key) {
    if (key == null)
        throw new NullPointerException();
    Comparator<? super K> cmp = comparator;
    outer: for (;;) {
        // 通过findPredecessor 方法查找其前驱节点
        for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
            Object v; int c;
            if (n == null) //不存在该key
                break outer;
            Node<K,V> f = n.next;
            //其它线程更新了跳跃表，重试
            if (n != b.next)                // inconsistent read
                break;
            //需要删除数据    
            if ((v = n.value) == null) {    // n is deleted
                n.helpDelete(b, f); //协助删除
                break;
            }
            //b会被删除，重来
            if (b.value == null || v == n)  // b is deleted
                break;
            // 找到了    
            if ((c = cpr(cmp, key, n.key)) == 0) {
                @SuppressWarnings("unchecked") V vv = (V)v;
                return vv;
            }
            if (c < 0) //不存在该key
                break outer;
            // 其它线程添加了数据，向后继续遍历查找    
            b = n;
            n = f;
        }
    }
    return null;
}
```
查找数据比较简单，先通过findPredecessor 查找其前驱，然后顺着right一直往右找即可，同时在这个过程中，如果发现某个节点需要删除，则需要进行辅助删除，如果发现跳跃表数据结构被其它线程改变，会重新尝试获取其前驱。

### 删除数据
ConcurrentSkipListMap 支持并发操作，因此在删除的时候需要注意，因为在删除的同时，其它线程可能在该位置上进行数据插入，这样很容易造成数据的丢失，这个我们在前面的阻塞队列SynchronousQueue 也遇到类似的问题，在SynchronousQueue 中用了一个cleanMe标记需要删除的节点的前驱，在ConcurrentSkipListMap 中也是类似的机制，会在删除后面添加一个**特殊的节点进行标记**，然后再进行整体的删除，如果不进行标记，那么如果正在删除的节点，可能其它线程正在此节点后面添加数据，造成数据丢失.
```
public V remove(Object key) {
    return doRemove(key, null);
}
```
调用doRemove()方法，doRemove有两个参数，一个是key，另外一个是value，所以doRemove方法即提供remove key，也提供同时满足key-value。
```
//指定key 和value删除相应的节点
final V doRemove(Object key, Object value) {
    if (key == null)
        throw new NullPointerException();
    Comparator<? super K> cmp = comparator;
    outer: for (;;) {
        //查找其前驱
        for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
            Object v; int c;
            if (n == null) // 不存在该key
                break outer;
            Node<K,V> f = n.next;
            // 其它线程修改了跳跃表数据结构
            if (n != b.next)                    // inconsistent read
                break;
            //节点n 需要被删除，进行协助删除 ，然后再重试   
            if ((v = n.value) == null) {        // n is deleted
                n.helpDelete(b, f);
                break;
            }
            //b即将也被删除
            if (b.value == null || v == n)      // b is deleted
                break;
            // 不存在该key    
            if ((c = cpr(cmp, key, n.key)) < 0)
                break outer;
            if (c > 0) { //向后继续遍历查找
                b = n;
                n = f;
                continue;
            }
            //key 相等，value 不相等，退出
            if (value != null && !value.equals(v))
                break outer;
            //逻辑删除，设置value=null    
            if (!n.casValue(v, null)) 
                break;
            //先添加删除标记，然后再进行删除操作    
            if (!n.appendMarker(f) || !b.casNext(n, f))
                findNode(key);                  // retry via findNode
            else {
                // 清除索引层
                findPredecessor(key, cmp);      // clean index
                 //该层已经没有节点，删掉该层
                if (head.right == null)
                    tryReduceLevel();
            }
            @SuppressWarnings("unchecked") V vv = (V)v;
            return vv;
        }
    }
    return null;
}
```
调用findPredecessor()方法找到前驱节点，然后通过向右遍历，然后比较，找到后利用CAS把value替换为null（这样其它线程可以感知到这个节点状态，协助完成删除操作），添加删除标记节点，添加成功后，再执行删除操作（因为添加标记的cas和其它线程在这个节点后面添加数据的cas 只有一个能成功，所以可以避免数据丢失），如果该层已经没有了其它节点，调用tryReduceLevel()方法把这层移除掉。

添加标记节点：
```
boolean appendMarker(Node<K,V> f) {
    return casNext(f, new Node<K,V>(f));
}
```
协助删除节点：
```
void helpDelete(Node<K,V> b, Node<K,V> f) {

    if (f == next && this == b.next) {
        //如果没有添加标记节点，那么添加标记节点
        if (f == null || f.value != f) // not already marked
            casNext(f, new Node<K,V>(f));
        else
            b.casNext(this, f.next); //执行删除操作
    }
}
```

### size统计
ConcurrentSkipListMap的size()操作和ConcurrentHashMap不同，它并没有维护一个全局变量来统计元素的个数，每次调用该方法的时候都需要去遍历，在并发环境下，该size值只是一个瞬间值。

```
public int size() {
    long count = 0;
    //遍历数据链表
    for (Node<K,V> n = findFirst(); n != null; n = n.next) {
        if (n.getValidValue() != null)
            ++count;
    }
    return (count >= Integer.MAX_VALUE) ? Integer.MAX_VALUE : (int) count;
}
```
```
final Node<K,V> findFirst() {
    for (Node<K,V> b, n;;) {
        //没有数据
        if ((n = (b = head.node).next) == null)
            return null; 
        if (n.value != null)
            return n;
        //协助删除n这个节点    
        n.helpDelete(b, n.next);
    }
}
```
调用findFirst()方法找到第一个Node（如果有需要删除的节点，则进行协助删除），然后利用node的next去统计。最后返回统计数据，最多能返回Integer.MAX_VALUE

### 总结
ConcurrentSkipListMap 是一个基于 Skip List 实现的并发安全, 非阻塞读/写/删除 的 Map（key和value 不能为null）,它的value是有序存储的, 其内部是由纵横链表组成, 通过空间换时间的方式，使得链表也可以实现类似的二分查找，相比平衡树，其编程复杂度大大减小.