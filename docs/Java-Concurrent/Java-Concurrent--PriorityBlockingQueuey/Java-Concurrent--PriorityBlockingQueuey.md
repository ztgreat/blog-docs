在学习完ArrayBlockingQueue，LinkedBlockingQueue 之后，应该对阻塞队列有了一个比较清晰的认识，今天我们来学习另一个阻塞队列PriorityBlockingQueue。

### PriorityBlockingQueue 介绍(jdk 1.8)
PriorityBlockingQueue 是一个支持优先级的无界阻塞队列，默认情况下元素采取自然顺序升序排列，也可以自定义类实现compareTo()方法来指定元素排序规则，需要注意的是不能保证同优先级元素的顺序。
所谓无无界队列就是没有容量限制，可以"无限"的扩张，当然也不是无休止的扩张，这个还是有最大的限制的，这个我们在后面分析可以知道。

### 继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171027222814677.png)

PriorityBlockingQueue 实现了BlockingQueue接口，该接口中定义了阻塞的方法接口， 

PriorityBlockingQueue 继承了AbstractQueue，具有了队列的行为。 

PriorityBlockingQueue 实现了Serializable接口,可以序列化。

### 数据结构

```
public class PriorityBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {
    private static final long serialVersionUID = 5595510919245408276L;

    /**
     * Default array capacity.
     * 队列默认大小
     */
    private static final int DEFAULT_INITIAL_CAPACITY = 11;

    /**
     * The maximum size of array to allocate.
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     *队列最大容量，超过该容量抛oom异常
	 */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    //队列，数组实现 不序列化
    private transient Object[] queue;

    /**
     * The number of elements in the priority queue.
     * 队列元素个数 不序列化
     */
    private transient int size;

  
    private transient Comparator<? super E> comparator;

    /**
     * Lock used for all public operations
     * 用于队列操作的锁
     */
    private final ReentrantLock lock;

    /**
     * Condition for blocking when empty
     * 队列非空条件
     */
    private final Condition notEmpty;

    //队列扩容的 "锁" 不序列化
    private transient volatile int allocationSpinLock;

	//用于序列化
    private PriorityQueue<E> q;
    ...
 }
```
ArrayBlockingQueue，LinkedBlockingQueue 中通过指定大小来确定队列的大小，队列大小一旦确定后就不会改变，同时队列是否入队或者出队由两个条件来控制（notEmpty 和notFull ）,因此它们都是有界的阻塞队列
在PriorityBlockingQueue 我们看到只有notEmpty 条件，没有notFull 条件，同时也有默认的队列大小，也就是说PriorityBlockingQueue 没有队满的概念，当队列满了以后，那么就进行扩容，当达到最大的容量后就不能继续入队了，否则就会抛异常，而不是前面两个阻塞队列那样，等到notFull 条件满足，具体我们可以在PriorityBlockingQueue 后面的源码中看到。

PriorityBlockingQueue 中通过一个可重入锁来控制入队和出队行为，这个和ArrayBlockingQueue 中是一致的，LinkedBlockingQueue 中对入队和出队用了两个不同的可重入锁来控制。

PriorityBlockingQueue 底层还是通过数组来实现的。

### 二叉堆
PriorityBlockingQueue 是一种优先级的队列，其入队和出队会按照优先级进行排序，是基于优先级的FIFO.
PriorityBlockingQueue 是基于数据来存储的，但是其数据结构是二叉堆，这个我们在数据结构课本中有学习到，而且当时的二叉堆的应用也就是用来做优先级队列，二叉堆在很久学习数据结构的时候写过一篇博客：[数据结构之堆排序](http://blog.csdn.net/u014634338/article/details/39101817)
不是为了写博客而写博客，因此还是有必要再次来复习一下二叉堆，当然这里不会涉及到很多细节，只是来理解二叉堆及其操作方法。

#### 认识
**二叉堆本质是一颗二叉树**，首先我们来看一颗二叉树
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028114358760.png)
这是一颗普通的二叉树，也许你发现了父节点的值总是大于或等于任何一个子节点的值，但是在这里并不重要。
现在我们对这颗二叉树进行编号，编号的规则呢就是**从上到下，从左到右**，从0开始编号。

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028114622782.png)

