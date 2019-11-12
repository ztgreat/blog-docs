在前面我们分析了SynchronousQueue，SynchronousQueue这个阻塞队列很特殊，也很抽象，经过一番较量后，终于还是把它拿下，今天我们来看另一个也很抽象的阻塞队列，和SynchronousQueue有很大的相似之处，个人感觉类似扩展的SynchronousQueue。因此在阅读本文之前，最好对SynchronousQueue有比较好的认识，**如果你还没有学习过SynchronousQueue，那么个人不太建议你看这篇博客**，在SynchronousQueue 中我用了较多的文字来进行描述，因此可能要好理解一些，本文就不会再次重复的用过多文字来描述了，如果你在没有了解过其大致思想（执行流程）情况下，直接来看这篇的源码分析，个人觉得可能不太好理解。
可以参考一下我前面写的SynchronousQueue，如果感觉写得不好，还请麻烦指出:

[Java 并发 --- 阻塞队列之SynchronousQueue源码分析](http://localhost:8000/article/23)

## LinkedTransferQueue 介绍（jdk 1.8）

LinkedTransferQueue是基于链表的FIFO**无界**阻塞队列，它是JDK1.7才添加的阻塞队列.
在SynchronousQueue 中，如果进行的操作在队列中没有匹配的操作，那么就会阻塞，知道匹配的操作到来，在LinkedTransferQueue 也有类似的方法，同时更加的丰富，这个我们后面会慢慢展开来讨论。

## 继承体系

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171113210032046.png)

对比前面的阻塞队列，会发现LinkedTransferQueue 的继承体系有特殊之处。前面的阻塞队列都直接实现的BlockingQueue接口，在LinkedTransferQueue 却多了一个TransferQueue 接口，而该接口继承至BlockingQueue。
BlockingQueue 接口代表的是普通的阻塞队列，TransferQueue 则代表的是另一种特殊阻塞队列，它是指这样的一个队列：**当生产者向队列添加元素但队列已满时，生产者会被阻塞；当消费者从队列移除元素但队列为空时，消费者会被阻塞**。

前面我们分析的SynchronousQueue  不就是有这种特性吗，但是SynchronousQueue 并没有实现TransferQueue 接口，原因就在于TransferQueue 接口也是在jdk 1.7才出现的，应该是为了和前面的阻塞队列进行区分，同时为了后面扩充这种特殊的阻塞队列，才加入了TransferQueue ，这样功能才不至于混乱（单一职能原则）。

### TransferQueue 接口

```
public interface TransferQueue<E> extends BlockingQueue<E> {
    //立即转交一个元素给消费者，如果没有等待的消费者，则返回false(元素不入队)
    boolean tryTransfer(E e);
    
    //转交一个元素给消费者，如果没有等待的消费者，则阻塞直到消费者到来，或者发生异常

    //转交一个元素给消费者，如果没有等待的消费者，则阻塞直到超时
    boolean tryTransfer(E e, long timeout, TimeUnit unit)throws InterruptedException;
    
    //是否存在等待的消费者
    boolean hasWaitingConsumer();
    
    //返回等待的消费者的个数
    int getWaitingConsumerCount();
}

```
在SynchronousQueue  中也有类似的方法，当然没有这么多，只是以内部类的形式存在，而在LinkedTransferQueue 则把这种阻塞操作抽成了接口。

## 数据结构
首先来看看队列的节点定义
```
static final class Node {
     final boolean isData;   // 指示的是item 是否为数据
     volatile Object item;   // 数据域
     volatile Node next;     //后继指针
     volatile Thread waiter; // 等待线程
     final boolean casNext(Node cmp, Node val) {
         return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
     }
     final boolean casItem(Object cmp, Object val) {
         // assert cmp == null || cmp.getClass() != Node.class;
         return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
     }
     Node(Object item, boolean isData) {
         UNSAFE.putObject(this, itemOffset, item); // relaxed write
         this.isData = isData;
     }
     ...
}
```
这个和前面的SynchronousQueue 差别不大，这里就不细说了。(Node 中还有很多方法，这里没有列出，等后面需要的时候再展示)

