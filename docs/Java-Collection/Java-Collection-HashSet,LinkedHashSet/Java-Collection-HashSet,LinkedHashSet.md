HashSet，它是基于HashMap实现的，HashSet底层使用HashMap来保存所有元素，因此HashSet 的实现比较简单，相关HashSet的操作，基本上都是直接调用底层HashMap的相关方法来完成，因此如果明白了HashMap那么HashSet也就自然明白，如果还没有看HashMap，那么不建议直接来看HashSet，可以参考我前面写的关于HashMap的博客。

 [Java集合之HashMap源码分析](http://blog.ztgreat.cn/article/20)

 [Java集合之LinkedHashMap源码分析](http://blog.ztgreat.cn/article/24)

 [Java集合之TreeMap源码分析](http://blog.ztgreat.cn/article/26)

### HashSet 介绍（jdk 1.8）

HashSet类，是存在于java.util包中的类。同时也被称为集合，该容器中只能存储不重复的对象，当我们提到HashSet时，第一件事情就是在将对象存储在HashSet之前，要先确保对象重写equals()和hashCode()方法，这样才能比较对象的值是否相等，以确保set中没有储存相等的对象。如果我们没有重写这两个方法，将会使用这个方法的默认实现。
HashSet它是基于HashMap实现的，HashSet底层使用HashMap来保存所有元素。

### 继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171121200441213.png)

HashSet继承自AbstractSet，实现了Set接口。
Set接口则是定义的集合操作。
AbstractSet实现了用于set集合判断的 equals 和hashCode 实现。
### 数据结构
HashSet 的数据结构很简单，原因就在于其内部是组合HashMap 使用的
```
private transient HashMap<E,Object> map;
// Dummy value to associate with an Object in the backing Map
// HashMap key对应的value
private static final Object PRESENT = new Object();
```
HashMap是key-value结构，但是HashSet存储的相当于是key，因此HashSet内部用了一个固定值PRESENT 来充当每个key的value,没有实际意义的。

### 构造方法
1、默认构造
```
/**
 * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
 * default initial capacity (16) and load factor (0.75).
 */
public HashSet() {
    map = new HashMap<>();
}
```
内部就是初始化默认的HashMap.

2、指定容量和负载因子
```
public HashSet(int initialCapacity, float loadFactor) {
     map = new HashMap<>(initialCapacity, loadFactor);
 }
```
3、通过集合初始化HashSet
```
public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}
```
4、指定dummy参数
```
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```
内部使用的是LinkedHashMap，dummy也没有用。
### 添加元素
1、add(E e)
```
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```
看吧，直接调用的HashMap的put 方法，HashMap会计算key的hash值，同时会比较key 是否已经存在，如果存在默认会覆盖其value.

```
// HashMap 中put 代码片段
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
```
// HashMap 中put 代码片段
Node<K,V> e; K k;
if (p.hash == hash &&
    ((k = p.key) == key || (key != null && key.equals(k))))
    e = p;
```
因此如果HashSet存放的是复杂的数据类型，有必要重写hashCode和equals 方法。


### 删除操作
1、remove(Object o)
```
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```
内部调用HashMap的remove 方法。


### 迭代器
```
public Iterator<E> iterator() {
    return map.keySet().iterator();
}
```
使用map的迭代器。

### 克隆

```
public Object clone() {
    try {
        HashSet<E> newSet = (HashSet<E>) super.clone();
        newSet.map = (HashMap<E, Object>) map.clone();
        return newSet;
    } catch (CloneNotSupportedException e) {
        throw new InternalError(e);
    }
}
```
同样使用的HashMap中的克隆方法。HashMap中的clone方法会生成一个新的hash表，然后把原map 中的数据重新put到新map中。

### 序列化

```
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException {
    // Write out any hidden serialization magic
    s.defaultWriteObject();

    // Write out HashMap capacity and load factor
    s.writeInt(map.capacity());
    s.writeFloat(map.loadFactor());

    // Write out size
    s.writeInt(map.size());

    // Write out all elements in the proper order.
    for (E e : map.keySet())
        s.writeObject(e); //只需要序列化key
}
```
HashSet序列化只需要序列化key就可以了，其内部的value没有用，因此不需要使用HashMap的序列化工作。
### LinkedHashSet
看完HashSet,那么LinkedHashSet 就更简单了
#### 继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171128192226777.png)

LinkedHashSet  继承HashSet，你说是不是就很简单了。
#### 构造
```
public LinkedHashSet(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor, true);
}
public LinkedHashSet(int initialCapacity) {
    super(initialCapacity, .75f, true);
}
public LinkedHashSet() {
    super(16, .75f, true);
}
public LinkedHashSet(Collection<? extends E> c) {
    super(Math.max(2*c.size(), 11), .75f, true);
    addAll(c);
}
```
LinkedHashSet  都是调用的HashSet中特定的一个构造方法。
```
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```
在父类的这个构造方法中，使用的是LinkedHashMap 来实现HashSet，因此LinkedHashSet  本质上也是通过LinkedHashMap 来实现的，属于LinkedHashSet  特有的方法基本没有，因此只要明白了HashMap的一套，那么HashSet也就自然明白了，这里也就不多讲LinkedHashSet 。

### 总结
HashSet就这样分析了，与其说分析，还不如说是浏览了一遍，HashSet简单不是指其本身简单，得益于其框架设计的巧妙。HashSet内部是组合HashMap来使用的,因此非常简单，因为HashMap是非线程安全的，HashSet也是非线程安全的，允许插入空元素，HashMap的key不会重复，因此HashSet中自然也就不会出现重复的数据了。
LinkedHashSet  是通过HashSet来实现的，本质上是通过LinkedHashMap 实现的。LinkedHashMap里面元素是有序的（插入顺序或者访问顺序），因此LinkedHashSet 也是有序的（插入顺序或者访问顺序）。
在实际的使用中 HashSet的Key可能需要重写hashcode跟equals方法。

 #### [Java集合之HashMap源码分析](http://blog.ztgreat.cn/article/20)
 #### [Java集合之LinkedHashMap源码分析](http://blog.ztgreat.cn/article/24)
 #### [Java集合之TreeMap源码分析](http://blog.ztgreat.cn/article/26)