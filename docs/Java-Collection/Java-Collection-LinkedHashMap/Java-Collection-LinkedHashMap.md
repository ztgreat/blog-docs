前面我们分析了HashMap，HashMap在jdk 1.8中是红黑树和链表实现的，具有很好的查询性能，但是HashMap存储元素是无序，这里的无序指的是插入的顺序和遍历的顺序不一致，无法预测遍历的顺序，今天我们来看Map的另一个实现类LinkedHashMap。
LinkedHashMap继承自HashMap，同时也维护了元素的插入顺序，今天我们来看看它是如何实现的。

LinkedHashMap 是基于HashMap，HashMap我们在上一篇文章中已经分析过，因此本文不会再对HashMap的部分进行分析，只会分析LinkedHashMap 的部分。

[Java集合之HashMap源码分析](http://blog.ztgreat.cn/article/20)

###  LinkedHashMap（jdk 1.8）
从名称上看，LinkedHashMap 是HashMap,同时它具有Link 功能，可以提供可以预测的迭代访问，即**按照插入序 (insertion-order) 或访问序 (access-order) 来对哈希表中的元素进行迭代**。
插入顺序就是元素依次放入容器中的顺序，那么访问顺序又是什么呢?
**访问序模式 就是尾部节点是最近一次被访问的节点 (least-recently)，而头部节点则是最远访问 (most-recently) 的节点**。因而在决定失效缓存的时候，将头部节点移除即可。
### 继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171110140339779.png)

可以看到LinkedHashMap 是HashMap的子类。因此它有HashMap的所有功能。
### 数据结构
分析其功能，还必须先看看它的数据结构

```
//存储节点定义
 static class Entry<K,V> extends HashMap.Node<K,V> {
     Entry<K,V> before, after;
     Entry(int hash, K key, V value, Node<K,V> next) {
         super(hash, key, value, next);
     }
 }
 //链表头
 transient LinkedHashMap.Entry<K,V> head;

//链表尾
 transient LinkedHashMap.Entry<K,V> tail;
 //迭代顺序， true 使用最近被访问的顺序， false为插入顺序
 final boolean accessOrder;
```
为了实现双向链表，LinkedHashMap 的节点在父类的基础上增加了 before/after 引用，并且使用 head 和 tail 分别保存双向链表的头和尾。同时，增加了一个标识来保存 LinkedHashMap 的迭代顺序是插入序还是访问序。

```
 //HashMap Node 定义
 static class Node<K,V> implements Map.Entry<K,V> {
     final int hash;
     final K key;
     V value;
     Node<K,V> next;
  }
```
 HashMap 的节点中存在 next 引用，可以将每个桶中的元素都当作一个单链表看待；LinkedHashMap 的每个桶中当然也保留了这个单链表关系，不过这个关系由父类进行管理，LinkedHashMap 中只会对双向链表的关系进行管理。

### 构造方法
1、默认构造

```
 public LinkedHashMap() {
     super();
     accessOrder = false;
 }
```
调用HashMap中默认的构造方法，同时设置LinkedHashMap维护的元素的顺序为元素插入的顺序。

2、指定初始容器大小

```
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}
```
3、指定初始容器大小和负载因子

```
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}
```
### put 方法
既然LinkedHashMap 会维护插入元素的顺序，那么这些操作是在哪里进行的呢，这里我们就先从put 方法开始跟踪分析，主要是我个人就是这样分析的，因此不太喜欢直接将一系列方法直接摆出来分析，这样没有逻辑感，这种顺藤摸瓜的方式容易理清思路。

在LinkedHashMap 中并没有重写HashMap的方法，直接是调用的HashMap的put 方法。

###  newNode方法

在 LinkedHashMap 中重写了HashMap的newNode 方法，因此在put 过程中，在生成新节点时，将会对双向链表进行操作。

```
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}
```
生成一个LinkedHashMap.Entry的节点，同时后面调用了linkNodeLast 方法，而这个方法就是讲新生成的节点插入到链表末尾中。

### linkNodeLast 方法
```
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```
这个方法很简单，就是对双向链表的操作，将节点p链接到链表的末尾。

### afterNodeInsertion 方法
在HashMap 中，将元素插入容器后，会调用该方法，该方法在HashMap 中没有实现，专门供LinkedHashMap使用的，在LinkedHashMap有重写该方法。

```
void afterNodeInsertion(boolean evict) { // possibly remove eldest
     LinkedHashMap.Entry<K,V> first;
     if (evict && (first = head) != null && removeEldestEntry(first)) {
         K key = first.key;
         removeNode(hash(key), key, null, false, true);
     }
 }
```
该方法主要功能就是判断是否需要删除最"老"的元素（这里最老有两种：最先插入，最久未访问，这个取决于accessOrder 的值），双链表的头head 是最“老”的元素，其尾是最"年轻"的元素。

```
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
     return false;
 }
```
默认是不会删除最“老”的元素，如果自己需要（例如将LinkedHashMap 做缓存，需要删除最久未被访问的数据），可以自己重写这个方法来实现自己的逻辑。

removeNode 是HashMap中的方法，就是从容器中删除节点，在removeNode  中删除了节点后，会调用afterNodeRemoval 方法，该方法在LinkedHashMap 中实现，通过该方法可以从双向链表中进行删除该节点。

```
void afterNodeRemoval(Node<K,V> e) { // unlink
   LinkedHashMap.Entry<K,V> p =
       (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
   p.before = p.after = null;
   if (b == null)
       head = a;
   else
       b.after = a;
   if (a == null)
       tail = b;
   else
       a.before = b;
}
```
通过以上方法我们知道，在将元素插入到容器中时，会将节点也添加到LinkedHashMap 中的双向链表中，同时根据产生判断是否会移除最“老”元素节点。

### get 方法

LinkedHashMap 重写了父类的get 方法
```
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```
重写只是为了增加自己的逻辑操作，实际上还是调用了父类的getNode方法获取节点，在get后，如果设置的是访问顺序（也就是accessOrder 为true）,那么就会调整该访问元素的位置（移动到链表尾）
```
void afterNodeAccess(Node<K,V> e) { // move node to last
     LinkedHashMap.Entry<K,V> last;
     if (accessOrder && (last = tail) != e) {
         LinkedHashMap.Entry<K,V> p =
             (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
         p.after = null;
         if (b == null)
             head = a;
         else
             b.after = a;
         if (a != null)
             a.before = b;
         else
             last = b;
         if (last == null)
             head = p;
         else {
             p.before = last;
             last.after = p;
         }
         tail = p;
         ++modCount;
     }
 }
```
afterNodeAccess 将节点e 移动到链表末尾，这个操作不难，因此就不细说了。

### remove 方法

LinkedHashMap 调用的是HashMap中的remove 方法，在HashMap的remove 方法中，在移除节点后，会调用afterNodeRemoval 方法，该方法在LinkedHashMap  中进行了重写（实现了从容器中移除的节点也从链表中移除）

```
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```
将节点e 从链表中进行移除。

### LinkedHashMap 容器调整 
我们知道，HashMap 中的元素可能会在单链表和红黑树节点之间进行转换（当链表或者红黑树的个数操作了阀值），LinkedHashMap 中当然也是一样，不过在转换时还要调用 transferLinks 来改变双向链表中的连接关系。
在 LinkedHashMap 中重写了HashMap中的replacementNode，newTreeNode，replacementTreeNode 方法，下面一起来简单看看。

```
TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
     TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
     linkNodeLast(p);
     return p;
    }
```
在生成红黑树节点的时候也会将节点插入到LinkedHashMap 的双向链表中这个和生成HashMap中的链表节点是一样的道理。

```
Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
     LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
     LinkedHashMap.Entry<K,V> t =
         new LinkedHashMap.Entry<K,V>(q.hash, q.key, q.value, next);
     transferLinks(q, t);
     return t;
 }
```
在替换节点的时候，也会涉及到对LinkedHashMap  中的双向链表进维护，这个需要调用 transferLinks 方法。

```
 // apply src's links to dst
 private void transferLinks(LinkedHashMap.Entry<K,V> src,
                            LinkedHashMap.Entry<K,V> dst) {
     LinkedHashMap.Entry<K,V> b = dst.before = src.before;
     LinkedHashMap.Entry<K,V> a = dst.after = src.after;
     if (b == null)
         head = dst;
     else
         b.after = dst;
     if (a == null)
         tail = dst;
     else
         a.before = dst;
 }
```
将节点 src 替换成dst节点，就是改变一下节点指针指向就好了。

```
TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
    TreeNode<K,V> t = new TreeNode<K,V>(q.hash, q.key, q.value, next);
    transferLinks(q, t);
    return t;
}
```
replacementTreeNode 和上面的方法类似，这里就不展开了。

### 遍历及迭代器

在HashMap中没hash表索引上的位置 可能是链表也可能是红黑树，红黑树中也维护了一个next域，将红黑树中的节点链接起来，但是节点之间的顺序是没有意义的。
在 LinkeHashMap 的所有的节点都在一个双向链表中，因而可以通过该双向链表来遍历所有的 Entry，可以**按照插入序 (insertion-order) 或访问序 (access-order) 来对哈希表中的元素进行迭代**。

```
abstract class LinkedHashIterator {
    LinkedHashMap.Entry<K,V> next;
    LinkedHashMap.Entry<K,V> current;
    int expectedModCount;

    LinkedHashIterator() {
        next = head; // 从head 开始遍历
        expectedModCount = modCount;
        current = null;
    }

    public final boolean hasNext() {
        return next != null;
    }

    final LinkedHashMap.Entry<K,V> nextNode() {
        LinkedHashMap.Entry<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        current = e;
        next = e.after; //后继节点
        return e;
    }

    public final void remove() {
        Node<K,V> p = current;
        if (p == null)
            throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        current = null;
        K key = p.key;
        removeNode(hash(key), key, null, false, false); //从链表中移除节点
        expectedModCount = modCount;
    }
}
```
可以看到，在遍历所有节点时是通过节点的 after 引用进行的。这样，可以双链表的头部遍历到到双链表的尾部，相比HashMap要高效那么一点点。


###  总结
本文对LinkedHashMap 进行了简要的分析，LinkedHashMap  通过增加的属性来维护元素之间的顺序，在HashMap的基础上来分析LinkedHashMap  很简单，其整个过程就类似于分析链表一样，因此在看本文之前需要有HashMap的知识，这样才知道在讲什么。
    1. LinkedHashMap 继承自 HashMap，并在其基本结构上增加了双向链表的实现，允许存储null 值。
    2. LinkedHashMap 的一个重要的特点就是支持**按照插入顺序或访问顺序来遍历**所有的 Entry，这一点和 HashMap
    的乱序遍历很不相同。在一些对顺序有要求的场合，就需要使用 LinkedHashMap 来替代 HashMap。

[Java集合之HashMap源码分析](http://blog.ztgreat.cn/article/20)

[Java集合之LinkedList源码分析](http://blog.ztgreat.cn/article/15)
