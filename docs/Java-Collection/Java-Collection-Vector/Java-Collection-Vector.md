前面，我们已经学习了ArrayList，LinkedList，我们接着按着集合框架图学习--Vector。

### Vector（jdk 1.8）--线程安全
Vector和ArrayList大同小异，都是动态数组，Vector和ArrayList差别很小，因此本文不会按照ArrayList那样分析，只是简单揭开Vector的面纱，对比一下和ArrayList的区别。

#### 继承体系

![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171013205556840.png)

Vector和ArrayList的继承体系是一样的。
Vector实现RandmoAccess接口，即提供了随机访问功能，提供提供快速访问功能。在Vector我们可以直接访问元素。
Vector 实现了Cloneable接口，支持clone()方法，可以被克隆。
Vector 实现了Serializable接口，可以被序列化。
#### 数据结构

**(1)protected Object[] elementData:** 
Vector是基于数组的一个实现，elementData就是底层的数组

**(2)protected int elementCount:**
Vector里面元素的个数  

**(3)protected int capacityIncrement:**
Vector  扩容每次的增量

在ArrayList 中 底层数组是的声明有transient 关键字，而在Vector中没有。

Vector 中有一个capacityIncrement元素，这个是Vector 每次扩容时候的增量，这个可以在通过构造函数设置，在ArrayList，正常情况下每次扩容后是原来的1.5倍。

#### 构造方法
1、执行初始容量大小，和每次扩容的增量
```
    public Vector(int initialCapacity, int capacityIncrement) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity];
        this.capacityIncrement = capacityIncrement;
    }
```
2、默认增量值为0

```
    public Vector(int initialCapacity) {
        this(initialCapacity, 0);
    }
```
3、默认数组大小为10

```
    public Vector() {
        this(10);
    }
```

#### 扩容

```
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                         capacityIncrement : oldCapacity);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
如果增量为0，那么就扩大一倍，扩容后的大小是原来的两倍。

### 添加元素

```
    public synchronized boolean add(E e) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);
        elementData[elementCount++] = e;
        return true;
    }
```
这里只看Vector的一个添加方法，其余方法和ArrayList 方法类似，就不一样列出来了。
在Vector 方法中 多了一个synchronized 关键，在前面我们就学习过synchronized 关键字，它是用来进行线程同步的，被synchronized 修饰的方法将会被串行执行，也就是说是线程安全的，因此可以看出**Vector是线程安全的**，而**ArrayList是非线程的**。

#### 序列化
Vector的熟悉中没有被transient 所修饰，因此都可以进行默认的序列化，在Vector中我们还是发现了writeObject 方法

```
    private void writeObject(java.io.ObjectOutputStream s)
            throws java.io.IOException {
        final java.io.ObjectOutputStream.PutField fields = s.putFields();
        final Object[] data;
        synchronized (this) {
            fields.put("capacityIncrement", capacityIncrement);
            fields.put("elementCount", elementCount);
            data = elementData.clone();
        }
        fields.put("elementData", data);
        s.writeFields();
    }
```
Vector中的writeObject 和ArrayList中的writeObject 目的是一样的，减小不必要的数据序列化。因为elementData 也许并没有装满，完全序列化elementData 在网络传输中是浪费流量的。
ArrayList 中是手动的将elementData 进行序列化，Vector 严格来说不算手动，只是在序列化是动了点手脚。
Vector 中 将elementData添加到需要序列化的域，添加的elementData 是原elementData克隆的结果，克隆后的elementData 只有存在数据的地方，但是可能会包含空值（Vector 和ArrayList一样可以存放空置）,ArrayList中也是如此，只是实现方式不一样而已。

打个比方：把数据放入文件，有两种方式，一种写个程序把数据写到文件，还有一种就是可视化打开文件，然后把数据粘贴进行，完成。最后效果都是一样的，只是过程不一样而已。

Vector中没有readObject ，因为Vector的属性中没有被transient 所修饰，因此反序列化时就不要干预，自动就可以完成，而在ArrayList中因为elementData 是被transient 修饰的，不能自动序列化，需要我们手动设置数据。


#### 总结

Vector的 的分析很简单，因为和ArrayList基本上是差不多的。

    1. Vector 存储的数据允许为空
    2. Vector 允许存储重复数据
    3. Vector 存储数据是有序 （有序的意思是读取数据的顺序和存放数据的顺序是否一致）
    4. Vector **线程安全**（对底层数组的操作进行了同步）
    5. Vector 在扩容时会更加增量来进行扩容，没有设置增量，则扩大一倍。
