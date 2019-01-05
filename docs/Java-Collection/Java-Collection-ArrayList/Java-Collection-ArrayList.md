集合是Java中非常重要而且基础的内容，平时我们使用得最多，其用法也很简单，会使用一个，基本其它就很easy了，得益于集合框架的设计，既然第一步使用已经会了，那么还是有必要深入了解一下，学习其设计技巧，理解其本质，这样不仅会用，还会用得更好，有了更深层次的理解，那么使用过程中都很明白，而不是乱用一通，如果出现问题，也容易排查，今天我们就开始Java 集合框架的探险之旅。

说道Java集合，估计大家最熟悉的下面的图了，这个图是我从网上找的。
![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171012202420596.gif)

这个图很形象，基本上看这个图就能大致理清楚Java集合中的关系，先从ArrayList入手。

### ArrayList（jdk 1.8）
ArrayList是最常见集合类了，动态数组，顾名思义，ArrayList就是一个以数组形式实现的集合.

### 继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20170928093732069.png)

ArrayList 继承 AbstractList，implements List,Serializable,RandomAccess,Cloneable 接口。会发现有趣的现象:AbstractList 是implements了List接口，而ArrayList 也再次implements了List接口。
Random Access List(随机访问列表)，ArrayList 是随机访问的，因此需要打上这个tag,对应的还有Sequence Access List(顺序访问列表)。
Serializable，Cloneable  表明 ArrayList 是可以序列化的，克隆的。

### 数据结构

**transient Object[] elementData  :**

ArrayList是基于数组的一个实现，elementData就是底层的数组

**private int size:**

ArrayList里面元素的个数

对ArrayList的操作实际就是对数组的操作，只是这个数组可以动态扩张。


### 初始构造

```
/**
 * Constructs an empty list with an initial capacity of ten.
 */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

无参构造，底层数组指向一个空的数组。

```
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                initialCapacity);
    }
}
```

也可以指定一个初始大小，那么ArrayList将生成一个initialCapacity大小的数组。
### 动态扩张
我们来看看ArrayList是如何扩容的。

```
/**
 * Increases the capacity to ensure that it can hold at least the
 * number of elements specified by the minimum capacity argument.
 *
 * @param minCapacity the desired minimum capacity
 */
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

扩容的时候把元素组大小变为原来的1.5倍。

```
public static <T> T[] copyOf(T[] original, int newLength) {
    return (T[]) copyOf(original, newLength, original.getClass());
}
```

```
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0,
            Math.min(original.length, newLength));
    return copy;
}
```

可以看到，数组扩容很简单，产生一个和原来类型一致的，长度为newLength的大小的数组，然后将原来数组中的数据拷贝到新的数组中，完成扩容功能。这种我个人觉得我们应该都类似封装过这种类似的具有动态扩容功能的数组。

### 添加元素

```
/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return <tt>true</tt> (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

此add是将元素添加至数组末尾，添加前，先检查是否需要扩容，然后放到数组末尾，O(1)的时间复杂度

```
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
            size - index);
    elementData[index] = element;
    size++;
}
```

将元素添加到指定的位置上，首先会检查索引位置是否有效，这种方式意味着将会移动数组中的数据，将index后的数据依次往后挪一个，然后将数据插入到index位置上，这种添加方式的时间复杂度为O(n)，如果index越靠后，那么性能越好，反之越差。

### 删除元素

```
/**
 * Removes the element at the specified position in this list.
 * Shifts any subsequent elements to the left (subtracts one from their
 * indices).
 *
 * @param index the index of the element to be removed
 * @return the element that was removed from the list
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

删除指定位置上的元素，先把指定元素后面位置的所有元素，利用System.arraycopy方法整体向前移动一个位置，再把最后一个位置的元素指定为null，这样让gc可以去回收它。

```
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```

按元素删除，通过remove 方法，我们知道ArrayList里面是可以存null值的，如果删除的是null，那么就会找里面是null的元素进行删除，否则将调用对象的equals方法了判断两个对象是否相等（根据情况决定是否重写equals）,fastRemove 取名快速删除，但是里面还是利用的移动元素方式，其实一点也不快速呀。

其它类似的方法，和添加，删除差不多，本质上都是对数组的操作，因此这里就不一一展开了。


### 克隆
ArrayList 实现了 Cloneable 表明这个类是可以克隆的，查看Cloneable  发现什么都没有，其实Cloneable 我理解的只是一个标签，表面该类可以克隆，至于怎么克隆它不管，此外，在Object中 是有clone方法的，因此Cloneable  没必要再有clone接口了，我们可以重写父类clone 方法，按照我们要求进行克隆，看看ArrayList 里面的克隆。

```
/**
 * Returns a shallow copy of this <tt>ArrayList</tt> instance.  (The
 * elements themselves are not copied.)
 *
 * @return a clone of this <tt>ArrayList</tt> instance
 */
