

###  树的路径长度

树的路径长度是从树根到树中每一结点的路径长度之和。在结点数目相同的二叉树中，完全二叉树的路径长度最短。

### 树的带权路径长度

结点的权值：在一些应用中，赋予树中结点的一个有某种意义的实数、

结点的带权路径长度：结点到树根之间的路径长度与该结点上权的乘积

树的带权路径长度(`wpl`)：定义为树中所有结点的带权路径长度之和

![20150716175727929](http://img.blog.ztgreat.cn/document/algorithm/20150716175727929.jpg)



### 最优二叉树

在权为`w1,w2,...,wn`的n个叶子结点所构成的所有二叉树中，带权路径长度最小(即代价最小)的二叉树成为最优二叉树。
注意：

- 叶子上的权值均相同时，完全二叉树一定是最优二叉树，否则完全二叉树不一定是最优二叉树。
- 最优二叉树中，权值越大的叶子结点离根越近
- 最优二叉树的形态不唯一，`wpl`最小



### 构造最优二叉树

#### 哈夫曼算法

1. 根据给定的n个权值`w1,w2,...,wn`构成n棵二叉树的森林`F={T1,T2,..,Tn}`,其中每棵二叉树Ti中只有一个权值为`wi`的根节点，其左右子树均为空。
2. 在森林F中选出两棵根结点权值最小的树(当这种的树不止两棵时，可以从中任选两棵)，将这两棵树和并成一棵新树，为了保证新树仍是二叉树，需要增加一个新结点作为新树的根，并将所选的两棵树根分别作为新树的左右孩子，将这两个孩子的权值之和作为新树根的权值
3. 对新的森林F重复2,直到森林F只剩一棵树为止。

注意

- 初始森林中的n棵二叉树，每棵树都有一个孤立的结点，它们既是根，又是叶子
- n个叶子结点的哈夫曼树需要经过n-1次合并，产生n-1个新结点。最终求得的哈夫曼树共有2n-1个结点
- 哈夫曼树是严格的二叉树，没有度数为1的分支结点

#### 哈夫曼树存储结构（数组形式）

```
struct HuffNode
{
    int weight;
    int lchild;
    int rchild;
    int parent;
};
```

注意：

1. 因此数组下界为0,因此用-1表示空指针。树中某结点的lchild,rchild,parent不等于-1时，它们分别是该结点的左、右孩子和双亲结点在向量中的下标。
2. parent域的作用：其一是查找双亲结点更为简单。其二是可通过判定parent值是否为-1来区分根和非根结点。

#### 实现代码

（1）初始化，将`T[0..m-1]`中的2n-1个结点里的三个指针均置空（==-1）,权值置为0.

```
void initHuffmantree(struct HuffNode *tree, int len)
{
    for(int i = 0; i < len; i ++)
    {
        tree[i].weight = 0;
        tree[i].lchild = -1;
        tree[i].rchild = -1;
        tree[i].parent = -1;
    }
}
```

  哈夫曼树合并，对森林中的树共进行n-1次合并，所产生的新结点依次放入向量T的第i个分量中(n<=i<=m-1).每次合并分两步：

1. 在当前森林`T[0..i-1]`的所有结点中，选取权最小和次小的两个根节点`T[p1]`和`T[p2]`作为合并对象，这里`0<=p1,p2<=i-1`
2. 将根为`T[p1]`和`T[p2]`的两棵树作为左右子树合并为一棵新的树，新的树根是新节点`T[i]`.具体操作是：将`T[p1]`和`T[p2]`的parent置为i,将`T[i]`的lchild和rchild分别置为p1和p2.新结点`T[i]`的权值置为`T[p1]`和`T[p2]`的权值之和。
3. 合并后，`T[p1]`和`T[p2]`在当前森林中已不再是根，因为它们的双亲指针均已指向了`T[i]`,所以下次合并时不会被选中为合并对象

```
void createHuffmantree(struct HuffNode *tree, int n, int m)
{
    int Min1, Min2, k1, k2;
    for(int i = n; i < m; i ++)
    {
        //初始化最小值、次小值
        Min1 = Min2 = MIN_NUM;
        k1 = k2 = -1;
        //在尚未构造二叉树的节点中查找权值最小的两棵树
        for(int k = 0; k < i; k ++)
        {
            if(tree[k].parent == -1)
            {
                if(tree[k].weight <Min1)
                {
                    Min2 = Min1;
                    k2 = k1;
                    Min1 = tree[k].weight;
                    k1 = k;
                }
                else if(tree[k].weight <Min2)
                {
                    Min2 = tree[k].weight;
                    k2 = k;
                }
            }
        }
        
        //修改2个小权重节点的双亲
        tree[k1].parent = i;
        tree[k2].parent = i;
        
        //修改parent的权重
        tree[i].weight = tree[k1].weight + tree[k2].weight;
        
        //修改parent的孩子
        tree[i].lchild = k1;
        tree[i].rchild = k2;
    }
}
```



### 哈夫曼编码

#### 编码和解码

数据压缩的过程成为编码。即将文件中的每个字符均转换为一个唯一的二进制位串

数据解压过程成为解码。即将二进制位串转换位对应的字符

#### 前缀码&&最优前缀码

对字符集进行编码时，要求字符集中任一字符的编码都不是其它字符的编码的前缀，这种编码成为前缀编码

平均码长或文件总长最小的前缀编码成为最优前缀编码，最优前缀编码对文件的压缩效果亦最佳。

#### 哈夫曼编码为最优前缀编码

1. 每个叶子字符`ci`的码长恰为从根到该叶子的路径长度li,平均码长（或文件总长）又是二叉树的带权路径长度WPL.而哈夫曼树是WPL最小的二叉树，因此编码的平均码长亦最小
2. 树中没有一片叶子是另一个叶子的祖先，每片叶子对应的编码就不可能是其它叶子编码的前缀。

#### 哈夫曼编码算法

**算法思想**:

给定字符集的哈夫曼树生成后，求哈弗曼编码的具体实现过程是：依次以叶子`T[i](0 <= i< n)`为出发点，向上回溯到根为止，上溯是走左分支则生成代码0,走右分支则生成代码1.

```
//哈夫曼编码
struct HuffCode
{
    char code[MAX_SIZE];
    HuffCode()
    {
        memset(code, 0, sizeof(code));
    }
};
```

```
void createHuffmancode(struct HuffNode *htree, struct HuffCode *hcode, int n)
{
    //c和p分别指示T中的孩子和双亲的位置
    int p, i, c;
    //临时存放编码
    char cd[n];
    //指示编码在cd中起始位置
    int start;
    
    //依次求叶子T[i]的编码
    for(i = 0; i < n; i ++)
    {
        c = i;
        start = n;
        memset(cd, 0, sizeof(cd));
        p = htree[c].parent;
        while(p != -1)
        {
            cd[-- start] = (htree[p].lchild == c)? '0' : '1';
            c = p;
            p = htree[p].parent;
        }
        strcpy(hcode[i].code, cd + start);
    }
}
```

### 实战

#### (1)UVA 10954

![20150716180607439](http://img.blog.ztgreat.cn/document/algorithm/20150716180607439.png)

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#define MAX_SIZE 10010
#define MIN_NUM 1000000000
struct HuffNode
{
    int weight;
    int lchild;
    int rchild;
    int parent;
};
//哈夫曼编码
struct HuffCode
{
    char code[MAX_SIZE];
    HuffCode()
    {
        memset(code, 0, sizeof(code));
    }
};
 
void createHuffmancode(struct HuffNode *htree, struct HuffCode *hcode, int n)
{
    //c和p分别指示T中的孩子和双亲的位置
    int p, i, c;
    //临时存放编码
    char cd[n];
    //指示编码在cd中起始位置
    int start;
    
    //依次求叶子T[i]的编码
    for(i = 0; i < n; i ++)
    {
        c = i;
        start = n;
        memset(cd, 0, sizeof(cd));
        p = htree[c].parent;
        while(p != -1)
        {
            cd[-- start] = (htree[p].lchild == c)? '0' : '1';
            c = p;
            p = htree[p].parent;
        }
        strcpy(hcode[i].code, cd + start);
    }
}
 
void initHuffmantree(struct HuffNode *tree, int len)
{
    for(int i = 0; i < len; i ++)
    {
        tree[i].weight = 0;
        tree[i].lchild = -1;
        tree[i].rchild = -1;
        tree[i].parent = -1;
    }
}
void createHuffmantree(struct HuffNode *tree, int n, int m)
{
    int Min1, Min2, k1, k2;
    for(int i = n; i < m; i ++)
    {
        //初始化最小值、次小值
        Min1 = Min2 = MIN_NUM;
        k1 = k2 = -1;
        //在尚未构造二叉树的节点中查找权值最小的两棵树
        for(int k = 0; k < i; k ++)
        {
            if(tree[k].parent == -1)
            {
                if(tree[k].weight <Min1)
                {
                    Min2 = Min1;
                    k2 = k1;
                    Min1 = tree[k].weight;
                    k1 = k;
                }
                else if(tree[k].weight <Min2)
                {
                    Min2 = tree[k].weight;
                    k2 = k;
                }
            }
        }
        
        //修改2个小权重节点的双亲
        tree[k1].parent = i;
        tree[k2].parent = i;
        
        //修改parent的权重
        tree[i].weight = tree[k1].weight + tree[k2].weight;
        
        //修改parent的孩子
        tree[i].lchild = k1;
        tree[i].rchild = k2;
    }
}
 
long long ans;
void dfs(HuffNode *hfmtree,int root)
{
    if (hfmtree[root].lchild!=-1 && hfmtree[root].rchild!=-1)
    {
        ans+=hfmtree[root].weight;
        dfs(hfmtree, hfmtree[root].lchild);
        dfs(hfmtree, hfmtree[root].rchild);
    }
}
struct HuffNode hfmtree[MAX_SIZE];
int main()
{
    int n, m;
    struct HuffCode hcode[MAX_SIZE];
    while(scanf("%d", &n) && n)
    {
        //哈夫曼树总共结点数
        ans=0;
        m = 2 * n - 1;
        
        //初始化哈夫曼树
        initHuffmantree(hfmtree, m);
        
        //权限赋值
        for(int i = 0; i < n; i ++)
        {
            
            scanf("%d", &hfmtree[i].weight);
        }
        
        //构造哈夫曼树
        createHuffmantree(hfmtree, n, m);
        //        createHuffmancode(hfmtree, hcode, n);
        //        printf("%d\n", hfmtree[m - 1].weight);
        
        //        for (int i=0; i<n; i++)
        //        {
        //            printf("%s\n",hcode[i].code);
        //        }
        
        for (int i=n;i<2*n-1; i++)
        {
            if(hfmtree[i].parent==-1)
            {
                dfs(hfmtree,i);
                break;
            }
        }
        
        //     for (int i=0; i<2*n-2; i++)
        //     {
        //         ans+=hfmtree[i].weight;
        //     }
        printf("%lld\n",ans);
    }
    
    return 0;
}
```

#### (2) hdu 2527

Problem Description
Javac++ 一天在看计算机的书籍的时候，看到了一个有趣的东西！每一串字符都可以被编码成一些数字来储存信息，但是不同的编码方式得到的储存空间是不一样的！并且当储存空间大于一定的值的时候是不安全的！所以Javac++ 就想是否有一种方式是可以得到字符编码最小的空间值！显然这是可以的，因为书上有这一块内容--哈夫曼编码(Huffman Coding)；一个字母的权值等于该字母在字符串中出现的频率。所以Javac++ 想让你帮忙，给你安全数值和一串字符串，并让你判断这个字符串是否是安全的？

Input
输入有多组case，首先是一个数字n表示有n组数据，然后每一组数据是有一个数值m(integer)，和一串字符串没有空格只有包含小写字母组成！

Output
如果字符串的编码值小于等于给定的值则输出yes，否则输出no。

Sample Input

```
2 12 helloworld 66 ithinkyoucandoit
```

Sample Output

```
no yes
```



```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#define MAX_SIZE 1001
#define MIN_NUM 1000000000
struct HuffNode
{
    int weight;
    int lchild;
    int rchild;
    int parent;
};
void initHuffmantree(struct HuffNode *tree, int len)
{
    for(int i = 0; i < len; i ++)
    {
        tree[i].weight = 0;
        tree[i].lchild = -1;
        tree[i].rchild = -1;
        tree[i].parent = -1;
    }
}
void createHuffmantree(struct HuffNode *tree, int n, int m)
{
    int Min1, Min2, k1, k2;
    for(int i = n; i < m; i ++)
    {
        //初始化最小值、次小值
        Min1 = Min2 = MIN_NUM;
        k1 = k2 = -1;
        //在尚未构造二叉树的节点中查找权值最小的两棵树
        for(int k = 0; k < i; k ++)
        {
            if(tree[k].parent == -1)
            {
                if(tree[k].weight <Min1)
                {
                    Min2 = Min1;
                    k2 = k1;
                    Min1 = tree[k].weight;
                    k1 = k;
                }
                else if(tree[k].weight <Min2)
                {
                    Min2 = tree[k].weight;
                    k2 = k;
                }
            }
        }
        
        //修改2个小权重节点的双亲
        tree[k1].parent = i;
        tree[k2].parent = i;
        
        //修改parent的权重
        tree[i].weight = tree[k1].weight + tree[k2].weight;
        
        //修改parent的孩子
        tree[i].lchild = k1;
        tree[i].rchild = k2;
    }
}
 
int ans;
void dfs(HuffNode *hfmtree,int root)
{
    if (hfmtree[root].lchild!=-1 && hfmtree[root].rchild!=-1)
    {
        ans+=hfmtree[root].weight;
        dfs(hfmtree, hfmtree[root].lchild);
        dfs(hfmtree, hfmtree[root].rchild);
    }
}
struct HuffNode hfmtree[MAX_SIZE];
int main()
{
    int n,N,cnt;
    char str[MAX_SIZE];
    scanf("%d",&N);
    int num[30];
    while (N--)
    {
        cnt=0;
        ans=0;
        scanf("%d",&n);
        getchar();
        scanf("%s",str);
        int len=strlen(str);
        memset(num, 0, sizeof(num));
        for (int i=0; i<len; i++)
        {
            num[str[i]-'a'+1]++;
        }
        initHuffmantree(hfmtree,60);
        for (int i=1; i<=26; i++)
        {
            if(num[i])
            {
                hfmtree[cnt++].weight=num[i];
            }
        }
        if(cnt==1) //注意这种特殊情况
        {
            if (hfmtree[0].weight<=n)
                printf("yes\n");
            else
                printf("no\n");
            continue;
        }
        
        //构造哈夫曼树
        createHuffmantree(hfmtree,cnt, cnt*2-1);
        
        for (int i=cnt;i<2*cnt-1; i++)
        {
            if(hfmtree[i].parent==-1)
            {
                dfs(hfmtree,i);
                break;
            }
        }
        if (ans<=n)
            printf("yes\n");
        else
            printf("no\n");
 
    }
    return 0;
}
```

