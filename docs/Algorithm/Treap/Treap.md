##  前言

这篇博客是以前LZ 学习算法的时候，学习记录的，已经过去很远了，直接从csdn搬过来的，文中的代码是用c 语言写的，因为是学习记录因此内容和代码可能都缺乏严谨，因此仅供参考，后面有时间争取review一遍，LZ 觉得一些重要的算法思想还是很重要的，你要说我现在对平衡树还有多大感觉，LZ只想说忘完了，学习算法或者数据结构的重点在于锻炼思维，而不是去死记硬背一种代码，LZ觉得当初的算法经历对个人的编码能力还是有很大影响的。

##  介绍



前面我们介绍了AVL树，伸展树，它们都是二叉搜索树，二叉搜索树的主要问题就是其结构与数据相关，树的深度可能会很大，Treap树就是一种解决二叉搜索树可能深度过大的另一种数据结构。

Treap=Tree+Heap。Treap本身是一棵二叉搜索树，它的左子树和右子树也分别是一个Treap，和一般的二叉搜索树不同的是，

Treap纪录一个额外的数据，就是优先级。Treap在以关键码构成二叉搜索树的同时，还满足堆的性质。这些优先级是是在结点插入时，随机赋予的，Treap根据这些优先级满足堆的性质。这样的话，Treap是有一个随机附加域满足堆的性质的二叉搜索树，其结构相当于以随机数据插入的二叉搜索树。

。其基本操作的期望时间复杂度为O(logn)。相对于其他的平衡二叉搜索树，Treap的特点是实现简单，且能基本实现随机平衡的结构。

Treap维护堆性质的方法只用到了旋转，只需要两种旋转，编程复杂度比Splay要小一些。

##  数据结构

这里我们还是先给出treap的结构定义:

```
typedef struct TreapNode* Tree;
typedef int ElementType;
struct TreapNode
{
    ElementType val; //结点值
    int priority;   //优先级
    Tree lchild;
    Tree rchild;
    TreapNode(int val=0,int priority=0) //默认构造函数
    {
        lchild=rchild=NULL;
        this->val=val;
        this->priority=priority;
    }
};

```

可以看到这个和前面的AVL树以及伸展树基本都是一样的，这里去掉了父节点了，因为父节点的存在只是为了方便我们的操作，这里将不用到父节点统一的进行各项操作。

treap的搜索和一般的BST树是一致的，这里我们就跳过吧，直接说插入操作。

和二叉搜索树的插入一样，先把要插入的点插入到一个叶子上，然后跟维护堆一样，如果当前节点的优先级比它父节点优先级小就旋转，如果当前节点是其父节点的左儿子就右旋，如果当前节点是其父节点的右儿子就左旋。（简单点说把优先级小的往上提）

**注：这里我们都调整为最小堆的形式。**

##  旋转操作

如果知道AVL树或者伸展树的旋转方式，那么treap的旋转就显得很简单了，这里我们还是用图来展示一下过程：

###  （1）左旋转