public Object clone() {
    try {
        ArrayList<?> v = (ArrayList<?>) super.clone();
        v.elementData = Arrays.copyOf(elementData, size);
        v.modCount = 0;
        return v;
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError(e);
    }
}
```

本质上很简单，就是把底层的数组复制拷贝一遍，这样就产生了一个新的含有相同数据的ArrayList ，可以看出这种克隆方式是浅拷贝，至于深浅拷贝 这里不会谈，原来我在C++中有研究，博客里面也有，不过可能写的质量一般，对于深浅拷贝很简单，只要理解了基本数据类型和引用性数据类型的区别就明白了。
#### 序列化
ArrayList 实现了 Serializable，说明ArrayList 可以序列化，可以通过网络传播，但是我们看一下ArrayList中的数组，是这么定义的：

```
transient Object[] elementData; // non-private to simplify nested class access
```

为什么elementData是使用transient修饰的呢？（被声明为transient的属性不会被序列化，这就是transient关键字的作用）

Java并不强求用户非要使用默认的序列化方式，用户也可以按照自己的喜好自己指定自己想要的序列化方式。
进行序列化、反序列化时，虚拟机会首先试图调用对象里的writeObject和readObject方法，进行用户自定义的序列化和反序列化。如果没有这样的方法，那么默认调用的是ObjectOutputStream的defaultWriteObject以及ObjectInputStream的defaultReadObject方法。换言之，利用自定义的writeObject方法和readObject方法，用户可以自己控制序列化和反序列化的过程。

既然elementData 不能被自动序列化，那么肯定会被手动序列化，那么我们看看ArrayList 里面的writeObject方法。

```
/**
 * Save the state of the <tt>ArrayList</tt> instance to a stream (that
 * is, serialize it).
 *
 * @serialData The length of the array backing the <tt>ArrayList</tt>
 *             instance is emitted (int), followed by all of its elements
 *             (each an <tt>Object</tt>) in the proper order.
 */
private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
    // Write out element count, and any hidden stuff
    int expectedModCount = modCount;
    s.defaultWriteObject();

    // Write out size as capacity for behavioural compatibility with clone()
    s.writeInt(size);

    // Write out all elements in the proper order.
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }

    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}
```

其实发现简单，writeObject中只序列化了elementData中有数据的部分，其实这个还是很容易理解的，ArrayList里面的elementData未必是满的，那么是否有必要序列化整个elementData呢？显然没有这个必要，因此ArrayList中重写了writeObject方法。
当然还有对应的反序列化 readObject方法，这个就是writeObject的逆过程，这里就不再展开了，可以自己去看一下。

### ArrayList与AbstractList

ArrayList重写了父类的indexOf等方法，在AbstractList中，indexOf方法是用迭代器实现的，在ArrayList中对数组进行顺序遍历实现，重写的原因就是为了速度。因为ArrayList底层是数组实现的可以随机访问，随机访问性能比迭代器高。

AbstractList中的indexOf方法：
```
public int indexOf(Object o) {
    ListIterator<E> it = listIterator();
    if (o==null) {
        while (it.hasNext())
            if (it.next()==null)
                return it.previousIndex();
    } else {
        while (it.hasNext())
            if (o.equals(it.next()))
                return it.previousIndex();
    }
    return -1;
}
```

ArrayList 中的indexOf方法：
```
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

### 总结

从上面的几个过程总结一下ArrayList的特点：

>**ArrayList 存储的数据允许为空**
>
>**ArrayList 允许存储重复数据**
>
>**ArrayList 存储数据是有序**（有序的意思是读取数据的顺序和存放数据的顺序是否一致）
>
>**ArrayList非线程安全**（对底层数组的操作未同步）


总结一下ArrayList优缺点。ArrayList的优点如下：

1、ArrayList底层以数组实现，是一种**随机访问模式**，查找也就是get的时候非常快

2、ArrayList在**顺序添加**一个元素的时候非常方便，只是往数组里面添加了一个元素而已

ArrayList的缺点：

1、删除元素的时候，涉及到一次元素复制，如果要复制的元素很多，那么就会比较耗费性能

2、插入元素的时候，涉及到一次元素复制，如果要复制的元素很多，那么就会比较耗费性能

因此，ArrayList比较适合**顺序添加、随机访问**的场景。
