



###  前言

并查集是一种树型的数据结构，顾名思义就是有"合并集合"和"查找集合"中的元素"两种操作,用于处理一些不相交集合的合并问题。

并查集的主要操作有

1.合并两个不相交集合(`Union(int x,int y)`)

2.判断两个元素是否属于同一个集合(`Find(int x)`)

3.路径压缩

定义一个数组father,用`father[i]`表示元素i的父亲结点.

初始化每个集合只有其本身（编号从1-N）：

```
for (int i=1; i<=n; i++){
	father[i]=i;
}
```

即对于节点`i`，它的组号也是i 
合并操作（`Union(int x,int y)`），`father[i]=j`,表示j是i的父亲（即`i,j`属于同一集合）

对于查找操作，假设需要确定x所在的的集合，也就是确定集合的代表元。可以沿着`parent[x]`不断在树形结构中向上移动，直到到达根节点。

### 合并操作

下面 通过图示 来展示这个过程：

下面 是一组元素，后面的图示 都是基于这组元素 所做的操作

![图示1](http://img.blog.ztgreat.cn/document/algorithm/20141011150450110.png)



在 **图示1** 上合并元素` 1,2`

![20141011150510655](http://img.blog.ztgreat.cn/document/algorithm/20141011150510655.png)

 在 **图示1** 合并元素 `1,2` ， `1,3`

![20141011150309906](http://img.blog.ztgreat.cn/document/algorithm/20141011150309906.png)





在 **图示1** 合并元素`1,2` ， `1,3`，`5,4`

![20141011150550514](http://img.blog.ztgreat.cn/document/algorithm/20141011150550514.png)





在 **图示1** 合并元素`1,2` ， `1,3`，`5,4`,`5,3`

![20141011150354984](http://img.blog.ztgreat.cn/document/algorithm/20141011150354984.png)



### 查找操作

​	

查找两个元素是否属于同一集合（father[x]=x表示该元素没有合并过，自身为一个集合）

```
int Find(int x){   //查找x所属集合
    while (x!=Father[x]) {
        x=Father[x];
    }
    return Father[x];
 }
```



根据上面就可以写成合并操作了

```
void Union(int x,int y){
    x=Find(x);   //查找所属集合
    y=Find(y);
    if(x!=y){   //不在同一个集合
        Father[x]=y;  //合并
    }
}
```



又如：

![20141011201300323](http://img.blog.ztgreat.cn/document/algorithm/20141011201300323.png)



上面所有元素最终形成三个集合。

### 合并操作的改进

最上面的Union的执行是相当任意的，它通过使得第二课棵树的子树而完成合并，对其进行简单的改进是借助任意的方法打破现有的关系，使得总让较小的树成为较大的树的子树，这种方法叫做按`大小求并`。

#### 树大小规则

进行一次任意的并的结果（分别`Union(5,6)`，`Union(7,8)`，`Union(5,7)`  后`Union(4,5)）`：


![按树大小合并](http://img.blog.ztgreat.cn/document/algorithm/20141011205216953.png)



按大小求并的结果:

![按树大小合并](http://img.blog.ztgreat.cn/document/algorithm/20141011205508183.png)

即总是**size**小的树作为子树和**size**大的树进行合并。这样就能够尽量的保持整棵树的平衡。

在初始情况下，每个组的大小都是1，因为只含有一个节点，所以我们可以使用额外的一个数组来维护每个组的大小，对该数组的初始化也很直观：

```
for (int i = 0; i < N; i++){
    sz[i] = 1;    // 初始情况下，每个组的大小都是1
}
```

而在进行合并的时候，会首先判断待合并的两棵树的大小，然后按照上面图中的思想进行合并，实现代码：

```
void Union(int x, int y){
    int i = find(x);
    int j = find(y);
    if (i == j) return;
    // 将小树作为大树的子树
    if (sz[i] < sz[j]){
        Father[i] = j;
        sz[j] += sz[i];
    } else {
        Father[j] = i;
        sz[i] += sz[j];
    }
}
```



#### 树高度规则

另一种实现方法为按高度求并，我们跟踪每棵树的高度而不是大小并执行那些Union使得浅点的树成为高点的树的子树，因为只有两棵相同深度的的树求并时树的高度才增加（此时树的高度增加1）。

```
void Union(int x,int y) {
    if(height[x]==height[y]){
        height[x]=height[x]+1;
        Father[y]=x;
    } else if (height[x]<height[y]){
        Father[y]=x;
    } else {
        Father[x]=y;
    }
}
```



#### 路径压缩

为了加快查找速度，查找时将x到根节点路径上的所有点的parent设为根节点，该优化方法称为压缩路径。

通过 路径压缩使得结点离根更加的近，一个未采用路径压缩的例子：

![路径压缩1](http://img.blog.ztgreat.cn/document/algorithm/20141011203854648.png)

路径压缩后：

![路径压缩2](http://img.blog.ztgreat.cn/document/algorithm/20141011203927673.png)

这样采用路径压缩后使得Find更加的高效。

实现代码：

```
int Find(int x) {
   if(x!=Father[x]) {
       Father[x]=Find(Father[x]);   //递归找根进行更新
   }
    return Father[x];
}
```

### 总结

并查集是一种支持合并集合和查找集合的一种数据结构，能用均摊线性的复杂度执行各种操作，在kruskal算法、求联通分支数等算法中起到关键的作用。

抛个问题，有一群人，他们之间有若干好友关系。如果两个人有直接或者间接好友关系，那么我们就说他们在同一个朋友圈中，这里解释下，如果Alice是Bob好友的好友，或者好友的好友的好友等等，即通过若干好友可以认识，那么我们说Alice和Bob是间接好友。随着时间的变化，这群人中有可能会有新的朋友关系，这时候我们会对当中某些人是否在同一朋友圈进行询问，如何实现？