现在我们需要把这颗逻辑上的二叉树存储到具体的物理介质上，那么就需要把逻辑结构转换为物理结构，一般我们都会用链式进行存储，这里我们用数组来进行存储，因为数组是顺序的，不像链式那样容易遍历有关系的节点，那么就需要想个办法，把父节点和孩子节点通过某种关系联系起来，这里编号的重要性的来了，现在我们先按照编号顺序把树放到数组中。

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028120247643.png)

观察一下父子之间的编号，会发现如果节点在数组中的位置是i(i是节点在数组中的下标), 则i节点对应的子节点在数组中的位置分别是 2i + 1 和 2i + 2，同时i的父节点的位置为 (i-1)/2。
因此现在我们就把树存储到了数组中，同时通过这种规律可以很快找到每个节点的父亲和孩子节点。

但是我们的二叉树会不会是下面这种结构呢：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028122301813.png)

如果是这种结构那么岂不是就不能满足父子间的位置关系了，答案是不会的。
前面说了**二叉堆本质是一颗二叉树**，但是这个还准确，**二叉堆是一种特殊的堆，二叉堆是完全二叉树或者是近似完全二叉树**（引用维基百科） ，近似完全二叉树 是什么？？？？，因此二叉堆不会是这种结构。

#### 上浮
上浮就是向上调整元素，把元素调整到它该放的位置，下面我们用图解来进行演示这个过程：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028124344584.png)

为了比较直观，我们还是采用树形结构的二叉堆来进行演示，实际上对于数组结构的本质上也是一样的。
对于编号为0的位置，我把称为堆顶，按照什么规则来“顺序”放置元素这个可以自行定义，例如：可以按照升序或降序的方式来排列元素，那么堆顶对于的就是最小或者最大值的元素。
这里我们定义：元素值小的元素，优秀级越高，优秀级最高的元素就会被放在堆顶，如果把优秀级最高的元素放在堆顶呢？---上浮

1、从非叶子节点并且编号最大的节点开始调整。

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028130341928.png)

编号为3的节点 是非叶子节点并且编号最大的节点，从这里开始依次进行向上调整。
把编号3的节点的元素值和其孩子进行比较，元素小（这里优先级高）的节点上位，因此元素值为20的节点替代元素值为60的节点。

2、依次向上进行调整
对编号3的节点调整结束后，接下来就是对编号2的节点进行的调整，后面结果类似，就直接以图展示了
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028130422186.png)

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028130545448.png)

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028130841034.png)

现在最小的元素就到了堆顶了，如果需要把整个堆都进行有序化，那么再次重复上面的过程即可，依次将第二小的元素，第三小元素... 放到该放的位置，这里我们就不在展示了。

#### 下沉
下沉就是将元素向下进行调整，我们接着上面的图进行讲解

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028141304713.png)

在上面我们已经把最小元素放到了堆顶，现在我们把该元素取走，该元素就是数组的第一个元素，因此很方便的就可以获取到，当把该元素取走后，该位置就空了，那么该如何处理呢？

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028141603550.png)

我们用最后一个元素代替了堆顶，此时堆顶虽然不空，但是堆顶并不是我们想要的元素（最小），那么这个时候就需要进行调整，如何调整呢 --- 下沉
下沉其实和上浮本质上是差不多的，就是依次和孩子节点进行比较，如果发现自己比孩子小，那么就和孩子节点进行交换，那么自己就下沉了，下面我们用图来演示这个过程。

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028142442140.png)

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028142653088.png)

这就是下沉操作，其实和上浮是不是差不多的，但是我们发现堆顶元素仍然不是我们想要的元素，其实这个问题不是下沉的原因，而是在于我们没有将整个堆进行有序化，也就是说，前面的上浮操作把整个堆都进行有序化，那么编号为1的地方就是堆中第二小的元素，那么再去除堆顶元素，进行下沉后，那么最小的元素就将出现在堆顶，因此如果堆不是有序化的，那么进行下沉操作是没有意义的，我们只有再次进行上浮操作，才能将符合条件的元素放到堆顶。

