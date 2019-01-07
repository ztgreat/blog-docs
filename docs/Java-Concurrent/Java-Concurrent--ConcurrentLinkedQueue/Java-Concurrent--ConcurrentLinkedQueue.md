在并发编程中，有时候需要使用线程安全的队列，如果要实现一个线程安全的队列有两种方式：一种是使用阻塞算法，另一种是使用非阻塞算法，在前面我们逐一分析过阻塞队列，这篇文章过后，会写篇关于阻塞队列的总结，也算是回顾知识，非阻塞的实现方式则可以使用循环cas的方式来实现，对于循环cas的算法，都已经遇到多了，在阻塞队列中，也有使用循环cas的队列,比如:SynchronousQueue，在jdk 1.8 中的ConcurrentHashMap则也是通过循环cas 实现的并发操作，今天我们一起来研究一下一个非阻塞的线程安全队列--ConcurrentLinkedQueue
### ConcurrentLinkedQueue 介绍（jdk 1.8）
ConcurrentLinkedQueue 是一个基于链表的无界线程安全队列，它采用的是先进先出的规则，当我们添加一个元素的时候，它会添加到队列的尾部；当我们获取一个元素时，它会返回队列头部的元素，它采用了CAS 算法来实现。
###  继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171217153858397.png)

ConcurrentLinkedQueue 是非阻塞队列，因此相比阻塞队列来说，并没有实现BlockingQueue 接口。

### 数据结构
ConcurrentLinkedQueue  的结构很简单，和普通队列差不多。

```
private static class Node<E> {
    volatile E item;  // 数据域
    volatile Node<E> next; // 后继指针

    /**
     * Constructs a new node.  Uses relaxed write because item can
     * only be seen after publication via casNext.
     */
    Node(E item) {
        // 通过unsafe 直接操控item
        UNSAFE.putObject(this, itemOffset, item);
    }
    ...// 省略部分方法
}
```
Node 是队列的存储单元，其属性被volatile  修饰，保证在多线程下的可见性，其构造方法，直接通过unsafe操控主存的item。

```
/**
 * A node from which the first live (non-deleted) node (if any)
 * can be reached in O(1) time.
 * Invariants:
 * - all live nodes are reachable from head via succ()
 * - head != null
 * - (tmp = head).next != tmp || tmp != head
 * Non-invariants:
 * - head.item may or may not be null.
 * - it is permitted for tail to lag behind head, that is, for tail
 *   to not be reachable from head!
 */
// 队列头指针
private transient volatile Node<E> head;
/**
 * A node from which the last node on list (that is, the unique
 * node with node.next == null) can be reached in O(1) time.
 * Invariants:
 * - the last node is always reachable from tail via succ()
 * - tail != null
 * Non-invariants:
 * - tail.item may or may not be null.
 * - it is permitted for tail to lag behind head, that is, for tail
 *   to not be reachable from head!
 * - tail.next may or may not be self-pointing to tail.
 */
// 队列尾指针
private transient volatile Node<E> tail;
```
### 构造方法
1、默认构造
```
public ConcurrentLinkedQueue() {
     head = tail = new Node<E>(null);
 }
```
初始化队列是，head和tail 并不是为null，而是都指向一个item域为null的节点，也就是说是一个带头结点的链表，这样可以避免很多复杂的判断（尝试写带头结点和不带头结点的链表就明白了）
2、通过集合初始化
```
public ConcurrentLinkedQueue(Collection<? extends E> c) {
    Node<E> h = null, t = null;
    for (E e : c) {
        checkNotNull(e);
        Node<E> newNode = new Node<E>(e);
        if (h == null)
            h = t = newNode;
        else {
            t.lazySetNext(newNode); // 设置t的next域
            t = newNode;
        }
    }
    if (h == null) // 空队列会指向一个item为null的节点
        h = t = new Node<E>(null);
    head = h;
    tail = t;
}
```
通过集合初始化队列，会遍历集合，然后依次加入到队列中（构建链表）。

