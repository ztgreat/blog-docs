前面我们分析了ArrayBlockingQueue，ArrayBlockingQueue是基于数组来实现的阻塞队列，通过可重入锁和Condition 实现了ArrayBlockingQueue的线程安全和条件阻塞，说到数组，一般就会有链表，今天我们一起来看看另一个阻塞队列---LinkedBlockingQueue

在看本文之前，最好要有 AbstractQueuedSynchronizer,ReentrantLock,Condition的知识，下面是我个人分析的博客，仅供参考： 
[Java 并发 —AbstractQueuedSynchronizer-共享模式与Condition](http://blog.ztgreat.cn/article/9) 
[Java 并发 —ReentrantLock源码分析](http://blog.ztgreat.cn/article/14)

### LinkedBlockingQueue 介绍（jdk 1.8）
LinkedBlockingQueue 是一个用链表实现的有界阻塞队列，此队列按照先进先出（FIFO）的原则对元素进行操作。 

### 继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171023215338712.png)

LinkedBlockingQueue 实现了BlockingQueue接口，该接口中定义了阻塞的方法接口， 

LinkedBlockingQueue 继承了AbstractQueue，具有了队列的行为。 

LinkedBlockingQueue 实现了Serializable接口,可以序列化。

### 数据结构

```
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
   
    //队列存储节点
    static class Node<E> {
        E item;  //元素值
        //节点后继
        Node<E> next;
        Node(E x) { item = x; }
    }

    /** The capacity bound, or Integer.MAX_VALUE if none 队列容量  */
    private final int capacity;

    /** Current number of elements 队列含有的数据个数 */
    private final AtomicInteger count = new AtomicInteger();

    //队列头，默认不会序列化 （transient 修饰）
    transient Node<E> head;

   //队列尾 ，默认不会序列化（transient 修饰）
    private transient Node<E> last;

    //用于出队列的可重入锁,使用非公平锁
    private final ReentrantLock takeLock = new ReentrantLock();

    // 队列不为空的条件
    private final Condition notEmpty = takeLock.newCondition();

    //用于入队的可重入锁，使用非公平锁 
    private final ReentrantLock putLock = new ReentrantLock();

    // 队列不满的条件
    private final Condition notFull = putLock.newCondition();
	...
 }
```

可重入锁是独占式的锁，如果用可重入锁来控制对 队列的操作访问，那么此队列将是线程安全的，阻塞操作，那么什么情况下该阻塞，什么情况下不阻塞，这个是由Condition来控制的。 
notEmpty ：表示的是队列不为空，符合这种条件那么可以进行出队操作，否则将会阻塞，直到队列不为空为止。

notFull： 表示是队列没有满，符合这种条件的可以进行入队操作，否则将会阻塞。

在ArrayBlockingQueue中，入队和出队都是用的同一个可重入锁，而LinkedBlockingQueue 对于出队和入队使用了不同的可重入锁来控制，ArrayBlockingQueue 入队和出队是不能同时并发的，而在LinkedBlockingQueue 中出队和出队是可以同时并发执行的（锁不一样）。
正是如此，对于count（队列中元素的个数）使用了原子类AtomicInteger，来保证对count的操作具有原子性。

### 构造方法
1、默认构造方法

```
/**
 * Creates a {@code LinkedBlockingQueue} with a capacity of
 * {@link Integer#MAX_VALUE}.
 */
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}
```

在LinkedBlockingQueue 中可以不用指定队列大小，默认是用Integer.MAX_VALUE 来作为队列的大小。

2、指定队列大小

```
/**
 * Creates a {@code LinkedBlockingQueue} with the given (fixed) capacity.
 *
 * @param capacity the capacity of this queue
 * @throws IllegalArgumentException if {@code capacity} is not greater
 *         than zero
 */
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    //初始化 队头和队尾
    last = head = new Node<E>(null);
}
```

队头和队尾初始化为一个空节点（不存储元素值的节点）

3、通过集合初始化

```
public LinkedBlockingQueue(Collection<? extends E> c) {
    //设置队列大小为Integer.MAX_VALUE，而不是集合的大小
    this(Integer.MAX_VALUE);
    //将集合中的元素入队，因此会用到入队锁
    final ReentrantLock putLock = this.putLock;
    //入队锁 加锁
    putLock.lock(); // Never contended, but necessary for visibility
    try {
        int n = 0;
        //遍历集合
        for (E e : c) {
            //不允许存储空值
            if (e == null)
                throw new NullPointerException();
            if (n == capacity)
                throw new IllegalStateException("Queue full");
            // 入队    
            enqueue(new Node<E>(e));
            ++n;
        }
        //设置队列实际大小
        count.set(n);
    } finally {
        //释放锁
        putLock.unlock();
    }
}
```

通过集合初始化队列，队列的大小不是集合的大小，而是Integer.MAX_VALUE，获取入队锁，然后依次向队列中添加元素，在finally 中释放锁，这样能保证就是出现异常，也能正确的释放锁。

### 入队

LinkedBlockingQueue提供了诸多方法，可以将元素加入队列尾部 

1、add(E e) 

将指定的元素插入到此队列的尾部，在成功时返回 true，如果此队列已满，则抛出 IllegalStateException

```
public boolean add(E e) {
    //调用offer 方法 完成
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
public boolean offer(E e) {
    //入队元素不能为空
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    //达到了队列的容量，则入队失败
    if (count.get() == capacity)
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    //入队锁
    final ReentrantLock putLock = this.putLock;
    //入队锁 加锁
    putLock.lock();
    try {
        //如果还可以入队，则入队
        if (count.get() < capacity) {
            enqueue(node);
            //原子增加 count的值，返回旧值（注意是返回旧值）
            c = count.getAndIncrement();
            //c+1 表示入队后的队列中元素的个数，如果此时没有队满，那么not Full条件满足，唤醒阻塞在notFull条件上的一个线程
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        //释放入队锁
        putLock.unlock();
    }
    //c 是旧值，如果c==0 表示原来队列为空，现在入队了一个元素，则不为空了，则notEmpty 条件满足，唤醒阻塞在notEmpty条件上的一个线程
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
}
```

从offer中可以看出来：LinkedBlockingQueue和ArrayBlockingQueue一样，存储的元素是不能为空的，ArrayList和LinkedList可以存储空元素。
```
/**
 * Signals a waiting take. Called only from put/offer (which do not
 * otherwise ordinarily lock takeLock.)
 */
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    //出队锁 加锁
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}
```

notEmpty 用于出队控制条件上，当队列不为空时，notEmpty 条件满足，则可以出队，因此需要获取出队锁，在入队的同时，可以出队，在出队的同时也可以出队，只要满足notEmpty 或者notFull 条件就可以了。

3、offer(E e, long timeout, TimeUnit unit) 

将指定的元素插入此队列的尾部，如果该队列已满，则在到达指定的等待时间之前等待。

```
/**
 * Inserts the specified element at the tail of this queue, waiting if
 * necessary up to the specified wait time for space to become available.
 *
 * @return {@code true} if successful, or {@code false} if
 *         the specified waiting time elapses before space is available
 * @throws InterruptedException {@inheritDoc}
 * @throws NullPointerException {@inheritDoc}
 */
public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

    //入队元素不能为空
    if (e == null) throw new NullPointerException();
    long nanos = unit.toNanos(timeout);
    int c = -1;
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    //入队锁 可中断的上锁
    putLock.lockInterruptibly();
    try {
        //如果队满，则进行限时等待
        while (count.get() == capacity) {
            //超时，返回false
            if (nanos <= 0)
                return false;
            // 未超时，notFull 条件不满足，则在notFull条件上等待
            nanos = notFull.awaitNanos(nanos);
        }
        //出队
        enqueue(new Node<E>(e));
        //c 是入队前的旧值
        c = count.getAndIncrement();
        //notFull 条件满足
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    //notEmpty条件满足
    if (c == 0)
        signalNotEmpty();
    return true;
}
```

这里在获取锁的时候和前面的不一样，lockInterruptibly 表示可以中断的，前面的加锁对中断不敏感，也就是说，在前面的获取锁的方式中，别的线程对当前线程中断，当前线程不会理会（会记录中断状态），而可中断的获取锁，其它线程中断该线程的时候，会抛出中断异常，举个例子：如果队列满了，但是超时时间还没有到，此时如果不想执行入队操作了，那么就可以中断当前线程（注意：中断不是立即取消，中断后也可能会执行入队操作，参考：[Java 并发 —中断机制](http://blog.csdn.net/u014634338/article/details/77844521)）。

4、put(E e) 

将指定的元素插入此队列的尾部，如果该队列已满，则等待。

```
/**
 * Inserts the specified element at the tail of this queue, waiting if
 * necessary for space to become available.
 *
 * @throws InterruptedException {@inheritDoc}
 * @throws NullPointerException {@inheritDoc}
 */
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    //入队锁 可中断的加锁
    putLock.lockInterruptibly();
    try {
        //如果队满，则notFull 条件不满足，阻塞在该条件上，直到条件满足，或者发生中断
        while (count.get() == capacity) {
            notFull.await();
        }
        enqueue(node);
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
}
```

当队列满时，notFull 条件不满足，因此会阻塞在该方法上（await()），这里用的while循环，因为当线程从notFull 条件阻塞中唤醒时，需要重新检查（可以此时被其它线程抢先了，导致还是不满足）。

### 出队
和入队一样，LinkedBlockingQueue 也提供了很多出队的方法。 

1、poll() 

获取并移除此队列的头，如果此队列为空，则返回 null

```
public E poll() {
    final AtomicInteger count = this.count;
    //如多队列中没有元素，返回false
    if (count.get() == 0)
        return null;
    E x = null;
    int c = -1;
    final ReentrantLock takeLock = this.takeLock;
    //出队锁 加锁
    takeLock.lock();
    try {
        if (count.get() > 0) {
            x = dequeue();
            //c 为出队前的队列大小
            c = count.getAndDecrement();
            // 元素出队前，队列>1 那么出队后就>0,notEmpty条件满足
            if (c > 1)
                notEmpty.signal();
        }
    } finally {
        //释放锁
        takeLock.unlock();
    }
    //出队前，队满，出队后则队列不满，notFull条件满足，唤醒阻塞在notFull条件上的一个线程
    if (c == capacity)
        signalNotFull();
    return x;
}
```

```
/**
 * Signals a waiting put. Called only from take/poll.
 */
private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    //入队锁 加锁
    putLock.lock();
    try {
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}
```

2、poll(long timeout, TimeUnit unit) 

获取并移除此队列的头部，在指定的等待时间前等待。

```
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E x = null;
    int c = -1;
    long nanos = unit.toNanos(timeout);
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        //如果队列为空，则进行超时等待
        while (count.get() == 0) {
            if (nanos <= 0)
                return null;
            //notEmpty 条件上阻塞    
            nanos = notEmpty.awaitNanos(nanos);
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

在poll()方法上增加超时等待，如果队列为空，则等待指定的时间，如果在超时发生前，队列不空，则可以出队，否则发生超时，返回false。
因为可以超时等待，因此使用的可以中断的上锁，同时抛出中断异常，当在超时未发生时，如果不想再继续等待，那么就可以中断该线程，让其从阻塞等待中返回（**中断并不一定就可以马上取消任务**）。

3、take() : 

获取并移除此队列的头部，在元素变得可用之前一直等待

```
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    // 出队锁，可中断加锁
    takeLock.lockInterruptibly();
    try {
        // 如果队空，则notEmpty 条件不满足，阻塞在notEmpty条件上，直到队列不空，或者发生中断。
        while (count.get() == 0) {
            notEmpty.await();
        }
        //出队
        x = dequeue();
        c = count.getAndDecrement();
        //notEmpty 条件满足
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    //notFull 条件满足
    if (c == capacity)
        signalNotFull();
    return x;
}
```

4、peek()

调用此方法，可以返回队头元素，但是元素并不出队。

```
public E peek() {
    //队列为空，返回false
    if (count.get() == 0)
        return null;
    final ReentrantLock takeLock = this.takeLock;
    //虽然不会出队，但是会访问队头元素，因此还是需要加锁
    takeLock.lock();
    try {
        //head 是一个空节点（不存储具体元素值的节点），因此head.next 才是第一个元素节点
        Node<E> first = head.next;
        if (first == null)  // 队空，返回false 
            return null;
        else
            return first.item;  //返回元素组
    } finally {
        takeLock.unlock();
    }
}
```

在前面我们虽然判断了队列是否为空，但是仍然可能出现队列为空，为什么呢，因为我们判断队列空的地方，并没有获取出队锁，那么如果队列此时只有一个元素，那么队列不为空，在出队锁加锁前，如果其他线程抢先了一步进行了出队，那么此时队列变为空了，当前线程是感应不到的。

### 序列化

在LinkedBlockingQueue 我们看到其部分属性被声明为了transient，被transient 修饰的属性默认不会被序列化.
因为该阻塞队列是基于链表实现的，物理上是不连续的，因此不能通过ArrayBlockingQueue 那么来默认序列化，需要我们自己"手动"序列化。

```
private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
    //出队锁，入队锁都 加锁
    fullyLock();
    try {
        // Write out any hidden stuff, plus capacity
        //先进行默认序列化工作
        s.defaultWriteObject();

        // Write out all elements in the proper order.
        //将队列中的元素，依次进行序列化
        for (Node<E> p = head.next; p != null; p = p.next)
            s.writeObject(p.item);

        // Use trailing null as sentinel
        //标记队列序列化结束，可以结合反序列化来看
        s.writeObject(null);
    } finally {
        fullyUnlock();
    }
}
```

writeObject 就是序列化过程，这个并不复制，就是把队列中的数据依次序列化即可，反序列化就是一个逆过程。

```
private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
    // Read in capacity, and any hidden stuff
    //进行默认反序列化
    s.defaultReadObject();

    //初始化队列大小为0
    count.set(0);
    last = head = new Node<E>(null);

    // Read in all elements and place in queue
    for (;;) {
        //不断读取元素，添加到队列中，知道读到null 值。
        E item = (E)s.readObject();
        if (item == null)
            break;
        add(item);
    }
}
```

### 总结

1、LinkedBlockingQueue 底层是基于链表实现的队列，容量指定后，不会改变，可以不指定容量，默认容量为Integer.MAX_VALUE。

2、LinkedBlockingQueue 线程安全，和Vector不一样，Vector 中用的synchronized
关键字进行线程同步，LinkedBlockingQueue 中通过ReentrantLock来完成的。

3、LinkedBlockingQueue和ArrayBlockingQueue 不一样，前者对于出队和入队使用了不同的可重入锁来完成，后者使用了一个可重入锁来完成，因此前者的并发高于后者，后者比前者更线程安全，LinkedBlockingQueue 并不完全线程安全，因为入队和出队是不同的锁，因此二者可以并发执行。