![20151103100828878](http://img.blog.ztgreat.cn/document/algorithm/20151103100828878.png)



黑色的数字是优先级，当插入数值为7优先级为20的结点后，违反了最小堆的定义，所以这里需要进行左旋转，将结点7进行提升一个层次，当然这个左旋转方式有很多种，如果把结点7看做操作对象，那么就需要知道它的父节点以及祖父结点，但是我们在定义中又去掉了父节点指针，这样在获取父节点的是否似乎变得困难了，不妨换个角度，如果我们结点7的父节点为对象，那么只需要知道结点3的父节点就可以操作了，这样的话我们可以通过引用传参数，就可以不用父节点指针，直接进行旋转操作，直接看代码吧

```
//左旋转
void left_rotate(Tree &node)
{
    Tree temp=node->rchild;
    node->rchild=temp->lchild;
    temp->lchild=node;
    node=temp;
}

```



参数node为需要调整结点的父节点，在图示中也就是结点3，对比图看就明白了，注意这里是指针的引用。

###  （2）右旋转

![20151103112406527](http://img.blog.ztgreat.cn/document/algorithm/20151103112406527.png)



同左旋转本质是一样的。

```
//右旋转
void right_rotate(Tree &node)
{
    Tree temp=node->lchild;
    node->lchild=temp->rchild;
    temp->rchild=node;
    node=temp;
}

```

##  插入操作

先来看看图解：



![20151103185313376](http://img.blog.ztgreat.cn/document/algorithm/20151103185313376.png)



插入值为18，优先级为20的结点后，违反了最小堆的定义，因此要进行调整，把优先级小的往上提，也就是小的优先级插入的是右子树，那么需要进行左旋转，这里进行一次旋转过后就OK了。



![20151103185547874](http://img.blog.ztgreat.cn/document/algorithm/20151103185547874.png)

同样，这种情况左旋转，旋转后发现还是不满足最小堆的定义，并且小优先级的结点在左子树，所以还需要进行右旋转，如下图所示:



![20151103185759470](http://img.blog.ztgreat.cn/document/algorithm/20151103185759470.png)

右旋后，很遗憾还是不行，还需要左旋：



![20151103190032649](http://img.blog.ztgreat.cn/document/algorithm/20151103190032649.png)

OK，终于完成，插入一个数据，也许要进行多次旋转，不过也仅仅是左旋或者右旋而已，相对AVL树来说，简单很多了。

下面来看看插入的代码

```
//插入函数
bool insert(Tree &root,ElementType val=0,int priority=0)
{
    Tree node=new TreapNode(val,priority);
    return insert_val(root, node);
}
//内部插入函数
//参数:根结点，需插入的结点
bool insert_val(Tree &root,Tree &node)
{
    if (!root)
    {
        root=node; //插入
        return true;
    }
    else if(root->val>node->val)
    {
       bool flag=insert_val(root->lchild, node);
        if (root->priority>node->priority) //检查是否需要调整
            right_rotate(root);
        return flag;
    }
    else if(root->val<node->val)
    {
        bool flag=insert_val(root->rchild, node);
        if (root->priority>node->priority) //检查是否需要调整
            left_rotate(root);
        return flag;
    }
    //已经含有该元素，释放结点
    delete node;
    return false;
 
}

```

对比图解，应该很容易就明白啦。

##  删除

###  （1）用二叉搜索树的方式删除

在前面的AVL树以及伸展树的删除结点中，我们都是采用的[BST树中删除结点](http://blog.csdn.net/u014634338/article/details/39102739)的办法，也就是对于叶子结点或者只有一个孩子的结点的删除方式是直接删除，否则找替代元素，化繁为简。但是如果treap也是采用这样的方式我们看可不可以呢，首先我们肯定能正确的删除结点，但是我们删除后我们可能需要进行调整，因为也许优先级不在满足要求，可能需要向下调整，也可能需要向下调整，这样呢，比较麻烦，这里我们讲用堆的方式删除。

###  （2）用堆的方式删除

因为Treap满足堆性质，所以只需要把要删除的节点旋转到叶节点上，然后直接删除就可以了，**具体的方法：**如果该节点的左子节点的优先级小于右子节点的优先级，右旋该节点,使该节点降为右子树的根节点，然后访问右子树的根节点,继续操作；反之，左旋该节点,使该节点降为左子树的根节点，然后访问左子树的根节点，继续操作，直到变成可以直接删除的节点。（即：让小优先级的结点旋到上面，满足最小堆的性质）。

下面图解展示：

![20151103191638120](http://img.blog.ztgreat.cn/document/algorithm/20151103191638120.png)



删除结点11的右孩子的优先级更小，因此需要左旋，旋转后如图所示，这个时候，结点11还未到最简单的情况，需要再次旋转:

![20151103191823126](http://img.blog.ztgreat.cn/document/algorithm/20151103191823126.png)



这次是其左孩子优先级较低，需要右旋。当然旋转后，还是还要继续的。。。



![20151103191955835](http://img.blog.ztgreat.cn/document/algorithm/20151103191955835.png)



到这里，我们喜欢的情况出现了，需要删除的结点11只有左孩子，这样我们重接左孩子就可以了。



![20151103192113723](http://img.blog.ztgreat.cn/document/algorithm/20151103192113723.png)



对于删除操作应该是明白了吧，下面看代码：

```
//删除函数
bool remove(Tree &root,ElementType val)
{
    if (root==NULL)
        return false;
    else if (root->val>val)
        return remove(root->lchild,val);
    else if(root->val<val)
        return remove(root->rchild,val);
    else //找到执行删除处理
    {
 
        Tree *node=&root;
        while ((*node)->lchild && (*node)->rchild)  //从该结点开始往下调整
        {
            if ((*node)->lchild->priority<(*node)->rchild->priority) //比较其左右孩子优先级
            {
                right_rotate(*node); //右旋转
                node=&((*node)->rchild); //更新传入参数，进入下一层
            }
            else
            {
                left_rotate(*node); //左旋转
                node=&((*node)->lchild); //更新传入参数，进入下一层
            }
        }
        
        //最后调整到（或者本来是）叶子结点，或者只有一个孩子的情况，可以直接删除了
        if ((*node)->lchild==NULL)
            (*node)=(*node)->rchild;
        else if((*node)->rchild==NULL)
            (*node)=(*node)->lchild;
        return true;
    }
}

```



OK，treap到这里就讲完了，treap的编程复杂度比起AVL,和伸展树来说要小很多，这是只是简单的对treap进行了实现，对于treap的很多应用还需要学习学习，扩展思路。

```
#include <stdio.h>
#include <stdlib.h>
#include <iostream>
using namespace std;
typedef struct TreapNode* Tree;
typedef int ElementType;
struct TreapNode
{
    ElementType val; //结点值
    int priority;   //优先级
    Tree lchild;
    Tree rchild;
    TreapNode(int val=0,int priority=0) //默认构造函数
    {
        lchild=rchild=NULL;
        this->val=val;
        this->priority=priority;
    }
};
Tree search(Tree &,ElementType);
void right_rotate(Tree &);
void left_rotate(Tree &);
bool insert_val(Tree &,Tree &);
bool insert(Tree &,ElementType,int);
bool remove(Tree &,ElementType);
 
 
//查找函数
Tree search(Tree &root,ElementType val)
{
    if (!root)
        return NULL;
    else if (root->val>val)
       return search(root->lchild,val);
    else if(root->val<val)
      return  search(root->rchild,val);
    return root;
}
//插入函数
bool insert(Tree &root,ElementType val=0,int priority=0)
{
    Tree node=new TreapNode(val,priority);
    return insert_val(root, node);
}
//内部插入函数
//参数:根结点，需插入的结点
bool insert_val(Tree &root,Tree &node)
{
    if (!root)
    {
        root=node; //插入
        return true;
    }
    else if(root->val>node->val)
    {
       bool flag=insert_val(root->lchild, node);
        if (root->priority>node->priority) //检查是否需要调整
            right_rotate(root);
        return flag;
    }
    else if(root->val<node->val)
    {
        bool flag=insert_val(root->rchild, node);
        if (root->priority>node->priority) //检查是否需要调整
            left_rotate(root);
        return flag;
    }
    //已经含有该元素，释放结点
    delete node;
    return false;
 
}
//右旋转
void right_rotate(Tree &node)
{
    Tree temp=node->lchild;
    node->lchild=temp->rchild;
    temp->rchild=node;
    node=temp;
}
//左旋转
void left_rotate(Tree &node)
{
    Tree temp=node->rchild;
    node->rchild=temp->lchild;
    temp->lchild=node;
    node=temp;
}
 
//删除函数
bool remove(Tree &root,ElementType val)
{
    if (root==NULL)
        return false;
    else if (root->val>val)
        return remove(root->lchild,val);
    else if(root->val<val)
        return remove(root->rchild,val);
    else //找到执行删除处理
    {
 
        Tree *node=&root;
        while ((*node)->lchild && (*node)->rchild)  //从该结点开始往下调整
        {
            if ((*node)->lchild->priority<(*node)->rchild->priority) //比较其左右孩子优先级
            {
                right_rotate(*node); //右旋转
                node=&((*node)->rchild); //更新传入参数，进入下一层
            }
            else
            {
                left_rotate(*node); //左旋转
                node=&((*node)->lchild); //更新传入参数，进入下一层
            }
        }
        
        //最后调整到（或者本来是）叶子结点，或者只有一个孩子的情况，可以直接删除了
        if ((*node)->lchild==NULL)
            (*node)=(*node)->rchild;
        else if((*node)->rchild==NULL)
            (*node)=(*node)->lchild;
        return true;
    }
}
 
 
//前序
void PreOrder(Tree root)
{
    if (root==NULL)
        return;
    printf("%d ",root->val);
    PreOrder(root->lchild);
    PreOrder(root->rchild);
}
//中序
void InOrder(Tree root)
{
    if (root==NULL)
        return;
    InOrder(root->lchild);
    printf("%d ",root->val);
    InOrder(root->rchild);
}
int main()
{
    Tree root=NULL;
 
    insert(root,11,6);
    insert(root,7,13);
    insert(root,14,14);
    insert(root,3,18);
    insert(root,9,22);
    insert(root,18,20);
    insert(root,16,26);
    insert(root,15,30);
    insert(root,17,12);
    printf("插入:\n");
    printf("前序:");
    PreOrder(root);
    printf("\n");
    printf("中序:");
    InOrder(root);
    printf("\n");
    printf("删除:\n");
    remove(root,11);
    printf("前序:");
    PreOrder(root);
    printf("\n");
    printf("中序:");
    InOrder(root);
    printf("\n");
    return 0;
}

```