#### 二叉堆小结

1、二叉堆是一种特殊的堆，二叉堆是完全二叉树或者是近似完全二叉树

2、如果节点在数组中的位置是i(i是节点在数组中的下标), 则i节点对应的子节点在数组中的位置分别是 2i + 1 和 2i +2，同时i的父节点的位置为 (i-1)/2（i从0 开始）

3、通过一次上浮可以把符合条件的元素放到堆顶，反复进行上浮操作，可以将整个堆进行有序化

4、下沉操作可以把替代堆顶后的元素放到该放的位置，同时堆顶元素是符合条件的元素（前提是二叉堆已经是有序的）


###  构造方法
说完二叉堆，我们接下来继续看PriorityBlockingQueue。

1、默认构造

```
public PriorityBlockingQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}
```

默认构造方法，会调用其它构造方法，指定一个默认的容量，同时比较器Comparator 为null
2、指定容量构造方法

```
public PriorityBlockingQueue(int initialCapacity) {
    this(initialCapacity, null);
}
```

3、指定容量，指定比较器

```
public PriorityBlockingQueue(int initialCapacity,
                             Comparator<? super E> comparator) {
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    this.comparator = comparator;
    //生成队列
    this.queue = new Object[initialCapacity];
}
```

4、通过集合初始化

```
public PriorityBlockingQueue(Collection<? extends E> c)
```
这个我们暂就不列出来了，里面涉及了其它方法，这个我们在后面分析完其它方法后，可以再来回顾这个方法。

### 入队

#### 1、add(E e) 
将指定的元素插入到此队列中，在成功时返回 true

```
public boolean add(E e) {
    return offer(e);
}
```

#### 2、offer(E e) 
将指定的元素插入到此队列中，在成功时返回 true，在前面的add 中，内部调用了offer 方法，我们也可以直接调用offer 方法来完成入队操作。

```
public boolean offer(E e) {
    //队列中的元素不能为空
    if (e == null)
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    int n, cap;
    Object[] array;
    //如果队列满了，则进行扩容
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);
    try {
        //比较器
        Comparator<? super E> cmp = comparator;
        //如果没有设置比较器，则进行默认比较
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp);
        size = n + 1;
        //入队后  notEmpty 条件满足，唤醒阻塞在notEmpty 条件上的一个线程
        notEmpty.signal();
    } finally {
        //释放锁
        lock.unlock();
    }
    return true;
}
```

1、判断元素是否未空

2、判断容器是否需要扩容，如果是则扩容

3、入队操作

4、Condition 释放信号

#### 上浮
这里我们先来看看入队操作siftUpComparable

```
private static <T> void siftUpComparable(int k, T x, Object[] array) {
    //元素自身默认比较器
    Comparable<? super T> key = (Comparable<? super T>) x;
    while (k > 0) {
        //元素x的父节点的位置 (k-1)/2
        int parent = (k - 1) >>> 1;
        //父节点
        Object e = array[parent];
        //判断是否将x元素存在位置k,当前元素和父节点比较，如果当前节点大于父节点，退出循环
        if (key.compareTo((T) e) >= 0)
            break;
        //当前节点比父节点小，父节点元素下沉，    
        array[k] = e;
        //从父节点位置开始，判断是否将x元素存在位置k
        k = parent;
    }
    //找到位置k,将元素x 存放在该位置
    array[k] = key;
}
```

为了更好的理解，我下面用图展示这一过程：

入队元素90：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028153649767.png)
在这个已有的堆中入队元素90
现在堆中的元素的个数是7，父节点位置：（7-1）/2=3,位置3中的元素为50 比入队元素小，因此满足元素小的优先级高，父元素比孩子元素小，因此直接将元素90存储在位置7上

入队元素10：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028153727779.png)

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028153800807.png)

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171028153843019.png)

