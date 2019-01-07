### CopyOnWriteArrayList 介绍（jdk 1.8）
CopyOnWriteArrayList，顾名思义，Write的时候总是要Copy，读写分离的思想，通俗地讲，当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器复制出一个新的容器，然后在新的容器里添加元素，添加玩元素之后再讲原来容器的引用指向新的容器。

CopyOnWriteArrayList的读是并发的，不需要加锁，juc包提供了两个使用CopyOnWrite思想实现的并发容器，分别是CopyOnWriteArrayList和CopyOnWriteArraySet(CopyOnWriteArraySet内部持有一个CopyOnWriteArrayList，很多方法都是调用CopyOnWriteArrayList来实现的)。

### 继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20180104094336482.png)

### 数据结构
```
/** The lock protecting all mutators */
//可重入锁，用于数据的调整
final transient ReentrantLock lock = new ReentrantLock();

/** The array, accessed only via getArray/setArray. */
// 存储数据的数组
private transient volatile Object[] array;
```
### 构造方法
1、默认构造
```
/**
 * Sets the array.
 */
final void setArray(Object[] a) {
    array = a;
}

/**
 * Creates an empty list.
 */
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}
```
2、通过集合初始化

```
public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    //如果c 是CopyOnWriteArrayList 类型，直接调用其getArray 方法
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        //调用c的toArray 方法
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] 
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}
```
### get 操作
因为是对数组的访问，因此可以做到随机访问，速度很快，同时读取操作，不需要进行同步操作。
```
public E get(int index) {
    return get(getArray(), index);
}
private E get(Object[] a, int index) {
    return (E) a[index];
}
```

### 添加操作
1、add(E e) 添加元素到数组末尾，允许元素为null
```
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        //copy到新的数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        //添加元素
        newElements[len] = e;
        //重新设置引用关系
        setArray(newElements);
        return true;
    } finally {
        //释放锁
        lock.unlock();
    }
}
```
2、add(int index, E element) 添加元素到指定的位置
```
public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+
                                                ", Size: "+len);
        Object[] newElements;
        int numMoved = len - index;
        if (numMoved == 0) //添加元素到末尾
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            newElements = new Object[len + 1];
            // 空出index 位置
            // 先把原数组index前面的元素复制到新的数组中
            System.arraycopy(elements, 0, newElements, 0, index);
            // 再把原数组index后面的元素复制到新的数组中，这样index的位置就空缺出来了
            System.arraycopy(elements, index, newElements, index + 1,
                             numMoved);
        }
        // 添加元素
        newElements[index] = element;
        // 重新设置引用关系
        setArray(newElements);
    } finally {
        //释放锁
        lock.unlock();
    }
}
```
添加操作其实很简单：

1、加锁

2、拷贝原数组数据到新数组，空出需要添加元素的位置（末尾或者其它位置）

3、设置原数组引用指向新数组

### 删除操作
1、remove(int index) 移除某个位置上的数据

```
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        //获取需要删除位置上的元素
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0) // 删除的是最后一个元素
            //copy 生成新数组，并重新设置引用关系
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            //copy 元素到新的数组
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            setArray(newElements);
        }
        //返回需要删除的数据
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```
2、remove(Object o) 移除某个元素

```
public boolean remove(Object o) {
    //存储数据的数组
    Object[] snapshot = getArray();
    //寻找元素o的位置
    int index = indexOf(o, snapshot, 0, snapshot.length);
    //找到了就进行remove 操作CopyOnWriteArrayList 
    return (index < 0) ? false : remove(o, snapshot, index);
}
```

```
private boolean remove(Object o, Object[] snapshot, int index) {
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        Object[] current = getArray();
        int len = current.length;
        // 如果不相等，说明被其它线程修改了数据，重新产生了新的数组
        // 调用remove 之前没有加锁控制，所有其他线程可能修改了数据
        if (snapshot != current) findIndex: {
            int prefix = Math.min(index, len);
            //遍历新的数组，看是否存在元素o
            for (int i = 0; i < prefix; i++) {
                //current[i] != snapshot[i] 数组变了，其元素内存地址也应不一样
                if (current[i] != snapshot[i] && eq(o, current[i])) {
                    index = i; 
                    break findIndex; // 找到了退出
                }
            }
            //如果index >=len 则说明现在的数组缩减了，同时所有数据中，不存在o元素，返回false
            if (index >= len)
                return false;
            //删除的元素就在数组中，引用一样    
            if (current[index] == o)
                break findIndex;
            //还没有找到，从index-len之间再次寻找需要删除的元素o    
            index = indexOf(o, current, index, len);
            //没找到，返回false
            if (index < 0)
                return false;
        }
        //找到了，拷贝数据到新数组，删除index上的数据
        Object[] newElements = new Object[len - 1];
        System.arraycopy(current, 0, newElements, 0, index);
        System.arraycopy(current, index + 1,
                         newElements, index,
                         len - index - 1);
        // 设置新数组引用                  
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
### 总结
CopyOnWriteArrayList 线程安全，读写分离，读取数据时无需同步控制，修改数据时，会进行单独拷贝数组元素进行操作，当操作完成后，在设置新的数组引用。

CopyOnWriteArrayList允许存储的数据为空，允许重复数据，并且元素有序（存储的顺序和添加的顺序一致）

CopyOnWriteArrayList 底层数据结构是一个Object[]，初始容量为0，之后每增加一个元素，容量+1，数组复制一遍。

因为每次调整CopyOnWriteArrayList 的结构，都会涉及到数组的拷贝复制，因此比较适合数据量不是很大，读多写少的并发环境。

CopyOnWriteArrayList 读写分离，CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性,当其他线程正在修改数据时，因为是另一份数据的拷贝，因此此时读到的数据可能是脏数据。