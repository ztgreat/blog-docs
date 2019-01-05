HashMap应该是使用的一个频率很高的一个集合了，平时用得很多，但是了解并不深入，一起来看看HashMap的结构实现和功能原理。

### HashMap（jdk 1.8）
HashMap是Java的Map家族中一个普通成员,它根据键的hashCode值存储数据，具有很快的访问速度，但遍历顺序却是不确定的,也就是说插入顺序和遍历顺序没有什么关系的，究其原因在于元素的存储结构，后面我们将会学习到，HashMap最多只允许一条记录的键为null，允许多条记录的值为null。HashMap非线程安全,如果需要满足线程安全，可以用 Collections的synchronizedMap方法使HashMap具有线程安全的能力，或者使用ConcurrentHashMap。

### 继承体系

![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171022154502593.png)

HashMap 继承AbstractMap， AbstractMap 提供了 Map 的基本实现。

HashMap 实现了Cloneable接口，支持clone()方法，可以被克隆。 

HashMap 实现了Serializable接口，可以被序列化。

### 数据结构

从HashMap的名字我们就可以知道，该集合采用了hash的方式，hash 通常来说会有一张hash表（一般是数组），hash表也称散列表,简单来说，就是对每个存储元素生成一个索引，把元素放到指定的索引位置上，查找元素的时候更新元素索引来取值。
生成hash索引的方法叫做hash 函数，不同的元素可能生成相同的索引，这种情况就是hash 冲突（碰撞），解决hash 冲突的方法一般有：

1、开放定址法

2、链地址法

HashMap 采用的是链地址法，也就是同一hash值的元素都存储在一个链表里（jdk 1.7）。
![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171024231520599.png)

在jdk 1.8 中 HashMap是数组+链表+红黑树实现的。
![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171024231757855.png)

对HashMap 存储结构有了了解过后，来看看每个具体的存储单元。

```
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;    //hash值，用来定位数组索引位置
    final K key;       //元素 key
    V value;           // 元素 value
    Node<K,V> next;   //链表的下一个node

    Node(int hash, K key, V value, Node<K,V> next) { ... }
    public final K getKey(){ ... }
    public final V getValue() { ... }
    public final String toString() { ... }
    public final int hashCode() { ... }
    public final V setValue(V newValue) { ... }
    public final boolean equals(Object o) { ... }
}
```
Node是HashMap的一个内部类，实现了Map.Entry接口，本质是就是一个映射(键值对)。
接下来再来了解下HashMap的几个字段
```
//hash 表
transient Node<K,V>[] table;

/**
 *The next size value at which to resize (capacity * load factor).
 */
int threshold;

/**
 * The default initial capacity - MUST be a power of two. hash表的默认大小
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
/**
 * The maximum capacity, used if a higher value is implicitly specified
 * by either of the constructors with arguments.
 * MUST be a power of two <= 1<<30. hash表的最大的size
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * The load factor used when none specified in constructor. 负载因子
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;
/**
 * The number of key-value mappings contained in this map.
 */
transient int size;
```

首先，table 就是我们所说的hash表，表的默认大小（length）是16，Load factor为负载因子(默认值是0.75)，threshold是HashMap所能容纳的最大数据量的Node(键值对)个数。threshold = length * Load factor。也就是说，在数组定义好长度之后，负载因子越大，所能容纳的键值对个数越多。

size是HashMap中**实际存在**的键值对数量。

在HashMap中，哈希表的长度length大小必须为2的n次方(一定是合数)，这是一种非常规的设计，常规的设计是把表的大小设计为素数。相对来说素数导致冲突的概率要小于合数，HashMap采用这种非常规设计，主要是为了在取模和扩容时做优化，同时为了减少冲突，HashMap定位哈希表索引位置时，也加入了高位参与运算的过程。

