前面，我们已经学习了ArrayList，我们接着学习集合框架——LinkedList。

### LinkedList(jdk 1.8)

### 介绍
LinkedList是基于链表实现的，是一种线性的存储结构。
LinkedList是一种**双向非循环**链表：链表中任意一个存储单元（除头尾结点）都可以通过向前或者向后寻址的方式获取到其前一个存储单元和其后一个存储单元

### 继承体系

![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171006143912283.png)



LinkedList 是一个继承于AbstractSequentialList的双向链表。它也可以被当作堆栈、队列或双端队列进行操作。
LinkedList 实现 List 接口，能对它进行队列操作。
LinkedList 实现 Deque 接口，即能将LinkedList当作双端队列使用。
LinkedList 实现 Cloneable接口，能克隆。
LinkedList 实现 Serializable接口，LinkedList支持序列化，能通过序列化去传输。


### 数据结构

```
public class LinkedList<E>  
    extends AbstractSequentialList<E>  
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable{     
    //LinkedList大小  
    transient int size = 0;  
    //第一个节点  
    transient Node<E> first;  
    //最后一个节点  
    transient Node<E> last;  
    //省略内部类和方法。。  
} 
```
Node 代表的是LinkedList 中的一个存储单元。它是LinkedList中的静态内部类。
```
private static class Node<E> {
    //存储的数据
    E item;
    Node<E> next;   //指向后一个Node
    Node<E> prev;   //指向前一个Node

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

LinkedList存储逻辑结构图：



![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171006231909099.png)

### 初始构造

1、 默认构造方法
```
/**
 * Constructs an empty list.
 */
public LinkedList() {
}
```

初始化的时候size为0，first和last的节点都为空。

2、有参构造

```
/**
 * Constructs a list containing the elements of the specified
 * collection, in the order they are returned by the collection's
 * iterator.
 *
 * @param  c the collection whose elements are to be placed into this list
 * @throws NullPointerException if the specified collection is null
 */
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

将集合c中的元素一个一个有序的添加到LinkedList中。

### 添加元素
1、add(E e)

```
/**
 * Appends the specified element to the end of this list.
 *
 * <p>This method is equivalent to {@link #addLast}.
 *
 * @param e element to be appended to this list
 * @return {@code true} (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    linkLast(e);
    return true;
}
```

通过LinkedList这个add 方法将元素添加到链表末尾中，内部调用的是linkLast 方法

```
/**
 * Links e as last element.
 */
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

linkLast 方法就是把新结点加入到末尾。

有了linkLast，自然就有linkFirst，linkFirst是把元素添加到最开始。
```
/**
 * Links e as first element.
 */
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

对于添加到头或者尾这种操作都非常快，时间复制度都是O(1)。

2、add(int index, E element)

```
/**
 * Inserts the specified element at the specified position in this list.
 * Shifts the element currently at that position (if any) and any
 * subsequent elements to the right (adds one to their indices).
 *
 * @param index index at which the specified element is to be inserted
 * @param element element to be inserted
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
```

将元素添加到指定的位置上，首先会检查index是否有效，然后将元素添加的导致的index位置上。