### ConcurrentLinkedQueue 特性
前面 队列 head和tail有一大堆英文注释，这个就是ConcurrentLinkedQueue 的特性，理解这个可以更好的理解ConcurrentLinkedQueue 的算法思想。

在ConcurrentLinkedQueue **head 并不一定代表真正的head**,**tail 不一定代表真正的tail**,但是可以通过head 遍历到真正的head，通过tail可以遍历到真正的tail.

同阻塞队列一样，ConcurrentLinkedQueue **入队的元素不能是空，当删除元素时，需要先设置元素节点Node的item域为null**
#### head的不变性和可变性：

##### 不变性

 - 所有有效的节点都可以通过head节点遍历到，通过succ()方法
 - head不能为null
 - head节点的next不能指向自身

##### 可变性

 - head的item可能为null，也可能不为null
 - 允许tail滞后head，也就是说调用succc()方法，从head不可达tail

#### tail的不变性和可变性

##### 不变性

 - 队列的最后节点总是可以通过tail利用succ() 方法
 - tail不能为null

##### 可变性

 - tail的item可能为null，也可能不为null
 - tail节点的next域可以指向自身
 - 允许tail滞后head，也就是说调用succc()方法，从head不可达tail

是不是看起来很抽象，这个很正常，因此此时并不知道为什么要这样，这个需要从代码中来反推这些特性，先展示这些特性的原因是 想有一个大概了解，不至于后面理解得很混乱，在看代码 之前，先来看一些ConcurrentLinkedQueue 操作的图解，明白这些图解，也就明白这些特性了，当然后面代码也就很easy了，不过这里我想多说一句：如果想提高自己代码分析能力，个人建议不要看图解，看了特性后，直接去分析代码，根据代码中的逻辑来反推这些特性，想想什么情况下会出现这些情况，多思考一会，总会理解的，这样对自己的能力提高是有很大帮助的，当然如果只是想简单理解ConcurrentLinkedQueue，那么也可以直接看图解即可。

###  图解ConcurrentLinkedQueue
####  初始化
初始化head和tail 都指向一个item为null的节点

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171219093250219.png)

#### 入队
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171219093419748.png)

当元素a 入队后，其tail 指针并没有改变（未更新tail），此时tail 并非真的是最后一个节点。

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171219093610511.png)

入队元素b之后，将会更新tail，此时tail指向的是真正的尾节点，

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171219093753697.png)

同样在入队元素c之后，未更新tail,减少对tail的更新，可以降低冲突，但是tail 又不能离真正的尾节点太远，因为寻找尾节点需要通过tail来进行遍历。
#### 出队
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171219094521674.png)

出队操作从head 开始遍历，当发现head的item为null,则表明head 非真正的head,继续向后遍历，找到第一个数据节点，然后将其item设置为空，同时更新head,返回数据。

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171219094555525.png)

当再次出队的时候，head本身就是第一数据节点，直接返回数据，不更新head.

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171219094707199.png)

当再次出队时，发现出队元素c之后 没有后继节点了，那么更新head指向最后一个出队的元素，此时队列已经空了，这个时候，我们发现tail 仍然没改变，并且**tail 滞后于head**

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20171219094940847.png)

当再次入队时，会发现tail 失效了，会将tail 更新指向head,然后就可以成功的添加元素d.

后面的过程就很类似了，通过上面的图解，我相信应该能明白ConcurrentLinkedQueue 是如何操作的吧，当然可能图中细节的地方，仍然存在迷惑，这个可以从后面的代码再来细细品味。