```
// 判断是否为多核
private static final boolean MP =Runtime.getRuntime().availableProcessors() > 1;

// 自旋次数
private static final int FRONT_SPINS   = 1 << 7;

// 前驱节点正在处理，当前节点需要自旋的次数
private static final int CHAINED_SPINS = FRONT_SPINS >>> 1;


static final int SWEEP_THRESHOLD = 32;

// 头节点
transient volatile Node head;

// 尾节点
private transient volatile Node tail;


private transient volatile int sweepVotes;

/*
 * 调用xfer()方法时需要传入,区分不同处理，这个我们后面再分析
 * xfer()方法是LinkedTransferQueue的最核心的方法
 */
private static final int NOW   = 0; // for untimed poll, tryTransfer
private static final int ASYNC = 1; // for offer, put, add
private static final int SYNC  = 2; // for transfer, take
private static final int TIMED = 3; // for timed poll, tryTransfer
```

## 入队操作
LinkedTransferQueue提供了add、put、offer三类方法，用于将元素放到队列中。
**注意**我们这里所说的入队操作是指add,put,offer这几个方法，而不是指真正的把节点入队的操作，因为LinkedTransferQueue 中针对的不是数据，而是操作，操作可能需要入队，而这个操作可能是放数据操作，也可能是取数据操作，这里注意区分一下，不要搞混了。
```
public void put(E e) {
    xfer(e, true, ASYNC, 0);
}

public boolean offer(E e, long timeout, TimeUnit unit) {
    xfer(e, true, ASYNC, 0);
    return true;
}

public boolean offer(E e) {
    xfer(e, true, ASYNC, 0);
    return true;
}

public boolean add(E e) {
    xfer(e, true, ASYNC, 0);
    return true;
}
```

可以看到，这几个入队方法，都调用的是同一个方法xfer，LinkedTransferQueue 是一个由链表组成的无界队列，因此不会有容量限制（一定范围内），因此这里入队的操作都不会阻塞（因此超时入队方法实际也没有用），也就是说，入队后线程会立即返回，这个是参数ASYNC的作用。
### xfer 方法
在看 xfer 方法之前呢，我还是先叙述一下大致流程，因为这样可能会帮助你理解。
LinkedTransferQueue和SynchronousQueue 是一样的，队列中主要的不是针对数据，而且操作（put或take,注意这里put，take 指的是放入数据和取数据），队列中既可以存储入队操作，也可以存储出队操作，当队列为空时，如果有线程进行出队操作，那么这个时候队列是没有数据的，那么这个操作就会被入队，同时线程也会阻塞，直到数据的到来（或出现异常），如果最开始队列为空，放入数据的操作到来，那么数据就会被放到队列中，此后如果取数据操作到来，那么就会从队列中取出数据，因此可以知道队列中存放的都是一系列相同的操作（put(放数据操作)或take(取数据操作)）。

这里我们说的是**放数据操作**，那么如果队列为空，那么直接将数据入队即可，同时因为是无界队列，线程不会阻塞，直接返回，如果队列不为空，那么队列里面可能有两种情况：

（1）存放的都是数据

（2）存放的都是取数据操作

如果是情况1：那么本次操作的和队列中的节点的操作是一样的，因此直接把数据放到队列末尾，线程返回。
如果是情况2：那么本次操作和队列中的节点的操作是不一样的（也就是匹配的,放入数据操作和取数据操作是匹配的，也就是不同的操作是匹配的，相同的操作是不匹配的），那么就把队头的节点出队，把本次的数据给队头节点，同时唤醒该节点的线程。