这里存在一个问题，即使负载因子和Hash算法设计的再合理，如果数据增多，也免不了会出现拉链过长的情况，一旦出现拉链过长，则会严重影响HashMap的性能。于是，在Jdk1.8版本中，对数据结构做了进一步的优化，引入了红黑树。而当链表长度太长（默认超过8）时，链表就转换为红黑树，利用红黑树快速增删改查的特点提高HashMap的性能，红黑树不是本文的重点，如果有兴趣可以参考我前面写红黑树（[亲自动手画红黑树](http://blog.ztgreat.cn/article/12)）

```
//链表转成红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;
//红黑树转为链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;
//存储方式由链表转成红黑树的容量的最小阈值
static final int MIN_TREEIFY_CAPACITY = 64;
```
当hash 表中同一个位置的拉链长度 大于等于8时，会将拉链转换为红黑树，如果红黑树的节点数少于等于6时，会将红黑树转换为链表结构。

### 构造方法
1、默认构造

```
public HashMap() {
    //负载因子为默认的0.75
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

2、指定容量
```
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```

指定容量来初始化HashMap。

3、指定容量和负载因子

```
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                initialCapacity);
    //初始容量不能 > 最大容量值，HashMap的最大容量值为2^30
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
```

在前面我们说过HashMap 中的hash 表的大小一定是2^n。我们通过手动自定初始化大小肯定没有任何限制的，那么也就是说实际分配的hash表的大小不一定就是我们指定的大小，我们来看看tableSizeFor 方法就明白了。

```
/**
 * Returns a power of two size for the given target capacity.
 */
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

从注释中我们就可以知道，该方法就是在计算hash表的大小，计算的结果是 大于等于我们指定的大小 的值（这个值是满足条件中的最小值），而且满足是2^n的关系。
因此当我们初始化HashMap指定大小时，并不是真正就用的这个大小，而是会计算一个新的符合条件的值来作为hash表的大小,并将threshold 设置为这个值。

4、通过map 集合初始化
```
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

### 索引值计算
我们知道了HashMap的存储结构，知道了通过索引来存取值，那么这个索引是如何计算的呢？

```
//得到hash 值
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
//得到元素具体存放的位置
h&(length-1)
```

我们可以发现，当key=null时，也是有hash值的，是0，所以，HashMap的key是可以为null的。
对于hash值的计算，首先计算出key的hashCode()为h，然后与h无条件右移16位后的二进制进行按位异或(^)得到最终的hash值，再通过h & (table.length -1)来得到该对象的保存位，而HashMap底层数组的长度总是2的n次方，当length总是2的n次方时，h& (length-1)运算等价于对length取模，也就是h%length，但是&比%具有更高的效率。
如此一来我们大概知道HashMap的索引计算方法，接下来就从方法入手，看看HashMap是怎么存数据的，以及如何取数据。

### put 方法

```
/**
 * Associates the specified value with the specified key in this map.
 * If the map previously contained a mapping for the key, the old
 * value is replaced.
 */