### 入队
入队方法始终是返回true,不能根据其返回值是否成功，ConcurrentLinkedQueue是无界非阻塞队列，理应也不会执行失败，只有执行成功快慢之分而已。
```
public boolean add(E e) {
    return offer(e); //调用offer
}
```
```
public boolean offer(E e) {
    checkNotNull(e);//入队元素不能为null
    //创建节点
    final Node<E> newNode = new Node<E>(e);
    //从tail 开始找最后一个节点（tail的不变性）
    for (Node<E> t = tail, p = t;;) {
        // tail 后继
        Node<E> q = p.next;
        //表明 tail 就是最后一个节点
        if (q == null) {
            //添加新节点到链表 
            if (p.casNext(null, newNode)) {
                //检查是否需要更新tail(如果原tail不是最后一个节点则更新)
                if (p != t) // hop two nodes at a time
                    //更新tail ,失败了无所谓（tail的不变性）
                    casTail(t, newNode);  // Failure is OK.
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        //p已经被移除队列了，这个对应图解可以明白
        else if (p == q)
            /*
	         * t如果不是tail，则更新t，否则说明tail 指向失效（也就是                         
	         * tail滞后于head了），将p指向head
	         */
            p = (t != (t = tail)) ? t : head;
        else
            //tail 不是最后一个节点，更新p,向后遍历
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```
整个入队代码并不长，但是要完全理解，恐怕还是需要先理解ConcurrentLinkedQueue的特性，当从tail向最后一个节点进行遍历的时候，如果tail就是最后一个节点，那么可以直接将节点添加到链表末尾，如果添加失败，则说明其它线程抢先了一步，无所谓，重新循环寻找最后一个节点即可，如果添加成功了，那么可能就需要更新一下tail,频繁更新tail肯定会导致竞争加剧，因此ConcurrentLinkedQueue 采用了不用每次都更新tail（也就导致了tail并非是最后一个节点的情况），当tail 离最后一个节点有一段距离后，那么才更新tail,这种情况下，可以减少tail的更新次数，但是也增加了遍历最后一个节点的次数，因此进行权衡利弊，在适当的时候进行更新tail.
当tail 还没来得及更新就被其它线程移除队列时，就会发生p.next=p,这种情况，这个是在更新head的时候发生的，这个可以后面结合代码来看，当然其实在前面的图解中也展示出这个情况的，当出现这种情况时，也就需要更新指针的指向，如果tail 被其它线程修改，那么执行tail，如果tail被移除队列了，并没有更新，那么就指向head即可，因此这里也就又体现了tail可能滞后于head的特性。

### 出队

```
public E poll() {
    restartFromHead:
    for (;;) {//无限循环
        // 从head开始遍历
        for (Node<E> h = head, p = h, q;;) {
            //取出数据
            E item = p.item;
            //如果head是真正的第一节点，那么设置head item为null
            if (item != null && p.casItem(item, null)) {
                //同tail一样，适当的时候更新head
                if (p != h) // hop two nodes at a time
                    updateHead(h, ((q = p.next) != null) ? q : p);
                // 返回数据    
                return item;
            }
            //如果head 不是第一个真正节点或者cas 失败，则说明head 被其它线程更新了，尝试更新head
            else if ((q = p.next) == null) {
                // 队列为空了
               
                //更新head
                updateHead(h, p);
                //返回null
                return null;
            }
            // 其它线程更新了head，导致的
            else if (p == q)
               //重新开始
                continue restartFromHead;
            else
                // head 不是真正的第一个节点，向后遍历
                p = q;
        }
    }
}
```
出队代码也很短，其整体逻辑和入队差不多，同样是从head开始遍历，如果head 就是真正的节点，那么取数据，然后看看是否更新head,频繁更新head也会导致冲突加剧，因此并没有每一次出队都要更新head,这个在图解中也给予了展示，如果head被其它线程所更新，那么当前线程的head节点边失效了，会导致p.next=p,因此需要重新获取head，再进行遍历。
在入队的时候和出队的时候，我们都说道p.next=p这种情况，这个究竟在什么地方发生的呢，下面我们看看updateHead 就明白了。
```
final void updateHead(Node<E> h, Node<E> p) {
    if (h != p && casHead(h, p))
        h.lazySetNext(h); // 设置h.next=h
}
```
看到updateHead 这下明白了吧，更新head的时候，会将原head的next指向其本身，因此当本线程在操作的时候，如果其他线程抢先一步更新了head,就会导致本线程中的head失效了。