```
 /**
  * Implements all queuing methods. See above for explanation.
  *
  * @param e the item or null for take
  * @param haveData true if this is a put, else a take
  * @param how NOW, ASYNC, SYNC, or TIMED
  * @param nanos timeout in nanosecs, used only if mode is TIMED
  * @return an item if matched, else e
  * @throws NullPointerException if haveData mode but e is null
  */
 private E xfer(E e, boolean haveData, int how, long nanos) {
     if (haveData && (e == null))
         throw new NullPointerException();
     Node s = null;                        // the node to append, if needed
     retry:
     for (;;) {                            // restart on append race
         // 遍历队列，看看有没有匹配的操作
         for (Node h = head, p = h; p != null;) { // find & match first node
             boolean isData = p.isData;
             Object item = p.item;
             //正常的操作节点，item等于本身则表示该操作被取消了或者匹配过了
             if (item != p && (item != null) == isData) { 
                 // 该节点的操作和本次操作不匹配，那么整个队列都将不匹配，break出来
                 if (isData == haveData)   
                     break;
                 // 该节点的操作和本次操作是匹配的，设置节点item域为本次操作数据域e
                 if (p.casItem(item, e)) { 
                     /**
                     *队列中的操作都是一样的，和节点p匹配成功，但是p不是head，那么p之前的节点都失效了（被匹配过了）
                     *但是还没有将这些失效节点移除队列，因此这里会帮忙做这个工作
                     */
                     for (Node q = p; q != h;) {
                         Node n = q.next;  // update by 2 unless singleton
                         //设置新head
                         if (head == h && casHead(h, n == null ? q : n)) {
                             h.forgetNext(); //head 成环，移除队列，方便gc
                             break;
                         } 
                         /**
                         *如果head 为null，则跳出来
                         *如果只有head，那么也跳出来（开始进入这个循环时，p在head 后面）
                         *如果q已经被匹配过了，跳出来
                         */                
                         if ((h = head)   == null ||
                             (q = h.next) == null || !q.isMatched())
                             break;        // unless slack < 2
                     }
                     //唤醒操作节点p的线程
                     LockSupport.unpark(p.waiter);
                     
                     /**
                      *返回操作节点p的item(如果操作节点p 是take 那么item 就是null
                      *如果是put，那么本次就是take,返回的就是数据)
                     */
                     return LinkedTransferQueue.<E>cast(item);
                 }
             }
             // 节点p 不正常了（该操作被取消了）
             Node n = p.next;
             //如果p 成环了，那么重新从head 开始遍历（成环了表示节点p已经完成了匹配了）
             p = (p != n) ? n : (h = head); // Use head if p offlist
         }
         //本次操作和队列中的节点操作是一直的，那么就将该操作入队，如果how 是NOW 则直接返回，不入队
         if (how != NOW) {                 // No matches available
             if (s == null)
                 s = new Node(e, haveData); //生成节点
             //将节点入队，同时返回其全驱    
             Node pred = tryAppend(s, haveData);
             if (pred == null)
                 continue retry;           // lost race vs opposite mode
             //如果how 是ASYNC 那么就不阻塞，直接返回
             if (how != ASYNC)
                 //否则进行自旋然后阻塞（如果设置了超时，则进行超时等待）
                 return awaitMatch(s, pred, e, (how == TIMED), nanos);
         }
         return e; // not waiting
     }
 }
```
通过注释和上面的描述看一下大致流程，不着急于细节，应该还是很容易明白吧。

**对于这里的操作**（**放入数据**），寻找匹配节点,如果找到了，就设置item值,然后unpark匹配节点的waiter线程,返回（其实就是看看队列里面的操作是不是取数据操作），否则就入队（NOW直接返回）：
如果没有找到匹配节点，则根据传入的how来处理，NOW直接返回，其余三种先入队，入队后如果是ASYNC则返回，SYNC和TIMED则会阻塞等待匹配。
下面我们来看看里面的部分方法：

```
/**
 * Returns true if this node has been matched, including the
 * case of artificial matches due to cancellation.
 */
final boolean isMatched() {
    Object x = item;
    return (x == this) || ((x == null) == isData);
}
```

如果操作节点已经被匹配了，那么item会被改变，对于取数据操作，那么item会被设置成数据，如果操作被取消了，那么会设置item为this，这里isMatched 包含了这种情况（注释已经说明白了）
```
/**
 * Links node to itself to avoid garbage retention.  Called
 * only after CASing head field, so uses relaxed write.
 */
final void forgetNext() {
    UNSAFE.putObject(this, nextOffset, this);
}
```
forgetNext 设置next 为自身，也就是脱离链表，同时方便gc回收自己。
接下来我们在看看入队调用的tryAppend方法：

