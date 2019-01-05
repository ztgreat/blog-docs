在前面我们分析了HashMap和LinkedHashMap，在jdk 1.8 中 HashMap是数组+链表+红黑树实现的,当hash表某个位置上的元素个数超过某个阀值过后就采用红黑树结构，否则采用链表结构，在HashMap中元素 相对插入顺序是无序的，也就是说其遍历顺序不可预测。
LinkedHashMap是基于HashMap实现的，使用了双向链表，保证按照插入顺序或者访问顺序进行迭代.

今天我们再来看看TreeMap，从名称我们大致知道，这是一个树形结构的map。

### TreeMap 介绍（jdk 1.8）
TreeMap 是一种基于红黑树实现的 Key-Value 结构,如果我们想要自定义一种元素之间的顺序，那么前两种map都是无法实现的，而我们今天要讲的TreeMap恰好满足我们的需求，TreeMap 支持按键值进行升序访问，或者由传入的比较器（Comparator）来控制。
####  基础知识
如果你不清楚了搜索二叉树的相关操作，那么还是有必要先回顾一下，这里有一篇我很久以前写的关于搜索二叉树的操作，这个里面是c语言代码，而且时间写的时间比较早，因此质量可能就那么一般了，因此仅供参考：
####  [二叉搜索树](http://blog.csdn.net/u014634338/article/details/39102739)
如果对红黑树从未曾了解过，那么还是先去看看红黑树吧，前面写过一篇红黑树的博客，可以参考一下。
####  [亲自动手画红黑树](http://blog.ztgreat.cn/article/12)