结合图和上面代码应该就很好理解了，对于siftUpUsingComparator其实是一样的，只是比较器变了而已，就不多讲了。

#### 扩容
接下来看看扩容tryGrow

```
private void tryGrow(Object[] array, int oldCap) {
    //释放锁
    lock.unlock(); // must release and then re-acquire main lock
    Object[] newArray = null;
    //用cas 将allocationSpinLock 设置为1
    if (allocationSpinLock == 0 &&
            UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                    0, 1)) {
        try {
            int newCap = oldCap + ((oldCap < 64) ?
                    (oldCap + 2) : // grow faster if small
                    (oldCap >> 1));
            if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                //超过最大容量，扩容增加1
                int minCap = oldCap + 1;
                //还是超过最大值 oom了
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;
            }
            //queue == array 这里保证 queue还未被修改
            if (newCap > oldCap && queue == array)
                newArray = new Object[newCap];
        } finally {
            //还原
            allocationSpinLock = 0;
        }
    }
    //其它线程对队列进行了改动，放弃扩容
    if (newArray == null) // back off if another thread is allocating
        Thread.yield();
    //重新加锁，准备回到offer 中    
    lock.lock();
    if (newArray != null && queue == array) {
        //扩容，复制内容到新数组
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```

这个扩容方法还是很有意思

1、释放了可重入锁，此时其它线程可以操控队列

2、如果allocationSpinLock=0 ，则cas 设置成为1 

3、如果超出最大容量，则抛oom

4、如果队列没有被修改，则扩容

5、准备回到offer 方法中，重新加锁，如果获取到锁，其它线程无法修改队列

6、如果期间队列没有被修改，那么扩容，复制队列元素到新队列

7、还原allocationSpinLock

这里的allocationSpinLock 其实相当于锁的功能，因为在该方法中，释放掉了锁，那么其它线程可能就会操作队列，那么也可能进行扩容操作，为了保证扩容的线程安全，那么就用allocationSpinLock 来进行记录，来保证只有一个线程能执行扩容代码。
通过判断 queue == array 是否相等（数组中的每个元素都是否相等），来判断是否其它线程对队列元素进行了修改，如果其它元素对队列进行了修改，那么就会放弃扩容，因此才会看到在 offer 中通过while 循环来判断是否真正需要扩容

```
while ((n = size) >= (cap = (array = queue).length))
     tryGrow(array, cap);
```
应该从offer 中进入到tryGrow 中释放了锁，因此最后需要重新获取锁，获取锁后，其它线程将不能操作队列，此时再次判断是否能扩容，如果是则进行扩容，复制队列元素到新队列中，完毕。

#### 3、offer(E e, long timeout, TimeUnit unit) 
```
public boolean offer(E e, long timeout, TimeUnit unit) {
    return offer(e); // never need to block
}
```

这个方法在前面的ArrayBlockingQueue，LinkedBlockingQueue 中都进行了超时等待，而这里实际上并没有进行超时等待。因为**PriorityBlockingQueue 是无界队列的概念，不会“队满”，实际当到达队列最大值后，就抛oom异常了，因此这点在使用优先队列的时候，需要注意**。

#### 4、put(E e) 
将指定的元素插入此队列中,队列达到最大值，则抛oom异常

```
public void put(E e) {
    offer(e); // never need to block
}
```

这个方法和上面的类似，就不多讲了。

### 出队
#### 1、poll() 
获取并移除此队列的头，如果此队列为空，则返回 null

```
public E poll() {
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    try {
        //返回出队元素
        return dequeue();
    } finally {
        //释放锁
        lock.unlock();
    }
}
```

```
private E dequeue() {
    int n = size - 1;
    //队列为空，返回null
    if (n < 0)
        return null;
    else {
        Object[] array = queue;
        //堆顶就是我们需要的元素
        E result = (E) array[0];
        //堆中最后一个元素
        E x = (E) array[n];
        array[n] = null;
        Comparator<? super E> cmp = comparator;
        //下沉操作
        if (cmp == null)
            siftDownComparable(0, x, array, n);
        else
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;
        return result;
    }
}
```