```
/**
 * Tries to append node s as tail.
 *
 * @param s the node to append
 * @param haveData true if appending in data mode
 * @return null on failure due to losing race with append in
 * different mode, else s's predecessor, or s itself if no
 * predecessor
 */
private Node tryAppend(Node s, boolean haveData) {
    for (Node t = tail, p = t;;) {        // move p to last node and append
        Node n, u;                        // temps for reads of next & tail
        //队列为空，则设置head，注意这里并没有设置tail也指向head哟
        if (p == null && (p = head) == null) {
            if (casHead(null, s))
                return s;                 // initialize
        }
        else if (p.cannotPrecede(haveData))//如果不合符入队要求，则不入队
            return null;                  // lost race vs opposite mode
        /**
         *p 本应该是队尾（p.next==null），可能其它线程操作了队列
         *导致了p线程不是队尾了，因此需要更新p
         */
        else if ((n = p.next) != null)    // not last; keep traversing
            //如果tail 被更新了，p也不等于t,那么也更新p,t
            //如果p==t 或者 t==taile,如果p没有被取消（p.next==p）,则遍历，否则设置p为null
            p = p != t && t != (u = tail) ? (t = u) : // stale tail
                    (p != n) ? n : null;      // restart if off list
        else if (!p.casNext(null, s))     //将s 入队
            //入队失败，那么p又不是队尾了，往后遍历
            p = p.next;                   // re-read on CAS failure
        else {
            //入队成功后，更新tail,更新失败则往后遍历，直到成功
            if (p != t) {                 // update if slack now >= 2
                while ((tail != t || !casTail(t, s)) &&
                        (t = tail)   != null &&
                        (s = t.next) != null && // advance and retry
                        (s = s.next) != null && s != t);
            }
            return p; // 返回前驱p
        }
    }
}
```

这个把操作入队（把节点链接到链表末尾）是不是也有点复杂，主要原因还是没有使用锁，存在很多并发情况下，有可能自己在添加节点入队的时候，其它线程已经把队列改变了，那么这个时候就需要重新找到队列尾，进行添加添加操作，添加成功后，也需要设置队尾指针，这个时候队尾指针可能也被其它线程设置了，那么这个时候自己也要保证队尾指针是正确的（遍历验证）。
在上面我们还看到有一个这个方法：p.cannotPrecede(haveData)，如果数据不符合要求，那么是不会入队的。
```
/**
 * Returns true if a node with the given mode cannot be
 * appended to this node because this node is unmatched and
 * has opposite data mode.
 */
final boolean cannotPrecede(boolean haveData) {
    boolean d = isData;
    Object x;
    return d != haveData && (x = item) != this && (x != null) == d;
}
```
这个就是验证操作和其数据节点的数据是否是吻合的。
到这里应该差不多都明白了吧，awaitMatch 这个我们在后面的出队来分析，因为放数据的过程是不会阻塞的，当然也更不会执行该方法。

## 出队操作
LinkedTransferQueue提供了poll、take方法用于出列元素
```
public E take() throws InterruptedException {
    //这里的参数又ASYNC 变成了SYNC
    E e = xfer(null, false, SYNC, 0);
    if (e != null)
        return e;
    Thread.interrupted();
    throw new InterruptedException();
}

public E poll() {
    //参数为NOW，如果取不到数据，该操作不会入队阻塞等待，而是直接返回
    return xfer(null, false, NOW, 0);
}

public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    //具有超时等待的操作
    E e = xfer(null, false, TIMED, unit.toNanos(timeout));
    if (e != null || !Thread.interrupted())
        return e;
    throw new InterruptedException();
}
```

这里**再次强调**，这里的出队操作，指的是poll、take方法，而不是真正指的是出队操作，因为poll、take操作也可能会入队（队列针对的是操作，不是数据）。

