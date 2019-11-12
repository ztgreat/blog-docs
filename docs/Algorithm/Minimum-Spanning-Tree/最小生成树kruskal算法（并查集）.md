**Kruskal算法**是一种用来查找最小生成树的算法

kruskal算法总共选择n- 1条边，所使用的贪婪准则是：从剩下的边中选择一条不会产生环路的具有最小耗费的边加入已选择的边的集合中。注意到所选取的边若产生环路则不可能形成一棵生成树。kruskal算法分e 步，其中e 是网络中边的数目。按耗费递增的顺序来考虑这e 条边，每次考虑一条边。当考虑某条边时，若将其加入到已选边的集合中会出现环路，则将其抛弃，否则，将它选入。经过n-1次选择后就构成了最小的一个树。

算法过程（采用**并查集**）：

1.将图各边按照权值进行排序

2.将图遍历一次，找出权值最小的边，（条件：此次找出的边不能和已加入最小生成树集合的边构成环），若符合条件，则加入最小生成树的集合中。不符合条件则继续遍历图，寻找下一个最小权值的边。

3.递归重复步骤1，直到找出n-1条边为止（设图有n个结点，则最小生成树的边数应为n-1条），算法结束。得到的就是此图的最小生成树。



![20141016003341734](http://img.blog.ztgreat.cn/document/algorithm/20141016003341734.jpg)



```
#include <iostream>
#include <algorithm>
using namespace std;
struct node
{
    int u;   //顶点
    int v;   //顶点
    int w;   //边的权值
};
int cmp(node a,node b)
{
    return a.w<b.w;
}
int father[1010];
int Find(int x) 
{
    while (x!=father[x])
    {
        x=father[x];
    }
    return x;
}
void Union(int u,int v)
{
    u=Find(u);
    v=Find(v);
    if(u!=v)
    {
        father[u]=v;
    }
}
int main()
{
    int n,m;
    node edges[15001];
    while (cin>>n>>m)
    {
        for (int i=1; i<=n; i++)
        {
            father[i]=i;
        }
        for (int i=0; i<m; i++)
        {
            cin>>edges[i].u>>edges[i].v>>edges[i].w;   //边及权值
        }
        sort(edges, edges+m, cmp);  //按权值升序排序
        int max=0;         //最长网线
        node num[15001];   //输出的集线器的编号
        int ans=0;         //网线数量
        for (int i=0; i<m; i++)  //从最小的边开始合并
        {
            if (Find(edges[i].u)!=Find(edges[i].v)) //如果不会成环就进行合并操作（没有公共祖先）
            {
                Union(edges[i].u, edges[i].v);  //合并（将此边加入集合）
                if(max<edges[i].w)
                {
                    max=edges[i].w;
                }
                num[ans].u=edges[i].u;
                num[ans].v=edges[i].v;
                ans++;
            }
            if(ans>=n-1)  //最多选n-1条边
            {
                break;
            }
        }
        cout<<max<<endl;
        cout<<ans<<endl;
        for (int i=0; i<ans; i++)  //生成树的”样子“
        {
            cout<<num[i].u<<" "<<num[i].v<<endl;
        }
    }
    return 0;
}
```

