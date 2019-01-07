在分析完LinkedTransferQueue后我们来看java中最后一个阻塞队列（jdk 1.8）LinkedBlockingDeque ,LinkedTransferQueue,SynchronousQueue都相对的比较难，但是再难我们也走过来了，而今天的LinkedBlockingDeque 则非常的简单。
### LinkedBlockingDeque 介绍（jdk 1.8）
LinkedBlockingDeque是基于**双向链表的双端**有界阻塞队列，默认使用非公平ReentrantLock实现线程安全，默认队列最大长度都为Integer.MAX_VALUE（这种不也可以称为无界队列么）；不允许null元素添加；双端队列可以用来实现 "窃取算法" ,两头都可以操作队列，相对于单端队列可以减少一半的竞争。
LinkedBlockingDeque 是基于链表的，因此分析它实际就是分析链表而已，这个和我们前面LinkedBlockingQueue实质是差不多的，只是这里是双端队列，可以两头操作队列（队尾也可以出队，队头也可以入队）。

### 继承体系
![LinkedBlockingDeque继承体系](http://img.blog.ztgreat.cn/document/juc/20171120115048461.png)

LinkedBlockingDeque实现了BlockingDeque接口，代表的是阻塞的双端队列接口 
LinkedBlockingDeque继承了AbstractQueue，具有了抽象的队列的行为。 
LinkedBlockingDeque实现了Serializable接口,可以序列化。
BlockingDeque 继承BlockingQueue，Deque，也就是实际的扩展了BlockingQueue接口，拥有了双端队列的行为。

### 数据结构
队列中节点的定义：
```
static final class Node<E> {
    /**
     * The item, or null if this node has been removed.
     */
    E item; //数据域

	//前驱节点指针
    Node<E> prev;
    
    //后继节点指针
    Node<E> next;

    Node(E x) {
        item = x;
    }
}
```
LinkedBlockingDeque 是基于双链表的，因此毫无疑问其队列节点也必然和双链表节点是差不多的。

```
//队头指针
transient Node<E> first;

队尾指针
transient Node<E> last;

/** Number of items in the deque */
private transient int count;

/** Maximum number of items in the deque */
private final int capacity;

//队列访问锁
final ReentrantLock lock = new ReentrantLock();

/** Condition for waiting takes 队列非空条件 */
private final Condition notEmpty = lock.newCondition();

/** Condition for waiting puts 队列不满条件*/
private final Condition notFull = lock.newCondition();
```
### 构造方法
1、默认构造

```
public LinkedBlockingDeque() {
    this(Integer.MAX_VALUE); //会默认指定队列的容量为Integer.MAX_VALUE
}
```
2、指定队列容量
```
public LinkedBlockingDeque(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
}
```
3、通过集合构造

```
public LinkedBlockingDeque(Collection<? extends E> c) {
    this(Integer.MAX_VALUE);// 队列容量设置为最大
    final ReentrantLock lock = this.lock;
    lock.lock(); // 操作队列需要加锁
    try {
        for (E e : c) {
            if (e == null)
                throw new NullPointerException();
            //将元素添加到队列末尾    
            if (!linkLast(new Node<E>(e)))
                throw new IllegalStateException("Deque full");
        }
    } finally {
        //释放锁
        lock.unlock();
    }
}
```

### 入队
#### 操作队尾
1、add(E e) 
将指定的元素插入到此队列的尾部，在成功时返回 true，如果此队列已满，则抛出 IllegalStateException

```
public boolean add(E e) {
    addLast(e);
    return true;
}
```
内部调用addLast，addLast内部调用offerLast
```
public void addLast(E e) {
    if (!offerLast(e))
        throw new IllegalStateException("Deque full");
}
```
offerLast 内部又调用linkLast
```
public boolean offerLast(E e) {
    //入队元素不能为空
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    final ReentrantLock lock = this.lock;
    //操作队列，加锁
    lock.lock();
    try {
        return linkLast(node);
    } finally {
        //释放锁
        lock.unlock();
    }
}
```
```
/**
 * Links node as last element, or returns false if full.
 */
private boolean linkLast(Node<E> node) {
    //超过最大容量了，操作失败
    if (count >= capacity)
        return false;
    Node<E> l = last;
    node.prev = l;
    last = node;
    //如果队头尾空，赋值给队头
    if (first == null)
        first = node;
    else
        l.next = node; //连接至队列尾
    ++count; //元素数量增加
    notEmpty.signal();//队列非空条件满足，唤醒阻塞在队列空上的一个线程
    return true;
}
```
2、offer(E e) 
将指定的元素插入到此队列的尾部，在成功时返回 true，如果此队列已满，则抛出 IllegalStateException

```
public boolean offer(E e) {
    return offerLast(e);
}
```
offer 内部调用offerLast，offerLast内部同样调用的是linkLast 方法。
```
public boolean offerLast(E e) {
     if (e == null) throw new NullPointerException();
     Node<E> node = new Node<E>(e);
     final ReentrantLock lock = this.lock;
     //操作队列，加锁
     lock.lock();
     try {
         return linkLast(node);
     } finally {
         lock.unlock();
     }
 }
```


3、offer(E e, long timeout, TimeUnit unit) 
将指定的元素插入此队列的尾部，如果该队列已满，则在到达指定的等待时间之前等待（如果发生中断，则抛出中断异常）
```
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    return offerLast(e, timeout, unit);
}
```

```
public boolean offerLast(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    //纳秒数
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    //可中断的 加锁
    lock.lockInterruptibly();
    try {
        //如果添加节点至队列末尾失败，则队列满了
        while (!linkLast(node)) {
            if (nanos <= 0) //如果超时时间到了，则返回false
                return false;
            nanos = notFull.awaitNanos(nanos);// 阻塞等待，等待nanos指定的纳秒数
        }
        return true;
    } finally {
        lock.unlock();
    }
}
```
**2、put(E e)** 
将指定的元素插入此队列的尾部，如果该队列已满，则阻塞等待。
```
public void put(E e) throws InterruptedException {
    putLast(e);
}
```
```
public void putLast(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    final ReentrantLock lock = this.lock;
    //操作队列，加锁
    lock.lock();
    try {
        //如果添加到队列末尾失败，则说明队列已经满了
        while (!linkLast(node))
            notFull.await(); //notFull条件不满足，阻塞在该条件上
    } finally {
        //释放锁
        lock.unlock();
    }
}
```

#### 操作队头
1、push(E e)
将元素添加至队列头部
```
public void push(E e) {
    addFirst(e);
}
```
调用addFirst
```
public void addFirst(E e) {
    if (!offerFirst(e))
        throw new IllegalStateException("Deque full");
}
```
调用offerFirst，如果失败则抛出IllegalStateException
```
public boolean offerFirst(E e) {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    try {
        return linkFirst(node);
    } finally {
        lock.unlock();
    }
}
```
调用linkFirst
```
//将元素添加到队列头部
private boolean linkFirst(Node<E> node) {
    // assert lock.isHeldByCurrentThread();
    if (count >= capacity)
        return false;
    Node<E> f = first;
    node.next = f; //设置后继
    first = node;
    if (last == null)
        last = node;
    else
        f.prev = node; //设置前驱
    ++count;
    notEmpty.signal(); //notEmpty 条件满足，唤醒阻塞在队列为空条件上的一个线程
    return true;
}
```
### 出队

#### 操作队头
1、poll() 
获取并移除此队列的头，如果此队列为空，则返回 null
```
public E poll() {
    return pollFirst();
}
```
```
public E pollFirst() {
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    try {
        return unlinkFirst(); //移除并返回对头
    } finally {
        lock.unlock();
    }
}
```
```
/**
 * Removes and returns first element, or null if empty.
 */
private E unlinkFirst() {
    // assert lock.isHeldByCurrentThread();
    Node<E> f = first;
    if (f == null)
        return null; //队列为空，返回null
    Node<E> n = f.next;
    E item = f.item;
    f.item = null;
    f.next = f; // help GC
    first = n;
    if (n == null) //出队后，队列为空
        last = null;
    else
        n.prev = null; //清空新head的前驱指针
    --count;
    notFull.signal(); //现在队列notFull条件满足，唤醒阻塞在队列满了上的一个线程
    return item;
}
```
2、poll(long timeout, TimeUnit unit) 
获取并移除此队列的头部，在指定的等待时间前等待（如果发生中断，则抛出中断异常）
```
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    return pollFirst(timeout, unit);
}
```
```
public E pollFirst(long timeout, TimeUnit unit)
    throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    //可中断的加锁
    lock.lockInterruptibly();
    try {
        E x;
        //如果移除节点失败，说明队列为空
        while ( (x = unlinkFirst()) == null) {
            if (nanos <= 0) //超时时间到，返回null
                return null;
            nanos = notEmpty.awaitNanos(nanos); //在notEmpty 条件上 阻塞等待nanos 纳秒
        }
        return x;
    } finally {
        lock.unlock();
    }
}
```

3、take() : 
获取并移除此队列的头部，在元素变得可用之前一直等待
```
public E take() throws InterruptedException {
    return takeFirst();
}
```
```
public E takeFirst() throws InterruptedException {
   final ReentrantLock lock = this.lock;
   lock.lock();
   try {
       E x;
       //移除元素失败，说明队列为空
       while ( (x = unlinkFirst()) == null)
           notEmpty.await();//在notEmpty 条件阻塞等待
       return x;
   } finally {
       lock.unlock();
   }
}
```
4、remove 方法
移除并返回队头，如果队列为空，则会抛出NoSuchElementException异常
```
public E remove() {
    return removeFirst();
}
```
```
public E removeFirst() {
    E x = pollFirst();
    if (x == null) throw new NoSuchElementException();
    return x;
}
```
5、pop方法
移除并返回队头元素
```
public E pop() {
    return removeFirst();
}
```
#### 操作队尾
1、takeLast()
取队尾元素，如果队列为空，则阻塞等待，直到队列不空，或者发生中断异常
```
public E takeLast() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        E x;
        //如果队列为空，则等待
        while ( (x = unlinkLast()) == null)
            notEmpty.await(); //在notEmpty 条件上阻塞等待
        return x;
    } finally {
        lock.unlock();
    }
}
```
2、pollLast()
取出队列末尾元素，如果队列为空，则返回null
```
public E pollLast() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return unlinkLast();
    } finally {
        lock.unlock();
    }
}
```

3、pollLast(long timeout, TimeUnit unit)
取出队尾元素，如果队列为空，则进行操作等待
```
public E pollLast(long timeout, TimeUnit unit)
    throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        E x;
        //如果队列为空，则进行超时等待
        while ( (x = unlinkLast()) == null) {
            if (nanos <= 0) 
                return null;//发生超时，返回null
            nanos = notEmpty.awaitNanos(nanos); // 在notEmpty 条件上等待nanos纳秒
        }
        return x;
    } finally {
        lock.unlock();
    }
}
```

### 序列化

队列头尾指针都是被transient关键字修饰，因此必须自己手动来实现序列化工作。
```
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // Write out capacity and any hidden stuff
        s.defaultWriteObject(); //对其它可以序列化的属性进行序列化
        // Write out all elements in the proper order.
        //遍历队列，序列化元素
        for (Node<E> p = first; p != null; p = p.next)
            s.writeObject(p.item);
        // Use trailing null as sentinel
        s.writeObject(null); //队列结束标记
    } finally {
        lock.unlock();
    }
}
```
反序列化就是上面的逆过程，这个也很简单，这里就不展示了，可以自己去看看readObject 方法。
### 总结
LinkedBlockingDeque 个人觉得很简单，没有前面LinkedTransferQueue,SynchronousQueue有意思，因此本文都只是简要的贴了一下代码，基本上一看就能懂，没有过多解释。
LinkedBlockingDeque 内部使用了可重入锁（线程安全），不像SynchronousQueue使用的循环cas，因此很简单，出队和入队使用的是同一个锁，但是两头都可以操作队列，相对于单端队列可以减少一半的竞争。
LinkedBlockingDeque 同其他阻塞队列一样，不能存储null值元素。

LinkedBlockingDeque 和LinkedBlockingQueue 实际上是差不多的，因此可以结合着来看

[Java 并发 --- 阻塞队列之LinkedBlockingQueue源码分析](http://blog.ztgreat.cn/article/19)