通过上面一系列的方法，我们看到，其实poll、take内部调用的仍然是xfer 方法，因为是取数据，因此参数部分发生了变化，这个注意一下。

```
/**
 * Implements all queuing methods. See above for explanation.
 *
 * @param e the item or null for take
 * @param haveData true if this is a put, else a take
 * @param how NOW, ASYNC, SYNC, or TIMED
 * @param nanos timeout in nanosecs, used only if mode is TIMED
 * @return an item if matched, else e
 * @throws NullPointerException if haveData mode but e is null
 */
private E xfer(E e, boolean haveData, int how, long nanos) {
    if (haveData && (e == null))
        throw new NullPointerException();
    Node s = null;                        // the node to append, if needed
    retry:
    for (;;) {                            // restart on append race
        //这里是取数据操作，那么遍历队列看看有没有匹配的操作（即放数据操作）
        for (Node h = head, p = h; p != null;) { // find & match first node
            boolean isData = p.isData;
            Object item = p.item;
            if (item != p && (item != null) == isData) { // unmatched
                if (isData == haveData)   // can't match
                    break;
                /**
                 *队列里面确实都是放数据的操作，则和当前操作是匹配的
                 *设置匹配操作节点的item域为null (e为null，原本item 域是数据)
                 */
                if (p.casItem(item, e)) { // match
                    //协助推进head,这个和上面是一样的
                    for (Node q = p; q != h;) {
                        Node n = q.next;  // update by 2 unless singleton
                        if (head == h && casHead(h, n == null ? q : n)) {
                            h.forgetNext();
                            break;
                        }                 // advance and retry
                        if ((h = head)   == null ||
                                (q = h.next) == null || !q.isMatched())
                            break;        // unless slack < 2
                    }
                    //唤醒阻塞线程（实际这里p.waiter是为null的，因为放数据操作是非阻塞的）
                    LockSupport.unpark(p.waiter);
                    // item线程是数据，本次操作是取数据操作，因此返回数据
                    return LinkedTransferQueue.<E>cast(item);
                }
            }
            Node n = p.next;
            p = (p != n) ? n : (h = head); // Use head if p offlist
        }
        //如果参数指定为NOW，那么就算没有被匹配，那么还是不入队，直接返回
        if (how != NOW) {                 // No matches available
            if (s == null)
                s = new Node(e, haveData);
            //添加节点    
            Node pred = tryAppend(s, haveData);
            if (pred == null)
                continue retry;           // lost race vs opposite mode
            /**
             *如果参数不是ASYC的这种，这可能需要阻塞等待
             *取数据操作其参数都不是ASYNC，因此如果没有取到数据（被匹配），那么就可能进行阻塞等待
             */
            if (how != ASYNC)
                return awaitMatch(s, pred, e, (how == TIMED), nanos);
        }
        return e; // not waiting
    }
}
```

当在理解上面代码和注释的时候，记住这里我们分析的是取数据部分，因为有了前面放数据部分的分析，这里应该还是很好理解，取数据和放数据都是差不多的，都是和队列里面的操作进行匹配，如果队里里面的操作是取数据操作，本次操作是取数据操作，那么此时是不匹配的，需要把本次操作入队（参数NOW，ASYNC,SYNC,TIMED 不一样），如果队列的操作都是放数据的操作，本次操作是取数据操作，那么这个是匹配的，就把队头的数据取出来，返回即可。

下面我们来看看awaitMatch 方法：