```
/**
 * Returns the (non-null) Node at the specified element index.
 */
Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

node(int index)方法查找并返回index位置上的元素， 若index < 双向链表长度的1/2,则从前先后查找，否则，从后向前查找。
因为node 方法涉及到查找元素，因此非特殊情况下该add 方法时间复杂度为O(n)。

### 删除元素
1、remove(Object o)

```
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```

遍历链表，查找 “相等”的两个元素，然后从链表中删除，从这个方法中我们知道，如果参数为空，那么将删除链表中第一个遇到元素为空的结点，这也意味着LinkedList里面运行元素为空。

```
/**
 * Unlinks non-null node x.
 */
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```

unlink(Node<E> x) 方法就是将结点x 从链表中删除，并连接该结点的前后继结点。

2、remove(int index)
```
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}
```

删除指定位置上的元素，先查找index中的元素，再将该结点从链表中去掉。

3、remove()
```
public E remove() {
    return removeFirst();
}
```

这个remove 删除链表中的第一个元素，这个方法重写的 Deque中的remove,用于LinkedList模拟双端队列操作（操作链表（队列）头或尾）。
### 查找元素
1、 get(int index)

```
/**
 * Returns the element at the specified position in this list.
 *
 * @param index index of the element to return
 * @return the element at the specified position in this list
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

查找某个位置的元素，通过遍历链表查找元素，时间复杂度O(n)，因此如果遍历LinkedList 调用get获取元素，那么时间复杂度为O(n平方)，可见性能只差，而对于ArrayList 因为底层是数组，可以随机方法，因此遍历ArrayList 时间复杂度为O(n)。

2、indexOf(Object o)

```
public int indexOf(Object o) {
    int index = 0;
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index;
            index++;
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index;
            index++;
        }
    }
    return -1;
}
```

按顺序遍历链表，返回第一次出现该元素的位置索引，如果没找到，则返回-1.

2、lastIndexOf(Object o)

```
public int lastIndexOf(Object o) {
    int index = size;
    if (o == null) {
        for (Node<E> x = last; x != null; x = x.prev) {
            index--;
            if (x.item == null)
                return index;
        }
    } else {
        for (Node<E> x = last; x != null; x = x.prev) {
            index--;
            if (o.equals(x.item))
                return index;
        }
    }
    return -1;
}
```

从后遍历链表，返回第一出现该元素的位置索引，如果没有找到则返回-1.
### 克隆
LinkedList 实现了 Cloneable 表明这个类是可以克隆的，查看Cloneable 发现什么都没有，其实Cloneable 我理解的只是一个标签，表面该类可以克隆，至于怎么克隆它不管，此外，在Object中 是有clone方法的，因此Cloneable 没必要再有clone接口了，我们可以重写父类clone 方法，按照我们要求进行克隆，看看LinkedList 里面的克隆

```
public Object clone() {
    LinkedList<E> clone = superClone();

    // Put clone into "virgin" state
    clone.first = clone.last = null;
    clone.size = 0;
    clone.modCount = 0;

    // Initialize clone with our elements
    for (Node<E> x = first; x != null; x = x.next)
        clone.add(x.item);

    return clone;
}
```

```
private LinkedList<E> superClone() {
    try {
        return (LinkedList<E>) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new InternalError(e);
    }
}
```

产生一个新的链表，将原链表中的数据全部重新添加到新链表中，这样就完成克隆了，可以看出这种克隆方式也是浅拷贝。


### 序列化
LinkedList 实现了 Serializable，说明LinkedList 可以序列化，可以通过网络传播。

```
transient int size = 0;

/**
 * Pointer to first node.
 * Invariant: (first == null && last == null) ||
 *            (first.prev == null && first.item != null)
 */
transient Node<E> first;

/**
 * Pointer to last node.
 * Invariant: (first == null && last == null) ||
 *            (last.next == null && last.item != null)
 */
transient Node<E> last;
```

在LinkedList中，类属性都是transient关键字修饰（被声明为transient的属性不会自动被序列化），这些属性的序列化是通过手动实现。
Java并不强求用户非要使用默认的序列化方式，用户也可以按照自己的喜好自己指定自己想要的序列化方式。 
进行序列化、反序列化时，虚拟机会首先试图调用对象里的writeObject和readObject方法，进行用户自定义的序列化和反序列化。如果没有这样的方法，那么默认调用的是ObjectOutputStream的defaultWriteObject以及ObjectInputStream的defaultReadObject方法。换言之，利用自定义的writeObject方法和readObject方法，用户可以自己控制序列化和反序列化的过程。

看看LinkedList序列化过程：
```
private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
    // Write out any hidden serialization magic
    s.defaultWriteObject();

    // Write out size
    s.writeInt(size);

    // Write out all elements in the proper order.
    for (Node<E> x = first; x != null; x = x.next)
        s.writeObject(x.item);
}
```

```
private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
    // Read in any hidden serialization magic
    s.defaultReadObject();

    // Read in size
    int size = s.readInt();

    // Read in all elements in the proper order.
    for (int i = 0; i < size; i++)
        linkLast((E)s.readObject());
}
```

首先调用默认的序列化方法，把未被transient修饰的属性进行序列化，然后再进行我们自定义的序列化，这里先记录链表大小，然后再把链表中的每个元素序列化。
反序列化，则是序列化的逆过程，读出数据添加到链表中。

### 总结

LinkedList允许存储的数据为空

LinkedList允许存储重复的数据

LinkedList存储的数据有序（有序的意思是读取数据的顺序和存放数据的顺序是否一致）

LinkedList非线程安全（对链表的的操作未同步）

LinkedList在头尾或者指定结点插入结点时非常快速，时间复杂度为O(1)，对于查询某个元素需要遍历链表，时间复杂度为O(n)。

LinkedList 理论上不会有容量限制，ArrayList最大存储容量为int的最大值，LinkedList每个元素结点之间逻辑相邻，物理不一定相邻，ArrayList 每个元素之间物理和逻辑都相邻，因此LinkedList能利用碎片化的内存。