public V put(K key, V value) {
    // 对key的hashCode()做hash,调用putVal 方法
    return putVal(hash(key), key, value, false, true);
}
```

从方法注释中我们知道，如果map 中含有了该key ,那么新的value 将会替代旧的value。

```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // tab为空则创建
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 计算存储的index，如果该位置没有元素，则构造节点并放在该位置上
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else { //产生了hash 冲突，该位置上已经有了元素。
        Node<K,V> e; K k;
        // 节点key存在，直接覆盖value
        if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
            // 存储的节点是红黑树节点，说明用的红黑树来解决的hash冲突    
        else if (p instanceof TreeNode)
            //将节点插入到红黑树中
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {  //是通过链表来解决hash 冲突的
            //遍历链表，将节点插入到链表尾
            for (int binCount = 0; ; ++binCount) {
                //表尾
                if ((e = p.next) == null) {
                    //插入节点
                    p.next = newNode(hash, key, value, null);
                    //如果此时链表中的节点的个数大于等于了阀值
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash); //将链表转换为红黑树
                    break;
                }
                if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //存在该key,根据情况看是否更新value
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);  //for LinkedHashMap
            return oldValue;
        }
    }
    ++modCount;
    // 超过最大容量 就扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict); //for LinkedHashMap
    return null;
}
```

1、如果table 为null,执行resize()进行扩容；

2、根据键值key计算hash值得到插入的数组索引i，如果table(i)==null，直接新建节点添加

3、判断table(i)的首个元素是否和插入的元素拥有相同的key，如果相同直接覆盖value

4、判断table(i) 是否为treeNode，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对。

5、遍历table(i)，判断链表长度是否大于等于阀值，如果是则把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value。

6、插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。

### 扩容
当向HashMap对象里不停的添加元素，而HashMap对象内部的数组无法装载更多的元素时，对象就需要扩大数组的长度，以便能装入更多的元素，其实现原理就是重新生成一个更大的数组（也就是hash表），然后将原来的hash表中的数据放到新表中。

将数组重就表中发到新表中，不像一般数组扩容那样，直接复制就可以了，在前面我们知道，每个元素放的位置是很讲究 的，不能乱放，会根据他们的hash 值和hash表的大小，确定存储的位置，因此现在把数据重新表放到新表中国，相当于要重新进行一遍put操作（计算索引值，然后将元素放到hash 表中）。
在jdk 1.7 中需要为每个元素，重新计算一遍索引值，但是在jdk 1.8 中发生了点小变化。

下面借用[美团点评技术团队（Java 8系列之重新认识HashMap）](https://tech.meituan.com/java-hashmap.html)一文中的部分解释:

在HashMap 中，**扩容后的大小是原来的2倍**（这个我们在后面会看到），所以，元素的位置要么是在原位置，要么是在原位置+n的位置（n 为原table的大小）。看下图可以明白这句话的意思，n为table的长度，图（a）表示扩容前的key1和key2两种key确定索引位置的示例，图（b）表示扩容后key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希与高位运算结果。
![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171024234842075.png)

回顾一下是如何计算元素的存储索引的：
```
//得到hash 值
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
//得到元素具体存放的位置
h&(length-1)
```

从上面代码中我们就可以知道，扩容前后，对元素的hash值的计算是没有任何影响的，而对于索引值有影响的便是h&(length-1)了（也就是hash表的大小），扩容后的hash 表大小是阔扩容先的两倍，而表大小又都是2^n这种，那么在二进制表示的length 大小中，只有一位是1，其余是0，同时扩容的length ,就是把原length 左移一位，对于length -1 就是最高为1 变为0 ，其后面的二进制位全是1，那么扩容后的length-1 比扩容前的length-1，在高位上比原来多一个1，那么这样和h相与后，产生变化的就是那个“多的那个1”与后的结果，“多的那个1”可能是0，可能是1，如果是0，那么索引值和原来一致，否则是原来的2倍，可以参考上面图进行理解。

好，有了上面的认识，我们在来看resize 源码（代码有点长，但是还是很好理解的）：

```
/**
 * Initializes or doubles table size.  If null, allocates in
 * accord with initial capacity target held in field threshold.
 * Otherwise, because we are using power-of-two expansion, the
 * elements from each bin must either stay at same index, or move
 * with a power of two offset in the new table.
 *
 * @return the table
 */
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    //原hash表length
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //原threshold
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        //如果原来的length 超过最大值就不再扩充了
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 新hash表大小 扩容成是原来的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // oldCap=0,HashMap初始化
        newCap = oldThr;
    else {               // 用默认值初始化话HashMap
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        //计算 threshold
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    //创建新table
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        //遍历table 进行搬数据
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            //当前位置有元素
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                //该节点后面没有其他数据，不管是链表节点还是树节点
                if (e.next == null)
                    //将节点放入新表中，这里重新计算了一次索引
                    newTab[e.hash & (newCap - 1)] = e;
                    //e.next != null,有多个元素需要搬运，并且是树节点
                else if (e instanceof TreeNode)
                    //调整节点
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // 重新整理链表
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        //判断扩容后高位的1 对索引值有无影响，==0 表示没影响，重新放到链表loHead中
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {// ==1索引值变为原索引（j）的2倍，重新放到链表hiHead中
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 将索引值没变的链表当到新表中索引为j的位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    //将变化的链表放到原索引(j)的2倍位置处
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

整个过程理解应该不难，依次对hash 表中的元素及“链”上的元素进行处理，如果是树节点，则进行调整（红黑树部分后面再分析），如果是链表，则对链表进行处理，将链表分成两部分，一部分就是索引值没有变的元素，一部分就是索引值变为原来2倍的元素，最后再讲这两部分链表放到各自该放的位置上。


### get 方法
通过get 方法可以从HashMap 中取出元素，一起来看看整个过程

```
public V get(Object key) {
    Node<K,V> e;
    //通过key ,计算hash 值，查找存储的节点
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

```
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
        //hash 表中的节点和我们查找的元素有相同hash 值并且key相同，那么就是我们需要查找的元素
        if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //如何hash 表中没有，就需要从"链"中查找    
        if ((e = first.next) != null) {
            // 这个"链" 是树节点
            if (first instanceof TreeNode)
                //在红黑树中查找该元素
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //这个"链"是链表    
            do {
                //在链表中查找元素
                if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null; //没查找返回null
}
```

这个查找就很简单

1、通过hash 值找到其在hash 表中存储的索引位置

2、然后对该索引位置上的节点进行判断，如果是树节点，则在执行红黑树查找

3、链表节点，则在链表中进行查找

4、查找到返回该节点，否则返回null。

### remove 方法

```
public V remove(Object key) {
    Node<K,V> e;
    //从hash表中删除 指定元素
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
}
```

删除肯定还是需要查找，因此主要还是查找的过程
```
/**
 * Implements Map.remove and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to match if matchValue, else ignored
 * @param matchValue if true only remove if value is equal
 * @param movable if false do not move other nodes while removing
 * @return the node, or null if none
 */
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        //索引位置上存储的就是该元素
        if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {  //该索引位置上有"链"
            if (p instanceof TreeNode) //树节点
                //从树节点中查找该元素节点
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                //链表中查找该元素节点
                do {
                    if (e.hash == hash &&
                            ((k = e.key) == key ||
                                    (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                (value != null && value.equals(v)))) {
            if (node instanceof TreeNode) //树节点
                //从树中删除该节点
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                //索引位置上就是该元素    
            else if (node == p)
                //将该索引位置上的节点指向该元素所在链上的后继。
                tab[index] = node.next;
            else //该元素存储在对于索引的链表上
                //链表中删除该元素
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

删除元素之前，需要查找元素的位置

1、如果元素在hash 表上，那么从hash 表中移除

2、如果元素在hash 表对于索引的"链"上

3、如果"链"是树节点，则查找该节点，然后删除

3、如果"链"是链表，则在链表中删除该节点。

### 序列化
因为HashMap中存在非物理上连续的数据，因此需要我们自己“手动”序列化。

1、writeObject
```
private void writeObject(java.io.ObjectOutputStream s)
        throws IOException {
    // hash 表的容量
    int buckets = capacity();
    // Write out the threshold, loadfactor, and any hidden stuff
    s.defaultWriteObject(); //先进行默认的序列化
    s.writeInt(buckets);  //写入hash 表大小
    s.writeInt(size);  //写入键值对个数
    internalWriteEntries(s); //写入键值对
}
```

```
// Called only from writeObject, to ensure compatible ordering.
void internalWriteEntries(java.io.ObjectOutputStream s) throws IOException {
    Node<K,V>[] tab;
    //遍历hash 表
    if (size > 0 && (tab = table) != null) {
        for (int i = 0; i < tab.length; ++i) {
            //遍历hash 表每个位置上的"链"
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                s.writeObject(e.key); //写入key
                s.writeObject(e.value); //写入value
            }
        }
    }
}
```

在序列化工程做，会遍历hash 表每个位置上的"链"，然后依次写入数据，我们知道hash 表每个位置上的“链” 有可能是链表，也可能是红黑树，但是在序列化中，看到的似乎都是遍历链表的方式，这是为什么呢？，这个我们在红黑其红黑树中的部分来分析。

2、readObject

```
private void readObject(java.io.ObjectInputStream s)
        throws IOException, ClassNotFoundException {
    // Read in the threshold (ignored), loadfactor, and any hidden stuff
    s.defaultReadObject();  //默认反序列化
    reinitialize();
    //默认反序列化后会得到 loadFactor
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new InvalidObjectException("Illegal load factor: " +
                loadFactor);
    s.readInt();  // hash 表容量
    int mappings = s.readInt(); // 键值对个数
    if (mappings < 0)
        throw new InvalidObjectException("Illegal mappings count: " +
                mappings);
    else if (mappings > 0) { // (if zero, use defaults)
        // Size the table using given load factor only if within
        // range of 0.25...4.0
        float lf = Math.min(Math.max(0.25f, loadFactor), 4.0f);
        float fc = (float)mappings / lf + 1.0f;
        // 重新确定hash 表的容量
        int cap = ((fc < DEFAULT_INITIAL_CAPACITY) ?
                DEFAULT_INITIAL_CAPACITY :
                (fc >= MAXIMUM_CAPACITY) ?
                        MAXIMUM_CAPACITY :
                        tableSizeFor((int)fc));
        float ft = (float)cap * lf;
        threshold = ((cap < MAXIMUM_CAPACITY && ft < MAXIMUM_CAPACITY) ?
                (int)ft : Integer.MAX_VALUE);
        Node<K,V>[] tab = (Node<K,V>[])new Node[cap];
        table = tab;

        // Read the keys and values, and put the mappings in the HashMap
        for (int i = 0; i < mappings; i++) {
            K key = (K) s.readObject(); //读取key
            V value = (V) s.readObject(); //读取value
            putVal(hash(key), key, value, false, false);
        }
    }
}
```

### 红黑树
从前面我们知道，hash表中可能存储的是链表结构，也可能存储的是树结构，至于链表部分我们已经很清楚了，接下来来看看树部分

#### 红黑树节点
红黑树节点TreeNode 是HashMap 中的一个静态内部类

```
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    //父节点
    TreeNode<K,V> parent;
    //左孩子
    TreeNode<K,V> left;
    //右孩子
    TreeNode<K,V> right;
    //前驱链节点
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    //.... 省略后序方法
}
```

TreeNode 继承了LinkedHashMap的Entry,而LinkedHashMap的Entry 继承至HashMap的Node，Node节点的定义在前面我们展示过，这里就不在列出来了。
```
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

从红黑树节点定义可以看出来，TreeNode 不仅是树形结构，里面还有链表节点的属性，那么是不是除了将这些节点构成一颗树，同时也构成一个链表呢，这个我们从后面的源码中来寻找答案。

在前面的putVal 方法中，有如下的代码：
```
for (int binCount = 0; ; ++binCount) {
    if ((e = p.next) == null) {
        p.next = newNode(hash, key, value, null);
        //如果此时链表中的节点的个数大于等于了阀值
        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            treeifyBin(tab, hash);  //将链表转化为红黑树
        break;
    }
    if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
        break;
    p = e;
}
```

那么我们就来看看treeifyBin 方法，参数tab 则是hash 表，hash是put元素的hash值。

```
/**
 * Replaces all linked nodes in bin at index for given hash unless
 * table is too small, in which case resizes instead.
 */
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //如果tab为null 或者tab的容量小于  存储方式由链表转成红黑树的容量的最小阈值那么则进行扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
        // 该元素应该存在tab 表中index 位置上，该位置如果被被占据，则才进行转换
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        // 遍历链表，将链表元素转化成TreeNode链
        TreeNode<K,V> hd = null, tl = null;
        do {
            //根据节点e 生成一个TreeNode 节点p
            TreeNode<K,V> p = replacementTreeNode(e, null);
            //下面则是链表操作，构成一个TreeNode链表（双向链表）
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        //下面才是真正的转换，把TreeNode 链表转换为红黑树
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}

// For treeifyBin
TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    return new TreeNode<>(p.hash, p.key, p.value, next);
}
```

在将tab(index)链上的链表转换为红黑树之前，需要把Node链表转换为TreeNode 链表，这样TreeNode 不仅是红黑树节点，也相当于是链表节点，这样处理的好处就是，既可以用树的方式遍历红黑树，也可以用链表的方式遍历红黑树，这个时候是不是就明白了序列化中都用的链表遍历方式序列化每个元素的呢。

接下来我继续看真正把链表转换成红黑树的过程
```
/**
 * Forms tree of the nodes linked from this node.
 * @return root of tree
 */
final void treeify(Node<K,V>[] tab) {
    //树根
    TreeNode<K,V> root = null;
    // 遍历TreeNode链
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        // 设置root节点
        if (root == null) {
            x.parent = null;
            x.red = false;  //root 节点为黑色
            root = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            // 循环查找当前节点插入的位置并添加节点
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;
                //通过hash 值来比较，也就是说元素的hash值用来表示红黑树中节点数值大小
                if ((ph = p.hash) > h)
                    // 当前节点值小于根节点，dir = -1
                    dir = -1;
                else if (ph < h)
                    // 当前节点值大于根节点, dir = 1
                    dir = 1;

                    // 当前节点的值等于根节点值。    
                else if ((kc == null &&
                        (kc = comparableClassFor(k)) == null) ||
                        (dir = compareComparables(kc, k, pk)) == 0)
                /**
                 *如果当前节点实现Comparable接口，调用compareTo比较大小并赋值dir
                 *如果当前节点没有实现Comparable接口，compareTo结果等于0，则调用tieBreakOrder比较大小
                 *tieBreakOrder本质是通过比较k与pk的hashcode
                 */
                    dir = tieBreakOrder(k, pk);

                //当前"根节点"(父节点)
                TreeNode<K,V> xp = p;
                // 如果当前节点小于根节点且左子节点为空 或者  当前节点大于根节点且右子节点为空，直接添加子节点
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    //前节点小于根节点
                    if (dir <= 0)
                        xp.left = x; //成为当前"根节点"的左孩子
                        //前节点大于根节点
                    else
                        xp.right = x;
                    // 插入节点后，需要进行红黑树的平衡操作（满足红黑树性质）    
                    root = balanceInsertion(root, x);
                    //退出循环，继续向红黑树添加下一个元素
                    break;
                }
            }
        }
    }
    //保证红黑树根节点是数组中该index的第一个节点，也就是tab[index]要等于 root
    moveRootToFront(tab, root);
}
```

遍历TreeNode 链，如果是小于当前比较节点的，则放在左边，否则放在右边，就是一般的排序二叉树的插入过程，当插入元素后，再执行红黑树的平衡操作，以满足红黑是性质，这里我们就不在细看红黑树的平衡操作了，感兴趣的可以看看我前面写的博客（[亲自动手画红黑树](http://blog.ztgreat.cn/article/12)）
moveRootToFront 方法，则是保证红黑树根节点是数组中该index的第一个节点(红黑树平衡操作可能会调整根节点)

```
static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
    int n;
    if (root != null && tab != null && (n = tab.length) > 0) {
        int index = (n - 1) & root.hash;
        TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
        if (root != first) {
            Node<K,V> rn;
            //数组中该index的第一个节点== root
            tab[index] = root;
            //root 前驱
            TreeNode<K,V> rp = root.prev;
            if ((rn = root.next) != null)
                ((TreeNode<K,V>)rn).prev = rp;  //root后继的前驱指向root前驱
            if (rp != null)
                rp.next = rn; //root前驱的后继指向root后继
            if (first != null)
                first.prev = root; //firt的前驱指向root
            root.next = first; //root 后继指向first
            root.prev = null;  //root 变成链表头
        }
        assert checkInvariants(root);
    }
}
```

通过下面图就很容易理解了：
![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171029123939489.png)
实线是树之间的连接，虚线是链表之间的连接（分别表示前驱和后继指针），在将TreeNode链表转换成红黑树之前，first 是链表头，转换成红黑树后，root是红黑树根节点，两者可能不是同一个节点。
以前通过first 才能通过链表方式完全遍历整颗树，现在通过root 使用链表可以完整遍历整颗树。

### 总结

1、在jdk 1.8 中 HashMap是数组+链表+红黑树实现的。

2、HashMap 中的容量总是2的n次方 这种形式，HashMap 运行存储的key和value 为null，当然只有一个key能为null，多个value 可以为null。

3、从存储结构中可以知道，HashMap的插入顺序和遍历顺序肯定是不一样的，因此HashMap 是无序的。

4、HashMap扩容时会整体搬移数据，虽然不用重新计算索引值，但是还是比较消耗性能，所以当在使用HashMap的时候，估算map的大小，初始化的时候给一个大致的数值，避免map进行频繁的扩容。

5、HashMap是**线程不安全的**，不要在并发的环境中同时操作HashMap，建议使用ConcurrentHashMap。