```
/**
 * Spins/yields/blocks until node s is matched or caller gives up.
 *
 * @param s the waiting node
 * @param pred the predecessor of s, or s itself if it has no
 * predecessor, or null if unknown (the null case does not occur
 * in any current calls but may in possible future extensions)
 * @param e the comparison value for checking match
 * @param timed if true, wait only until timeout elapses
 * @param nanos timeout in nanosecs, used only if timed is true
 * @return matched item, or e if unmatched on interrupt or timeout
 */
private E awaitMatch(Node s, Node pred, E e, boolean timed, long nanos) {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    int spins = -1; // initialized after first item and cancel checks
    ThreadLocalRandom randomYields = null; // bound if needed

    for (;;) {
        Object item = s.item;
        //原本item是等于e的，匹配过后或者取消后，会改变item
        if (item != e) {                  // matched  
            //将item 设置成自身，waiter 设置为null
            s.forgetContents();           // avoid garbage
            return LinkedTransferQueue.<E>cast(item);
        }
        //如果被中断，或者发生超时了，那么就取消该操作（设置item 为自身）
        if ((w.isInterrupted() || (timed && nanos <= 0)) &&
                s.casItem(e, s)) {        // cancel
            //从队列中移除该节点
            unsplice(pred, s);
            return e;
        }
        //下面都是进行的超时或者自旋操作
        if (spins < 0) {                  // establish spins at/near front
            if ((spins = spinsFor(pred, s.isData)) > 0)
                randomYields = ThreadLocalRandom.current();
        }
        else if (spins > 0) {             // spin
            --spins;
            if (randomYields.nextInt(CHAINED_SPINS) == 0)
                Thread.yield();           // occasionally yield
        }
        else if (s.waiter == null) {
            s.waiter = w;                 // request unpark then recheck
        }
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos > 0L)
                //如果设置了超时，则进行超时等待
                LockSupport.parkNanos(this, nanos);
        }
        else {
            //阻塞等待
            LockSupport.park(this);
        }
    }
}
```
```
final void forgetContents() {
    UNSAFE.putObject(this, itemOffset, this);
    UNSAFE.putObject(this, waiterOffset, null);
}
```
看起来稍微有点长，其实不算难，和SynchronousQueue的awaitFulfill差不多，这里主要进行了自旋，如果自旋后，仍然没有被匹配或者取消，则进行阻塞（如果设置了超时阻塞，则进行一段时间的阻塞），如果发生了中断异常，会取消该操作，该边item的值，匹配成功后也会更改item的值，因此如果item和原来的值不相等时，则说明发生了改变，返回即可。

在awaitMatch过程中，如果线程被中断了，或者超时了则会调用unsplice()方法去除该节点。

```
/**
 * Unsplices (now or later) the given deleted/cancelled node with
 * the given predecessor.
 *
 * @param pred a node that was at one time known to be the
 * predecessor of s, or null or s itself if s is/was at head
 * @param s the node to be unspliced
 */
final void unsplice(Node pred, Node s) {
    //清除s的部分数据
    s.forgetContents(); // forget unneeded fields
    /*
     * See above for rationale. Briefly: if pred still points to
     * s, try to unlink s.  If s cannot be unlinked, because it is
     * trailing node or pred might be unlinked, and neither pred
     * nor s are head or offlist, add to sweepVotes, and if enough
     * votes have accumulated, sweep.
     */
    if (pred != null && pred != s && pred.next == s) {
        Node n = s.next;
        if (n == null ||
            (n != s && pred.casNext(s, n) && pred.isMatched())) {
            /**
            *这个for循环，用于推进head,如果head已经被匹配了，则需要更新head
            */
            for (;;) {               // check if at, or could be, head
                Node h = head;
                if (h == pred || h == s || h == null)
                    return;          // at head or list empty
                 //h 没有被匹配，跳出循环，否则可能需要更新head  
                if (!h.isMatched())
                    break;
                Node hn = h.next;
                //遍历结束了，退出循环
                if (hn == null)
                    return;          // now empty
                //head 被匹配了，重新设置设置head    
                if (hn != h && casHead(h, hn))
                    h.forgetNext();  // advance head
            }
            //s节点被移除后，需要记录删除的操作次数，如果超过阀值，则需要清理队列
            if (pred.next != pred && s.next != s) { // recheck if offlist
                for (;;) {           // sweep now if enough votes
                    int v = sweepVotes;
                    //没超过阀值，则递增记录值
                    if (v < SWEEP_THRESHOLD) {
                        if (casSweepVotes(v, v + 1))
                            break;
                    }
                    else if (casSweepVotes(v, 0)) {
                        //重新设置记录数，并清理队列
                        sweep();
                        break;
                    }
                }
            }
        }
    }
}
```
```
/**
 * Unlinks matched (typically cancelled) nodes encountered in a
 * traversal from head.
 */
private void sweep() {
    for (Node p = head, s, n; p != null && (s = p.next) != null; ) {
        if (!s.isMatched()) // s节点未被匹配，则继续向后遍历
            // Unmatched nodes are never self-linked
            p = s;
        else if ((n = s.next) == null) //s节点被匹配，但是是尾节点，则退出循环
            //s为尾结点，则可能其它线程刚好匹配完，所有这里不移除s，让其它匹配线程操作
            break;
        else if (s == n)    // stale s节点已经脱离了队列了，重头开始遍历
            // No need to also check for p == s, since that implies s == n
            p = head;
        else
            p.casNext(s, n); //移除s节点
    }
}
```
看看这个移除操作其实也不是那么简单，这里并没有简单的就将节点移除就ok了，同时还检查了队列head的有效性，如果head被匹配了，则会推荐head,保持队列head 是有效的，
如果移除节点的前驱也失效了，说明其它线程再操作，这里就不操作了，当移除了节点后，需要记录移除节点的操作次数sweepVotes，如果这个值超过了阀值，则会对队列进行清理（移除那些失效的节点）