在前面分析中我们就知道了，队列中，第一个元素（堆顶）就是我们需要的元素。

```
/**
 * Inserts item x at position k, maintaining heap invariant by
 * demoting x down the tree repeatedly until it is less than or
 * equal to its children or is a leaf.
 *
 * @param k the position to fill
 * @param x the item to insert
 * @param array the heap array
 * @param n heap size
 */
private static <T> void siftDownComparable(int k, T x, Object[] array,
                                           int n) {
    if (n > 0) {
        Comparable<? super T> key = (Comparable<? super T>)x;
        int half = n >>> 1;           // loop while a non-leaf
        while (k < half) {
            // 2*k+1 表示的k的左孩子的位置
            int child = (k << 1) + 1; // assume left child is least
            Object c = array[child];
            // 2*k+2 右孩子位置
            int right = child + 1;
            //取左右孩子中元素值较小的值（这里的较小，是通过比较器来定义的较小）
            if (right < n &&
                    ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                c = array[child = right];
            //x 比左右孩子都小，那么不用继续下沉了    
            if (key.compareTo((T) c) <= 0)
                break;
            //下沉    
            array[k] = c;
            k = child;
        }
        array[k] = key;
    }
}
```

这个方法就是我们前面分析二叉堆的下沉的方法，因为上面我们给了图解的，因此这里就不在进行图解了，结合图应该还是很好理解了，这里需要说的是while的循环条件

```
 int half = n >>> 1; // loop while a non-leaf
  while (k < half) {
```
n是我们的队列中元素个数-1,因为数组是总下标0开始存的，因此n-1 就是最后一个元素的下标。对于任何一个角标i来说，2*i+1 就是左孩子的下标，因为二叉树是完全二叉树结构，因此有右孩子就必定有左孩子，有左孩子不一定有右孩子，没左孩子必定没右孩子，因此2*i+1 <=n,及 i<=(n-1)/2,当然 i<=n/2,因为叶子节点没有孩子，不需要再下沉了，进行i<n/2就可以了（这里的i 就相当于k）

#### 2、poll(long timeout, TimeUnit unit) 
获取并移除此队列的头部，在指定的等待时间前等待。

```
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    //可中断获取锁
    lock.lockInterruptibly();
    E result;
    try {
        超时等待
        while ( (result = dequeue()) == null && nanos > 0)
            nanos = notEmpty.awaitNanos(nanos);
    } finally {
        lock.unlock();
    }
    return result;
}
```

#### 3、take() : 
获取并移除此队列的头部，在元素变得可用之前一直等待

```
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        //如果队列为空，则阻塞在notEmpty条件上
        while ( (result = dequeue()) == null)
            notEmpty.await();
    } finally {
        lock.unlock();
    }
    return result;
}
```

4、peek() 
调用此方法，可以返回队头元素，但是元素并不出队。
```
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (size == 0) ? null : (E) queue[0];
    } finally {
        lock.unlock();
    }
}
```

这些方法都很简单，前面也分析过，这里就不多讲了。

### 回顾
在前面我们遗留了一个问题，就是用集合初始化PriorityBlockingQueue，现在我们回过头来看看

