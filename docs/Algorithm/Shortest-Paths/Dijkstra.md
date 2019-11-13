单源最短路径问题
给定一个带权有向图 G=(V,E) ，其中每条边的权是一个非负实数。另外，还给定 V 中的一个顶点，称为源。现在我们要计算从源到所有其他各顶点的最短路径长度。这里的长度是指路上各边权之和。这个问题通常称为单源最短路径问题。

### 最短路径的最优子结构性质

   该性质描述为：如果`P(i,j)={Vi....Vk..Vs...Vj}`是从顶点i到j的最短路径，k和s是这条路径上的一个中间顶点，那么`P(k,s)`必定是从k到s的最短路径。下面证明该性质的正确性。

   假设`P(i,j)={Vi....Vk..Vs...Vj}`是从顶点i到j的最短路径，则有`P(i,j)=P(i,k)+P(k,s)+P(s,j)`。而P(k,s)不是从k到s的最短距离，那么必定存在另一条从k到s的最短路径P'(k,s)，那么`P'(i,j)=P(i,k)+P'(k,s)+P(s,j)<P(i,j)`。则与`P(i,j)`是从i到j的最短路径相矛盾。因此该性质得证。

### Dijkstra算法

   由上述性质可知，如果存在一条从i到j的最短路径(`Vi.....Vk,Vj`)，Vk是Vj前面的一顶点。那么(Vi...Vk)也必定是从i到k的最短路径。为了求出最短路径，Dijkstra就提出了以最短路径长度递增，逐次生成最短路径的算法。譬如对于源顶点V0，首先选择其直接相邻的顶点中长度最短的顶点Vi，那么当前已知可得从V0到达Vj顶点的最短距离`dist[j]=min{dist[j],dist[i]+map[i][j]}`。根据这种思路，

假设存在`G=<V,E>`，源顶点为V0，U={V0},`dist[i]`记录V0到i的最短距离，`path[i]`记录从V0到i路径上的i前面的一个顶点。

1.从V-U中选择使`dist[i]`值最小的顶点i，将i加入到U中；

2.更新与i直接相邻顶点的dist值。(`dist[j]=min{dist[j],dist[i]+map[i][j]}`)

3.知道U=V，停止。

举例hdu 2544 最短路

```
#include <iostream>
#include <cstdio>
#include <cstring>
#include <algorithm>
using namespace std;
int map[101][101];
const int Max=1000000;
bool path[101];    //路径记录
int dist[101];     //用于存储到各点最短路径的权值和 dist[v]表示v0到v的最短路径长度
int dijkstra(int v0,int n)
{
    
    bool visit[101];   //顶点是否加入集合
    for (int i=0; i<n; i++)
    {
        visit[i]=false;  //都还没有加入集合状态
        dist[i]=map[v0][i];  //将与v0点有连线的顶点加上权值
        path[i]=v0;        //初始化为v0
    }
    dist[v0]=0;  //v0-v0路径为0
    visit[v0]=true;  //v0已加入集合，标记
    for (int i=0; i<n-1; i++)  //选择n-1次
    {
        int min=1000000;
        int k=-1;
        for (int j=0; j<n; j++)
        {
            if(!visit[j] && min>dist[j])   //寻找离v0最近的未加入集合的顶点
            {
                min=dist[j];
                k=j;
            }
        }
        visit[k]=true;
        for (int j=0; j<n; j++)
        {
            //如果经过v顶点的路径比现在这条路径的长度还要短的话
            if(!visit[j] && min+map[k][j]<dist[j])
            {//说明找到了更短的路径，修改dist,path.
                dist[j]=min+map[k][j];  //修改当前路径长度
                path[j]=k;    //当前路径经过k
            }
        }
    }
    return dist[n-1];
}
void Display(int v0,int v)  //逆向追踪    path[v1]=v2表示v0到v中顶点v1的前驱是v2.
{
   
    int num[100];
    int pos=0;
    printf("%d-%d weight:%d\n",v0+1,v+1,dist[v]);
    num[pos++]=v+1;
    int k=path[v];
    while (k!=v0)
    {
        num[pos++]=k+1;
        k=path[k];
    }
    num[pos++]=v0+1;
    
    
    cout<<"path:";
    for (int i=pos-1; i>=0; i--)
    {
        cout<<num[i]<<" ";
    }
    cout<<endl;
    
}
int main()
{
    int n,m;
    while (cin>>n>>m && (n || m))
    {
        int i,j;
        int c;
        memset(map, Max, sizeof(map));
        for (int k=0; k<m; k++)
        {
            cin>>i>>j>>c;
            map[i-1][j-1]=c;  //输入顶点最小是从1开始的，而计算写的是从顶点0开始，所以下标要减一，map也可以从1开始存储数据。
            map[j-1][i-1]=c;   //无向图
        }
        int ans=dijkstra(0,n);
        cout<<ans<<endl;
        
        Display(0, n-1);
    }
    return 0;
}
```