### size 计算
对于队列，一般很少会用size，同时在多线程下，size的值也并不可靠，但是这里为了更好的理解ConcurrentLinkedQueue，还是一起来看看吧。
```
public int size() {
    int count = 0;
    for (Node<E> p = first(); p != null; p = succ(p))
        if (p.item != null)
            // Collection.size() spec says to max out
            //最大size 为Integer.MAX_VALUE
            if (++count == Integer.MAX_VALUE)
                break;
    return count;
}
```
既然是链表，当然统计size 也就是遍历链表了，只是这里遍历链表的时候有点特殊，并不是直接遍历就行了，其奥妙就在于first()和succ()

```
// 寻找第一个真正的节点
/**
 * Returns the number of elements in this queue.  If this queue
 * contains more than {@code Integer.MAX_VALUE} elements, returns
 * {@code Integer.MAX_VALUE}.
 *
 * <p>Beware that, unlike in most collections, this method is
 * <em>NOT</em> a constant-time operation. Because of the
 * asynchronous nature of these queues, determining the current
 * number of elements requires an O(n) traversal.
 * Additionally, if elements are added or removed during execution
 * of this method, the returned result may be inaccurate.  Thus,
 * this method is typically not very useful in concurrent
 * applications.
 *
 * @return the number of elements in this queue
 */
Node<E> first() {
    restartFromHead:
    for (;;) { 
        // 从head 开始
        for (Node<E> h = head, p = h, q;;) {
            boolean hasItem = (p.item != null);
            if (hasItem || (q = p.next) == null) {
            　　//更新head指向真正的数据节点
                updateHead(h, p);
                
                return hasItem ? p : null;
            }
            // head失效，重新循环获取
            else if (p == q)
                continue restartFromHead;
            else
                // head不是真正有效的节点，向后遍历
                p = q;
        }
    }
}
```
first 其它和前面的出队很相识，但是first 方法用于寻找真正的第一个有效的节点，同时注意的时：**在寻找后，会同时更新head**

```
// 寻找当前节点的后继 
/**
 * Returns the successor of p, or the head node if p.next has been
 * linked to self, which will only be true if traversing with a
 * stale pointer that is now off the list.
 */
final Node<E> succ(Node<E> p) {
    Node<E> next = p.next;
    //如果p节点失效，那么返回head
    return (p == next) ? head : next;
}
```
在succ 寻找后继的时候，节点有可能失效了(也就是说被其它线程出队了)，那么这个时候需要返回head,重新循环遍历。

看到这里我们会发现size的值其实并不可靠，当然了在多线程中也不应该依赖该值，如果在统计过程中，有其它线程迅速的将元素进行出队，那么可能会重新从head开始循环，但是开始统计的值并没有进行清除掉，而是接着统计，因此size值并不可靠，正如jdk 注释的那样：not very useful in concurrent  applications.

### 获取队头元素
通过peek 方法可以获取到队头元素，而不会将元素出队。

```
public E peek() {
    restartFromHead:
    for (;;) {
        // 从head 开始寻找真正的第一元素
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;
            if (item != null || (q = p.next) == null) {
                //如果head不是第一有效的数据，会更新head
                updateHead(h, p);
                return item;
            }
            // 失效，重新开始
            else if (p == q)
                continue restartFromHead;
            else
                // 向后遍历有效的节点
                p = q;
        }
    }
}
```
有了上面的基础，我相信这个peek 也应该很容易就搞定了，值得注意的一点:调用peek的时候，如果head 不是真正的第一个元素节点，那么会更新head。


### 总结
在看完了ConcurrentLinkedQueue 后，其实我们发现它并不复杂，相比前面分析的一些其它循环cas算法来说，简直要简单许多，但是它确实又是一个非阻塞的高效的队列，在传统的队列设计中，队头总是队头(无头结点的队列)，队尾也总是队尾，但是在ConcurrentLinkedQueue 缺不是这样，在多线程下，队列头和队列尾的操作是非常频繁的，如果频繁的更新head和tail那么势必会导致冲突加剧，因此为了减小冲突，并不需要每次都更新队头和队尾，只有当队头和队尾离真正的队头和队尾有一段距离后，才会真正更新head和tail,也正是因此这样的原因，而产生了ConcurrentLinkedQueue的特性（head,tail的不变性与可变性）