```
public PriorityBlockingQueue(Collection<? extends E> c) {
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    //是否需要将堆进行有序化
    boolean heapify = true; // true if not known to be in heap order
    //扫描null 值，保证队列中不会有null 元素
    boolean screen = true;  // true if must screen for nulls
    if (c instanceof SortedSet<?>) {

        SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
        this.comparator = (Comparator<? super E>) ss.comparator();
        //SortedSet 本身是有序的，因此不用进行堆有序化
        heapify = false;
    }
    else if (c instanceof PriorityBlockingQueue<?>) {
        PriorityBlockingQueue<? extends E> pq =
                (PriorityBlockingQueue<? extends E>) c;
        this.comparator = (Comparator<? super E>) pq.comparator();
        //PriorityBlockingQueue 本身就不会存null 值，因此不用再次扫描
        screen = false;
        //如果已经是本身类结构，那么也无需再次堆有序化
        if (pq.getClass() == PriorityBlockingQueue.class) // exact match
            heapify = false;
    }
    Object[] a = c.toArray();
    int n = a.length;
    // If c.toArray incorrectly doesn't return Object[], copy it.
    //拷贝元素
    if (a.getClass() != Object[].class)
        a = Arrays.copyOf(a, n, Object[].class);
    //扫描集合，不允许出现null 
    if (screen && (n == 1 || this.comparator != null)) {
        for (int i = 0; i < n; ++i)
            if (a[i] == null)
                throw new NullPointerException();
    }
    this.queue = a;
    this.size = n;
    if (heapify)
        heapify(); //堆有序化
}
```

从集合中初始化PriorityBlockingQueue，需要进行判断

1、是否需要进行有序化，PriorityBlockingQueue，SortedSet 本身有序，无需再进行有序化

2、是否进行集合扫描，保证队列中不存储null 值元素

下面来看看有序化方法heapify

    private void heapify() {
        Object[] array = queue;
        int n = size;
        //非叶子节点并且编号最大的节点
        int half = (n >>> 1) - 1;
        Comparator<? super E> cmp = comparator;
        if (cmp == null) {
            //对每个元素进行下沉操作
            for (int i = half; i >= 0; i--)
                siftDownComparable(i, (E) array[i], array, n);
        }
        else {
            for (int i = half; i >= 0; i--)
                siftDownUsingComparator(i, (E) array[i], array, n, cmp);
        }
    }
在前面二叉堆我们说了，有序化，就是不断的进行上浮操作，而上浮从非叶子节点并且编号最大的节点开始调整，而这个编号怎么求呢，就是：n/2-1.

这里我们看到，实际上进行的是下沉操作，并不是上浮，其实这样是一样的，其思想就是：从从非叶子节点并且编号最大的节点开始向上调整，依次对元素进行下沉，那么该元素下沉，那么其它元素自然也就上浮了，只是看待的元素不一样而已。

### 序列化
在前面优先队列数据结构分析中，我们展示过PriorityBlockingQueue 中的属性，这些属性大部分都被transient 修饰，导致了不能进行默认的序列化，那么必然就需要进行手动序列化，这个我们可以从writeObject 中得出答案

```
private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
    lock.lock();
    try {
        // avoid zero capacity argument
        //生成一个新的队列
        q = new PriorityQueue<E>(Math.max(size, 1), comparator);
        //将队列中的元素添加到新的队列中
        q.addAll(this);
        //序列化新的队列
        s.defaultWriteObject();
    } finally {
        q = null;
        lock.unlock();
    }
}
```

这么做的用意就在于，队列的容量 >= 队列中元素的个数，为了避免把没必要的null 值序列化，因此就重新生成一个队列，避免过多的null 值被序列化，这种思想其实在前面各种集合框架中都用到了。


###  总结

1、PriorityBlockingQueue 是基于二叉堆来实现的，二叉堆底层用的是数组来进行存储

2、PriorityBlockingQueue 不能存储null 值（队列里面的元素一般具有意义）

3、PriorityBlockingQueue 通过一个重入锁来控制入队和出队操作，线程安全

4、PriorityBlockingQueue 是FIFO队列，但是该FIFO是基于优先级的，通过默认比较器 比较结果中 较小的元素靠近队头（优先级高），当然我们可以通过自定义比较器来实现排队规则。

5、PriorityBlockingQueue 中没有队满的概念，当队列满后，会进行扩容，当操作队列最大值后（Integer.MAX_VALUE - 8），将抛出oom异常，同时入队没有队满操时等待和队满阻塞操作，当队列达到最大值，如果继续入队，则会抛oom异常，这一点需要注意，在使用中，避免大量的元素不断入队，入队速度快，而出队速度又很慢。

6、总算写完了，比ArrayBlockingQueue，LinkedBlockingQueue有意思。