### 继承体系
![这里写图片描述](http://img.blog.ztgreat.cn/document/collection/20171118221058405.png)

TreeMap 继承AbstractMap， AbstractMap 提供了 Map 的基本实现。 
TreeMap 实现了Cloneable接口，支持clone()方法，可以被克隆。 
TreeMap 实现了Serializable接口，可以被序列化。
除此之外，我们还看到TreeMap 实现了NavigableMap 接口，而NavigableMap接口继承至SortedMap。

SortedMap 是一个扩展自 Map 的一个接口，对该接口的实现要保证所有的 Key 是完全有序的（也就是说可以比较的）。
这个顺序一般是指 Key 的自然序（实现 Comparable 接口）或在创建 SortedMap 时指定一个比较器（Comparator）。当我们元素迭代时，就可以按序访问其中的元素。
```
Comparator<? super K> comparator();
```
NavigableMap 是 JDK 1.6 之后新增的接口，扩展了 SortedMap 接口，提供了一些导航方法。
例如：
```
/**
 *Returns a key-value mapping associated with the greatest 
 *key strictly less than the given key 
 */
Map.Entry<K,V> lowerEntry(K key);//小于给定 Key 的 最大的Entry
/**
 *Returns the greatest key strictly less than the given key
 */
K lowerKey(K key);
/**
 *Returns a key-value mapping associated with the greatest key
 *less than or equal to the given key
 */
Map.Entry<K,V> floorEntry(K key);

/**
 *Returns a reverse order view of the mappings contained in this map
 */
NavigableMap<K,V> descendingMap();// 返回一个逆序的map
...
```
NavigableMap 可以按照 Key 的升序或降序进行访问和遍历。 descendingMap() 和 descendingKeySet() 则会获取和原来的顺序相反的集合，集合中的元素则是同样的引用，在该视图上的修改会影响到原始的数据。
### 数据结构

```
  //比较器，没有指定的话默认使用Key的自然序
  private final Comparator<? super K> comparator;

  //红黑树根节点
  private transient Entry<K,V> root;

  //树中节点的数量
  private transient int size = 0;

  //存储结构被 修改的次数
  private transient int modCount = 0;

  static final class Entry<K,V> implements Map.Entry<K,V> {
    K key; //键
    V value; //值
    Entry<K,V> left;  //左孩子
    Entry<K,V> right; //右孩子
    Entry<K,V> parent; //父节点
    boolean color = BLACK; //节点颜色
    省略方法
 }
```
TreeMap 是基于红黑树来实现的，排序时按照键的自然序（要求实现 Comparable 接口）或者提供一个 Comparator 用于排序。

### 构造方法
1、默认构造
```
public TreeMap() {
    comparator = null;
}
```
比较器为null,那么会使用key的比较器,也就意味着key必须实现Comparable 接口,否则在比较的时候就会出现异常，这个我们后面会看到。
2、指定比较器

```
public TreeMap(Comparator<? super K> comparator) {
    this.comparator = comparator;
}
```
使用自定义的比较器，来比较key之间的大小关系。
3、通过非SortedMap集合构造

```
public TreeMap(Map<? extends K, ? extends V> m) {
    comparator = null;
    putAll(m);
}
```
将map集合中的元素添加到TreeMap中，比较器为null.
4、通过SortedMap集合构造

```
public TreeMap(SortedMap<K, ? extends V> m) {
    comparator = m.comparator();
    try {
        buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
    } catch (java.io.IOException cannotHappen) {
    } catch (ClassNotFoundException cannotHappen) {
    }
}
```
通过SortedMap 集合构造，比较器是参数集合的比较器。
### 查询元素
红黑树的添加和删除都是稍微比较复杂的，因此我们先来看看TreeMap的查询方法。
#### get 方法
```
public V get(Object key) {
    //通过getEntry 获得节点
    Entry<K,V> p = getEntry(key);
    return (p==null ? null : p.value);
}
```
```
final Entry<K,V> getEntry(Object key) {
    //如果比较器不为null,则使用比较器获取节点
    if (comparator != null)
        return getEntryUsingComparator(key);
    //如果比较器为null,同时key为NULL，则抛出异常    
    if (key == null)
        throw new NullPointerException();
    //获取key的比较器，因此如果key没有实现Comparable接口，则会发生异常  
    Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K,V> p = root;
    //从根节点开始遍历二叉树
    while (p != null) {
        int cmp = k.compareTo(p.key);
        //在左子树寻找
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0) //在右子树寻找
            p = p.right;
        else
            return p; //相等，返回节点
    }
    return null;  //否则 没找到，返回null
}
```

```
// 通过指定的比较器进行比较，实际上和上面是差不多的
final Entry<K,V> getEntryUsingComparator(Object key) {
    
    K k = (K) key;
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = cpr.compare(k, p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
                return p;
        }
    }
    return null;
}
```
get 方法很简单，其实就是一颗搜索二叉树的遍历过程，如果查找的节点大于"根"节点，则在其"根"节点的右子树查找，如果小于"根"节点，则在其左子树查找，否则就是相当，返回"根"节点即可。
当指定了比较器时，使用指定的比较器来查找节点，否则通过key的比较器来查找，因此如果在没有指定比较器的情况下，同时key没有实现Comparable接口，则会抛出异常。
### 添加元素
接下来我们来看看TreeMap的添加方法，实际上就是红黑树的插入过程，因此这里需要红黑是知识。

```
public V put(K key, V value) {
    Entry<K,V> t = root;
    //如果根节点为null,则直接赋值给根节点
    if (t == null) {
        compare(key, key); 
        //默认是黑色节点。
        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    Entry<K,V> parent;
    Comparator<? super K> cpr = comparator;
    // 如果比较器不为空，就是用指定的比较器来维护TreeMap的元素顺序
    if (cpr != null) {
         // 查找key要插入的位置
        do {
            parent = t;
            // 比较当前节点的key和新插入的key的大小
            cmp = cpr.compare(key, t.key);
            // 新插入的key小的话，则以当前节点的左孩子节点为新的比较节点
            if (cmp < 0) 
                t = t.left;
            // 新插入的key大的话，则以当前节点的右孩子节点为新的比较节点    
            else if (cmp > 0) 
                t = t.right;
           // 如果当前节点的key和新插入的key相等的话，则覆盖map的value，返回旧值     
            else
                return t.setValue(value);
        } while (t != null);
    }
    else {
        /**
         *如果比较器为空，则使用key作为比较器进行比较
	     *这里要求key不能为空，并且必须实现Comparable接口
	     */
        if (key == null)
            throw new NullPointerException();
        
        Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    //生成节点，指定其父节点为 parent
    Entry<K,V> e = new Entry<>(key, value, parent);
    // 如果新节点key的值小于父节点key的值，则插在父节点的左侧
    if (cmp < 0)
        parent.left = e;
    // 如果新节点key的值大于父节点key的值，则插在父节点的右侧    
    else
        parent.right = e;
    // 插入新的节点后，为了保持红黑树平衡，对红黑树进行调整
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```
除去调整红黑树的部分，其余的都很简单，就是搜索二叉树的插入过程，这个应该都比较熟悉涩（数据结构中有学的哟），这里面的插入节点，使用的是循环的方式，如果不考虑性能，还可以用递归的方式，用递归的方式简洁明了，当插入节点后，可能破坏了红黑树的结构，因此需要进行红黑树的平衡调整操作fixAfterInsertion，这部分就有点复杂了，因此如果光看代码肯定意义是不大的，必须结合图解来分析，因为我在前面已经分析过了红黑树的部分，这里就不继续贴图了，我把地址贴出来，可以参考着一起看，记住一定要结合图解来看，否则看不懂就不是我的锅了。
#### [亲自动手画红黑树](http://blog.ztgreat.cn/article/12)
结合里面插入节点的图解来理解下面的代码。
```
private void fixAfterInsertion(Entry<K,V> x) {
    //设置插入节点的颜色为红色
    x.color = RED;
    // while循环，保证新插入节点x不是根节点或者新插入节点x的父节点不是红色（这两种情况不需要调整）
    while (x != null && x != root && x.parent.color == RED) {
        // 如果新插入节点x的父节点是祖父节点的左孩子
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            // 取得新插入节点x的叔叔节点
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            // 如果叔叔节点是红色
            if (colorOf(y) == RED) {
                // 将x的父节点设置为黑色
                setColor(parentOf(x), BLACK);
                // 将x的叔叔节点设置为黑色
                setColor(y, BLACK);
                // 将x的祖父节点设置为红色
                setColor(parentOf(parentOf(x)), RED);
                // 将x指向祖父节点，按照上面的步奏继续循环（向上回溯修复）
                x = parentOf(parentOf(x));
            } else {
                // 如果新插入x的叔叔节点是黑色或缺少，且x是父节点的右孩子
                if (x == rightOf(parentOf(x))) {
                    //x指向其父节点 （左旋父节点）
                    x = parentOf(x);
                    rotateLeft(x);
                }
                
                // 将x的父节点设置为黑色                      
                setColor(parentOf(x), BLACK);
                // 将x的祖父节点设置为红色
                setColor(parentOf(parentOf(x)), RED);
                // 右旋x的祖父节点
                rotateRight(parentOf(parentOf(x)));
            }
        } else {
            // 如果新插入节点x的父节点是祖父节点的右孩子，下面其实和上面是差不多的，只是旋转方向发生变化
            // 取得新插入节点x的叔叔节点
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            // 如果叔叔节点是红色
            if (colorOf(y) == RED) {
                // 将x的父节点设置为黑色
                setColor(parentOf(x), BLACK);
                // 将x的叔叔节点设置为黑色
                setColor(y, BLACK);
                // 将x的祖父节点设置为红色
                setColor(parentOf(parentOf(x)), RED);
                // 将x指向祖父节点，按照上面的步奏继续循环（向上回溯修复）
                x = parentOf(parentOf(x));
            } else {
                // 如果新插入x的叔叔节点是黑色或缺少，且x是父节点的左孩子
                if (x == leftOf(parentOf(x))) {
                    //x指向其父节点 （右旋父节点）
                    x = parentOf(x);
                    rotateRight(x);
                }
                // 将x的父节点设置为黑色
                setColor(parentOf(x), BLACK);
                // 将x的祖父节点设置为红色
                setColor(parentOf(parentOf(x)), RED);
                // 左旋x的祖父节点
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    //设置根节点为黑色
    root.color = BLACK;
}
```
上面的代码看起来复杂，其实也还好，一定要结合我的图解来看，如果你看不懂本段代码，要么是你没看图解，要么是我的图解写的不好，如果图解不好欢迎指出，这里我就不在重复用图解来描述了。（**左旋右旋也请参考图集来理解**）
### 删除元素
删除操作首先需要做的也是搜索二叉树的删除操作，删除操作会删除对应的节点，如果是叶子节点就直接删除，如果是非叶子节点，会用对应的**中序遍历的后继节**点来顶替要删除节点的位置。删除后就需要做删除修复操作，使的树符合红黑树的定义。

第一步：将红黑树当作一颗二叉查找树，将节点删除。 

① 被删除节点为叶节点。那么，直接将该节点删除就OK了。 

② 被删除节点只有一个孩子。那么，用孩子节点的代替该节点即可。 

③ 被删除节点有两个孩子。那么，先找出它的后继节点；然后把“它的后继节点的内容”复制给“该节点的内容”；之后，删除“它的中序后继节点”，其中序后继节点不可能存在有两个孩子节点的情况，因此就转换成了①，②的情况了。

第二步：通过”旋转和重新着色”等一系列来修正该树，使之重新成为一棵红黑树。
#### remove 方法
```
public V remove(Object key) {
    //通过getEntry 得到节点
    Entry<K,V> p = getEntry(key);
    if (p == null)
        return null;
    V oldValue = p.value;
    //删除节点p
    deleteEntry(p);
    // 返回值
    return oldValue;
}
```
deleteEntry 是删除节点操作，这个就是在搜索二叉树的删除节点，具体删除方法在前面已经提到了，下面我们再来温习一下。
```
/**
 * Delete node p, and then rebalance the tree.
 */
private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;

    // If strictly internal, copy successor's element to p and then make p
    // point to successor.
    //如果存在左右孩子
    if (p.left != null && p.right != null) {
        //后继替代节点
        Entry<K,V> s = successor(p);
        p.key = s.key;
        //将替代节点的值，赋值给p
        p.value = s.value;
        // 转移成 删除s，s要么是叶子节点，要么只有一个孩子，不会出现两个孩子的情况
        p = s;
       } // p has 2 children

       // Start fixup at replacement node, if it exists.
       //得到其左孩子或者右孩子，如果没孩子则为null
       Entry<K,V> replacement = (p.left != null ? p.left : p.right);
       
       //replacement 为null,则现在的p是叶子节点
       if (replacement != null) {
           // Link replacement to parent
            // 将replacement的父节点设置为p的父节点
           replacement.parent = p.parent;
           //p的parent为null,则说明p为根节点
           if (p.parent == null)
               //将根设置为replacement，删除了节点p
               root = replacement;
           // 如果替代节点p是其父节点的左孩子，则将replacement设置为其父节点的左孩子
           else if (p == p.parent.left)
               p.parent.left  = replacement;
           // 如果替代节点p是其父节点的左孩子，则将replacement设置为其父节点的右孩子    
           else
               p.parent.right = replacement;

           // Null out links so they are OK to use by fixAfterDeletion.
            // 将替代节点p的left、right、parent的指针都指向空(从树中移除)，使得gc可以回收
           p.left = p.right = p.parent = null;

           // Fix replacement
           // 如果替代节点p的颜色是黑色，则需要调整红黑树以保持其平衡
           if (p.color == BLACK)
               fixAfterDeletion(replacement); 
       } else if (p.parent == null) { // return if we are the only node.
           root = null; // 只有一个节点，且删除的就是根节点，删除后root置null
       } else { //  No children. Use self as phantom replacement and unlink.
		   //替代节点p 是叶子节点，且不是根节点
		   
		   // 如果p的颜色是黑色，则调整红黑树（先调整，后删除，因为还需要利用其引用关系查找父节点或者兄弟节点）
           if (p.color == BLACK)
               fixAfterDeletion(p);
               
           // 下面是删除替代节点p
           if (p.parent != null) {
               // 重新设置p的父节点的孩子指针
               if (p == p.parent.left)
                   p.parent.left = null;
               else if (p == p.parent.right)
                   p.parent.right = null;
               // 解除p对p父节点的引用    
               p.parent = null;
           }
       }
   }
```
这个整体应该不难，除外调整方法，完全就是对搜索二叉树的删除操作，如果你已经不清楚了搜索二叉树的相关操作，那么还是有必要先回顾一下，这里有一篇我很久以前写的关于搜索二叉树的操作，这个里面是c语言代码，而且时间写的时间比较早，因此质量可能就那么一般了，因此仅供参考：

#### [二叉搜索树](http://blog.csdn.net/u014634338/article/details/39102739)

查找后继节点方法
```
/**
 * Returns the successor of the specified Entry, or null if no such.
 */
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
       if (t == null)
           return null;
       //如果右孩子不为null
       else if (t.right != null) {
           // 查找右孩子的最左分支节点
           Entry<K,V> p = t.right;
           while (p.left != null)
               p = p.left;
           return p;
       } else {
           //沿着向上（向跟节点方向）找到该分支上第一个左孩子的父节点或者null
           Entry<K,V> p = t.parent;
           Entry<K,V> ch = t;
           while (p != null && ch == p.right) {
               ch = p;
               p = p.parent;
           }
           return p;
       }
   }
```
successor(Entry t)返回指定节点的继承者（**中序遍历的后继**）。分三种情况处理，第一。t节点是个空节点：返回null；第二，t有右孩子：找到t的右孩子中的最左子孙节点，如果右孩子没有左孩子则返回右节点，否则返回找到的最左子孙节点；第三，t没有右孩子：沿着向上（向跟节点方向）找到该分支上第一个左孩子的父节点或者null
这个想一想中序遍历的过程就应该很容易明白了。

下面我们再来看看删除操作中红黑树的调整操作
####  [亲自动手画红黑树](http://blog.ztgreat.cn/article/12)
结合里面删除节点的图解来理解下面的代码。
```
private void fixAfterDeletion(Entry<K,V> x) {
    // while循环，保证要删除节点x不是跟节点，并且是黑色（根节点和红色不需要调整）
    while (x != root && colorOf(x) == BLACK) {
        // 如果要删除节点x是其父亲的左孩子
        if (x == leftOf(parentOf(x))) {
            // 取出要删除节点x的兄弟节点
            Entry<K,V> sib = rightOf(parentOf(x));
			   // 如果删除节点x的兄弟节点是红色
               if (colorOf(sib) == RED) {
                   // 将x的兄弟节点颜色设置为黑色
                   setColor(sib, BLACK);
                   // 将x的父节点颜色设置为红色
                   setColor(parentOf(x), RED);
                   // 左旋x的父节点
                   rotateLeft(parentOf(x));
                   // 将sib重新指向旋转后x的兄弟节点
                   sib = rightOf(parentOf(x));
               }
               // 如果x的兄弟节点的两个孩子都是黑色
               if (colorOf(leftOf(sib))  == BLACK &&
                   colorOf(rightOf(sib)) == BLACK) {
                   // 将兄弟节点的颜色设置为红色
                   setColor(sib, RED);
                   // 将x的父节点指向x，继续向上回溯调整
                   x = parentOf(x);
               } else {
                   // 如果x的兄弟节点右孩子是黑色，左孩子是红色
                   if (colorOf(rightOf(sib)) == BLACK) {
                       // 将x的兄弟节点的左孩子设置为黑色
                       setColor(leftOf(sib), BLACK);
                       // 将x的兄弟节点设置为红色
                       setColor(sib, RED);
                       // 右旋x的兄弟节点
                       rotateRight(sib);
                       // 将sib重新指向旋转后x的兄弟节点
                       sib = rightOf(parentOf(x));
                   }
                   // 设置 x的兄弟节点颜色为其父节点的颜色
                   setColor(sib, colorOf(parentOf(x)));
                   // 将x的父节点设置为黑色
                   setColor(parentOf(x), BLACK);
                   // 将x的兄弟节点的右孩子设置为黑色
                   setColor(rightOf(sib), BLACK);
                   // 左旋x的父节点
                   rotateLeft(parentOf(x));
                   // 达到平衡，将x指向root，退出循环
                   x = root;
               }
           } else { // 如果要删除节点x是其父亲的右孩子，和上面情况一样，只是左右旋转交换了一下
               // 取出要删除节点x的兄弟节点 
               Entry<K,V> sib = leftOf(parentOf(x));
               // 如果删除节点x的兄弟节点是红色
               if (colorOf(sib) == RED) {
                   setColor(sib, BLACK);
                   setColor(parentOf(x), RED);
                   rotateRight(parentOf(x));
                   sib = leftOf(parentOf(x));
               }
               // 如果x的兄弟节点的两个孩子都是黑色
               if (colorOf(rightOf(sib)) == BLACK &&
                   colorOf(leftOf(sib)) == BLACK) {
                   setColor(sib, RED);
                   x = parentOf(x);
               } else {
                   // 如果x的兄弟节点左孩子是黑色，右孩子是红色
                   if (colorOf(leftOf(sib)) == BLACK) {
                       setColor(rightOf(sib), BLACK);
                       setColor(sib, RED);
                       rotateLeft(sib);
                       sib = leftOf(parentOf(x));
                   }
                   setColor(sib, colorOf(parentOf(x)));
                   setColor(parentOf(x), BLACK);
                   setColor(leftOf(sib), BLACK);
                   rotateRight(parentOf(x));
                   x = root;
               }
           }
       }
       //设置根节点的颜色为黑色
       setColor(x, BLACK);
   }
```
删除相对来说更加复杂，一定要对照着图解看代码，否则基本上读不懂的，如果没看图解就说看不懂我的内容，那这个锅我不能背的哈。

### 序列化
TreeMap中很多关键的属性都被transient修饰，因此需要我们手动的进行序列化工作，这个和我们前面其它集合的序列化是差不多的，遍历TreeMap把节点的key和value依次写入即可。
```
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException {
    // Write out the Comparator and any hidden stuff
    s.defaultWriteObject();

    // Write out size (number of Mappings)
    s.writeInt(size);

    // Write out keys and values (alternating)
    for (Iterator<Map.Entry<K,V>> i = entrySet().iterator(); i.hasNext(); ) {
        Map.Entry<K,V> e = i.next();
        s.writeObject(e.getKey());
        s.writeObject(e.getValue());
    }
}
```
### 总结
TreeMap 是一种基于红黑树实现的 Key-Value 结构，TreeMap 支持按键值进行升序访问，或者由传入的比较器（Comparator）来控制。当指定了比较器时，使用指定的比较器来查找节点，否则通过key的比较器来查找，因此如果在没有指定比较器的情况下，同时key没有实现Comparable接口，则会抛出异常。
TreeMap 实现了NavigableMap 接口，而NavigableMap接口继承至SortedMap。
SortedMap 是一个扩展自 Map 的一个接口，对该接口的实现要保证所有的 Key 是完全有序的（也就是说可以比较的）。
这个顺序一般是指 Key 的自然序（实现 Comparable 接口）或在创建 SortedMap 时指定一个比较器（Comparator）。当我们元素迭代时，就可以按序访问其中的元素。NavigableMap 是 JDK 1.6 之后新增的接口，扩展了 SortedMap 接口，提供了一些导航方法，本文对TreeMap的分析比较简单，里面还有很多类似的方式，这个可以尝试自己去分析一下，对于迭代器部分，后面根据情况可能会加上。
TreeMap 是非线程安全的，在前面对数据结构的操作中都没有进行同步操作。
分析TreeMap主要是分析红黑树，因此需要对红黑树有一定的认识，这样才好理解TreeMap。

#### [二叉搜索树](http://blog.csdn.net/u014634338/article/details/39102739)
#### [亲自动手画红黑树](http://blog.ztgreat.cn/article/12)