## 获取队列首个有效操作
在SynchronousQueue 中其队列是无法遍历的，而且也无法获取队头信息，但是在LinkedTransferQueue却不一样，LinkedTransferQueue可以获取对头，也可以进行遍历
#### peek 方法
```
public E peek() {
    return firstDataItem();
}
```
```
/**
 * Returns the item in the first unmatched node with isData; or
 * null if none.  Used by peek.
 */
private E firstDataItem() {
    //遍历队列，查找第一个有效的操作节点
    for (Node p = head; p != null; p = succ(p)) {
        Object item = p.item;
        //如果该节点是数据节点，同时没有被取消，则返回数据
        if (p.isData) {
            if (item != null && item != p)
                return LinkedTransferQueue.<E>cast(item);
        }
        else if (item == null)// 非数据节点返回null，这里注意
            return null;
    }
    return null;
}
```
```
/**
 * Returns the successor of p, or the head node if p.next has been
 * linked to self, which will only be true if traversing with a
 * stale pointer that is now off the list.
 */
final Node succ(Node p) {  //如果节点p 失效则返回head,否则返回p的后继
    Node next = p.next;
    return (p == next) ? head : next;
}
```
这个peek 方法返回的是队列的第一个有效的节点，而这个节点可能是数据节点，也可能是取数据的操作节点，那么peek 可能返回数据，也可能返回null,但是返回null 并不一定是队列为空，也可能是队列里面都是取数据的操作节点，这个需要**注意**一下。


## 总结
LinkedTransferQueue 总算是分析完了，相比SynchronousQueue ，废话应该要少很多，LinkedTransferQueue 和SynchronousQueue 其实基本是差不多的，两者都是无锁带阻塞功能的队列，SynchronousQueue 通过内部类Transferer 来实现公平和非公平队列
在LinkedTransferQueue 中没有公平与非公平的区分，LinkedTransferQueue 实现了TransferQueue接口，该接口定义的是带阻塞操作的操作，相比SynchronousQueue  中的Transferer 功能更丰富。
本文分析LinkedTransferQueue是基于在SynchronousQueue 基础上的，如果你知道SynchronousQueue 那么看LinkedTransferQueue应该也很容易明白，如果直接点看LinkedTransferQueue 估计可能有点懵逼，所以个人建议先看看SynchronousQueue 。

[Java 并发 --- 阻塞队列之SynchronousQueue源码分析](http://localhost:8000/article/23)

SynchronousQueue  中放数据操作和取数据操作都是阻塞的，当队列中的操作和本次操作不匹配时，线程会阻塞，直到匹配的操作到来。
LinkedTransferQueue  是无界队列，放数据操作不会阻塞，取数据操作如果没有匹配操作可能会阻塞，通过参数决定是否阻塞（ASYNC,SYNC,NOW,TIMED）。


