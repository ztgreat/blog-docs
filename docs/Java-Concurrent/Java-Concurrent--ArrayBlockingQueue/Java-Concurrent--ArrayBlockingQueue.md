队列是一种FIFO(先进先出)数据结构，在前面我们知道分析过LinkedList的源码，LinkedList可以作为一般的队列使用，既然有阻塞队列，那么肯定就和一般的队列是有不一样的地方，并且使用场景也可能不一样，一起来探究一下阻塞队列的源码。

在看本文之前，要有 AbstractQueuedSynchronizer,ReentrantLock,Condition的知识，下面是我个人分析的博客，仅供参考：
[Java 并发 ---AbstractQueuedSynchronizer-共享模式与Condition](http://blog.ztgreat.cn/article/9)
[Java 并发 ---ReentrantLock源码分析](http://blog.ztgreat.cn/article/14)

 阻塞队列是一个支持两个附加操作的队列，这两个附加操作支持阻塞的插入和移除方法。

1、支持阻塞的插入方法: 意思是当队列满时，队列会阻塞插入元素的线程，直到队列不满。

2、支持阻塞的移除方法:意思是在队列为空时，获取元素的线程会等待队列变为非空。

### ArrayBlockingQueue介绍(jdk 1.8)

ArrayBlockingQueue 是一个用数组实现的有界阻塞队列，此队列按照先进先出（FIFO）的原则对元素进行操作。
### 继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171015105238013.png)

ArrayBlockingQueue 实现了BlockingQueue接口，该接口中定义了阻塞的方法接口，

ArrayBlockingQueue 继承了AbstractQueue，具有了队列的行为。

ArrayBlockingQueue 实现了Serializable接口,可以序列化。

### 数据结构

```
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    private static final long serialVersionUID = -817911632652898426L;

    /** The queued items ，底层存储元素的数组*/
    final Object[] items;

    /** items index for next take, poll, peek or remove ，队首的索引（出队）*/
    int takeIndex;

    /** items index for next put, offer, or add ，队尾的索引（入队）*/
    int putIndex;

    /** Number of elements in the queue ，队列中元素的个数*/
    int count;

    /*
     * Concurrency control uses the classic two-condition algorithm
     * found in any textbook.
     */
    /** Main lock guarding all access ，重入锁*/
    final ReentrantLock lock;

    /** Condition for waiting takes ，出队的条件*/
    private final Condition notEmpty;

    /** Condition for waiting puts ，入队的条件*/
    private final Condition notFull;

    /**
     * Shared state for currently active iterators, or null if there
     * are known not to be any.  Allows queue operations to update
     * iterator state.
     */
    transient Itrs itrs = null;
}
```
相比传统队列，ArrayBlockingQueue 中有三个重要的属性，可重入锁和Condition，可重入锁是独占式的锁，如果用可重入锁来控制对 队列的操作访问，那么此队列将是线程安全的，阻塞操作，那么什么情况下该阻塞，什么情况下不阻塞，这个是由Condition来控制的。
因此通过重入锁和Condition 实现了ArrayBlockingQueue的线程安全和条件阻塞。

notEmpty ：表示的是队列不为空，符合这种条件那么可以进行出队操作，否则将会阻塞，直到队列不为空为止。

notFull： 表示是队列没有满，符合这种条件的可以进行入队操作，否则将会阻塞。

### 构造方法
1、
```
public ArrayBlockingQueue(int capacity) {
    //默认使用非公平锁
    this(capacity, false);
}
```

```
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```

ArrayBlockingQueue 因为是有界的队列，因此需要指定队列的大小，它不会像ArrayList那样自动扩容，ReentrantLock 有公平锁和非公平锁之分，因此需要指定，默认是非公平锁，ReentrantLock 我们前面也分析过，对于公平锁，那么会遵循获取锁的FIFO规则，先阻塞的线程先获取锁，对于非公平锁，那么第一次获取锁的时候，会进行抢占式获取锁，也就是不管等待队列中是否有等待线程，同样参与竞争获取锁，如果获取失败，加入到等待队列，那么以后遵循FIFO规则。（注意：这里的等待队列不是指的ArrayBlockingQueue，而是同步器内部的队列）

2、指定集合初始化

```
public ArrayBlockingQueue(int capacity, boolean fair,
                          Collection<? extends E> c) {
    this(capacity, fair);

    final ReentrantLock lock = this.lock;
    lock.lock(); // Lock only for visibility, not mutual exclusion
    try {
        int i = 0;
        try {
            for (E e : c) {
                checkNotNull(e);
                items[i++] = e;
            }
        } catch (ArrayIndexOutOfBoundsException ex) {
            throw new IllegalArgumentException();
        }
        count = i;
        putIndex = (i == capacity) ? 0 : i;
    } finally {
        lock.unlock();
    }
}
```

处理指定队列大小，锁的公平性，还可以通过一个集合来初始化队列，在队列产生后，会把集合中的元素依次添加到队列中，初始集合的元素大小要和队列的容量一致，同时对队列的操作进行了加锁，保证了线程安全性，在finally 中释放锁，这样能保证就是出现异常，也能正确的释放锁。

### 入队
ArrayBlockingQueue提供了诸多方法，可以将元素加入队列尾部
1、add(E e)

将指定的元素插入到此队列的尾部，在成功时返回 true，如果此队列已满，则抛出 IllegalStateException

```
public boolean add(E e) {
    return super.add(e);
}
//super.add(e)
public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}
```

2、offer(E e) 

将指定的元素插入到此队列的尾部在成功时返回 true，如果此队列已满，则返回 false。
在前面的add 中，内部调用了offer 方法，我们也可以直接调用offer 方法来完成入队操作。

```
/**
 * Inserts the specified element at the tail of this queue if it is
 * possible to do so immediately without exceeding the queue's capacity,
 * returning {@code true} upon success and {@code false} if this queue
 * is full.  This method is generally preferable to method {@link #add},
 * which can fail to insert an element only by throwing an exception.
 *
 * @throws NullPointerException if the specified element is null
 */
public boolean offer(E e) {
    //判断元素是否为空，如果为空，会抛出空指针异常
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    try {
        //如果队列已经满了，返回false
        if (count == items.length)
            return false;
        else {
            //将元素入队
            enqueue(e);
            return true;
        }
    } finally {
        //释放锁
        lock.unlock();
    }
}
```

从offer中可以看出来：ArrayBlockingQueue存储的元素是不能为空的，ArrayList和LinkedList可以存储空元素，如果队列满了不会阻塞，直接会返回false。
```
/**
 * Inserts element at current put position, advances, and signals.
 * Call only when holding lock.
 */
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x;
    //如果入队后，队列"满"了，那么putIndex=0
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    //队列不为空满足，唤醒阻塞在队列为空条件上的一个线程。
    notEmpty.signal();
}
```

putIndex指示的是入队元素的存储位置，当队列满后，putIndex=0，可以看出，这是环形队列的用法，队尾不一定要是物理上的队列末尾，而是逻辑上的队尾，通过这种环形队列的用法，可以减少不必要的元素拷贝（元素出队以后，不用把元素整体往前移动），这个可以结合后面出队来看。

3、offer(E e, long timeout, TimeUnit unit) 

将指定的元素插入此队列的尾部，如果该队列已满，则在到达指定的等待时间之前等待。

```
/**
 * Inserts the specified element at the tail of this queue, waiting
 * up to the specified wait time for space to become available if
 * the queue is full.
 *
 * @throws InterruptedException {@inheritDoc}
 * @throws NullPointerException {@inheritDoc}
 */
public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

    checkNotNull(e);
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    //加锁，可中断
    lock.lockInterruptibly();
    try {
        //如果队满，则等待
        while (count == items.length) {
            //超时，返回false
            if (nanos <= 0)
                return false;
            //（未超时）等待
            nanos = notFull.awaitNanos(nanos);
        }
        enqueue(e);
        return true;
    } finally {
        lock.unlock();
    }
}
```

这里在获取锁的时候和前面的不一样，lockInterruptibly 表示可以中断的，前面的加锁对中断不敏感，也就是说，在前面的获取锁的方式中，别的线程对当前线程中断，当前线程不会理会（会记录中断状态），而可中断的获取锁，其它线程中断该线程的时候，会抛出中断异常，举个例子：如果队列满了，但是超时时间还没有到，此时如果不想执行入队操作了，那么就可以中断当前线程（注意：**中断不是立即取消，中断后也可能会执行入队操作**

参考：[Java 并发 ---中断机制](http://blog.ztgreat.cn/article/14)）。

4、put(E e) 

将指定的元素插入此队列的尾部，如果该队列已满，则等待。

```
/**
 * Inserts the specified element at the tail of this queue, waiting
 * for space to become available if the queue is full.
 *
 * @throws InterruptedException {@inheritDoc}
 * @throws NullPointerException {@inheritDoc}
 */
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    //获取锁，可中断
    lock.lockInterruptibly();
    try {
        //如果队列满，则阻塞在notFull 条件上
        while (count == items.length)
            notFull.await();
        //入队
        enqueue(e);
    } finally {
        //释放锁
        lock.unlock();
    }
}
```

当队列满时，notFull 条件不满足，因此会阻塞在该方法上（await()），这里用的while循环，因为当线程从notFull 条件阻塞中唤醒时，需要重新检查（可以此时被其它线程抢先了，导致还是不满足）。
### 出队
和入队一样，ArrayBlockingQueue 也提供了很多出队的方法。

1、poll() 

获取并移除此队列的头，如果此队列为空，则返回 null

```
public E poll() {
    final ReentrantLock lock = this.lock;
    //获取锁
    lock.lock();
    try {
        //队列为空返回null,否则返回队头
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}
```

```
/**
 * Extracts element at current take position, advances, and signals.
 * Call only when holding lock.
 */
private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    //获取队头元素
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    //新队头位置(队"空")
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    //维护迭代器
    if (itrs != null)
        itrs.elementDequeued();
    //出队了一个元素，队列不满条件满足，唤醒阻塞在队满条件上的一个线程    
    notFull.signal();
    return x;
}
```

从这里出队和前面的入队相结合，可以看出队列是一种**逻辑环形队列**，队"满"(++putIndex == items.length)，队"空"(++takeIndex == items.length)并不是严格意义上的队空，队满，我从网上找了一个环形队列的图，稍稍修改了一下：
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171015205248575.jpg)
对于迭代器部分，这里不会讲，后面会专门来分析迭代器。

2、poll(long timeout, TimeUnit unit) 

获取并移除此队列的头部，在指定的等待时间前等待。

```
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    //获取锁，可中断
    lock.lockInterruptibly();
    try {
        //队空
        while (count == 0) {
            //超时
            if (nanos <= 0)
                return null;
            //（未超时）等待
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

3、take() :

获取并移除此队列的头部，在元素变得可用之前一直等待

```
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    //可中断的获取锁
    lock.lockInterruptibly();
    try {
        //如果队列空，那么阻塞等待
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

4、peek()

返回队头的元素，但是元素并不会出队。
```
public E peek() {
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    try {
        return itemAt(takeIndex); // null when queue is empty
    } finally {
        //释放锁
        lock.unlock();
    }
}
```

```
final E itemAt(int i) {
    return (E) items[i];
}
```

因为底层是数组结构，因此通过索引可以直接访问到队头。

5、remove(Object o) 

从此队列中移除指定元素的单个实例

```
public boolean remove(Object o) {
    if (o == null) return false;
    final Object[] items = this.items;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //队列中有元素
        if (count > 0) {
            final int putIndex = this.putIndex;
            int i = takeIndex;
            do {
                if (o.equals(items[i])) {
                    //移除
                    removeAt(i);
                    return true;
                }
                //循环
                if (++i == items.length)
                    i = 0;
            } while (i != putIndex);  //循环到达队尾为止
        }
        return false;
    } finally {
        lock.unlock();
    }
}
```

```
/**
 * Deletes item at array index removeIndex.
 * Utility for remove(Object) and iterator.remove.
 * Call only when holding lock.
 */
void removeAt(final int removeIndex) {

    final Object[] items = this.items;
    //需要移除元素的位置就是下个需要出队元素的位置，那么和一般出队方法一样
    if (removeIndex == takeIndex) {
        // removing front item; just advance
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
    } else {
        // an "interior" remove

        // slide over all others up through putIndex.
        final int putIndex = this.putIndex;
        //移动队列中的元素
        for (int i = removeIndex;;) {
            //下一个位置索引
            int next = i + 1;
            if (next == items.length)
                next = 0;
            //还不是队尾，依次"前移"
            if (next != putIndex) {
                items[i] = items[next];
                i = next;
            } else {
                //移动元素完毕，从新设置队尾索引
                items[i] = null;
                this.putIndex = i;
                break;
            }
        }
        count--;
        //维护迭代器
        if (itrs != null)
            itrs.removedAt(removeIndex);
    }
    //唤醒阻塞在队列满条件下的一个线程
    notFull.signal();
}
```

这种移除队列中的元素的方式，就相当于删除数组中 非两端的元素一样，需要移动数组中的元素，因此这种方式需要相对开销要大点，对于
ArrayBlockingQueue 尽量不要操作非队头和队尾的元素。

### 总结

1、ArrayBlockingQueue 底层是基于数组实现的队列，容量指定后，不会改变。

2、ArrayBlockingQueue 线程安全，和Vector不一样，Vector 中用的synchronized 关键字进行线程同步，ArrayBlockingQueue 中通过ReentrantLock来完成的。

3、ArrayBlockingQueue 中的ReentrantLock 有公平和非公平之分，因此ArrayBlockingQueue 相当于也有公平性和非公平性之分。

4、ArrayBlockingQueue 比一般的队列多了两个附加操作（阻塞式的插入和移除方法），这个依赖于Condition实现。

5、ArrayBlockingQueue 是一种逻辑上的环形队列。

6、ArrayBlockingQueue 在入队和出队上都使用了同一个重入锁，因此入队和出队是不能并发执行的。

