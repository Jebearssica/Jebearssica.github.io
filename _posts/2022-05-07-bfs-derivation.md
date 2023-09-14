---
layout: post
title: BFS Derivation
date: 2022-05-07 17:41 +0800
tags: [algorithm, bfs, dijkstra]
categories: [Algorithm, BFS]
author: Jebearssica
math: true
---

借着一道我愿称之为究极 BFS 的题目总结一下 BFS, [Leetcode 2258. 逃离火灾](https://leetcode-cn.com/problems/escape-the-spreading-fire/), 顺便延伸至 dijkstra

## BFS(宽度优先搜索)

和这道题的背景非常契合的, 火焰从开始点向四周扩散, 就是一个典型的 BFS 搜索过程. 我们每次都访问同一层(同一时刻)燃烧的点, 并将周遭可燃点推入队列以供下次访问.

BFS 相较于 DFS 能够得到当前点至起始点的最短路径(包含边, 即遍历层数最少)

### 遍历连通块及其变种

我们可以将一般的 BFS 过程都视为图的遍历, 线性时间( $O(N + M)$, 边数与点数)内可以求出所有连通块.

* 针对无环图来说, 我们不会重复遍历同一个点. 而涉及到有环图, 我们又不希望重复遍历, 我们可以存放遍历过的点(用 hash 或 布尔数组都行)
* 在遍历的过程中, 我们可以记录每个点至起点的最小路径, 此时必须保证是**无权图(或路径权重都为 1)**, 这才能够将遍历层数, 视作路径长度(权重为 1 或 0)

> 你要是想求单源最短路径, 你就用 dijkstra 呗
{: .prompt-tip }

我们可以写出下列代码来解决上述描述到的所有问题

```c++
vector<int> bfs(vector<vector<int>> &G, int start)
{
    // 记录最小路径大小
    vector<int> dis(G.size(), 0);
    // 记录最小路径
    vector<vector<int>> path(G.size(), 0);
    int step = 0;
    queue<int> q;
    q.push(start);
    visit.insert(start);
    // 防止重复访问(进入环), 你借助这个甚至能求有环无权图的最小环(每次重复时, 当前的 step 与即将重复点的最小路径大小 dis[curNode] 的差)
    unordered_set<int> visit;
    // 记录当前遍历的路径
    vector<int> curPath;
    while(!q.empty())
    {
        int sz = q.size();
        ++step;
        for(int i = 0; i < sz; ++i)
        {
            auto curNode = q.front();
            q.pop();
            path[curNode].push_back(curNode);
            for(auto &child : G[curNode])
            {
                if(visit.find(child) != visit.end())
                    continue;
                dis[child] = step;
                q.push(child);
                path[child].insert(path[child].begin(), path[curNode].begin(), path[curNode].end());
            }
            visit.insert(curNode);
        }
    }
    return dis;
}
```

### 2258. 逃离火灾

分析题意我们可以知道火的扩散与人的逃跑都可以看作一次 BFS 的扩散, 那么我们就用两个队列存当前人物可出现的位置(かげぶんしんのじゅつ!)与火蔓延至的最外延. 至于如何确定最多等待的时间, 很显然答案具有二段性, 小于等于结果的一定能逃离, 大于结果的一定不能逃离, 可用二分搜索.

```c++
class Solution
{
public:
    static constexpr int dirs[4][2] = { {0, 1}, {0, -1}, {1, 0}, {-1, 0} };
    int m, n;
    void spreadFire(queue<int> &fire, vector<vector<int>> &grid, vector<bool> &isFire)
    {
        int sz = fire.size();
        for (int idx = 0; idx < sz; ++idx)
        {
            auto pos = fire.front();
            fire.pop();
            for (int i = 0; i < 4; ++i)
            {
                int x = pos / m + dirs[i][0], y = pos % m + dirs[i][1];
                if (x >= 0 && x < n && y >= 0 && y < m && !isFire[x * m + y] && grid[x][y] != 2)
                    fire.push(x * m + y), isFire[x * m + y] = true;
            }
        }
    }
    bool check(vector<vector<int>> &grid, int time, queue<int> &fire, vector<bool> &isFire)
    {
        while (time-- && !fire.empty())
            spreadFire(fire, grid, isFire);
        if (isFire[0])
            return false;
        vector<bool> visit(n * m, false);
        visit[0] = true;
        queue<int> people;
        people.push(0);
        while (!people.empty())
        {
            int sz = people.size();
            for (int idx = 0; idx < sz; ++idx)
            {
                auto pos = people.front();
                people.pop();
                if (!isFire[pos])
                {
                    for (int i = 0; i < 4; ++i)
                    {
                        int x = pos / m + dirs[i][0], y = pos % m + dirs[i][1];
                        if (x >= 0 && x < n && y >= 0 && y < m && !isFire[x * m + y] && grid[x][y] != 2 && !visit[x * m + y])
                        {
                            if (x == n - 1 && y == m - 1)
                                return true;
                            else
                                people.push(x * m + y), visit[x * m + y] = true;
                        }
                    }
                }
            }
            spreadFire(fire, grid, isFire);
        }
        return false;
    }
    int maximumMinutes(vector<vector<int>> &grid)
    {
        n = grid.size(), m = grid[0].size();
        int left = 0, right = n * m;
        // init
        vector<bool> initIsFire(n * m, false);
        queue<int> initFire;
        for (int row = 0; row < n; ++row)
            for (int col = 0; col < m; ++col)
                if (grid[row][col] == 1)
                    initIsFire[row * m + col] = true, initFire.push(row * m + col);
        while (left <= right)
        {
            int mid = left + (right - left) / 2;
            auto fire = initFire;
            auto isFire = initIsFire;
            if (check(grid, mid, fire, isFire))
                left = mid + 1;
            else
                right = mid - 1;
        }
        return right == m * n ? 1e9 : right;
    }
};
```

## 双端队列 BFS( 0-1 BFS )

起因, [Leetcode 2290. 到达角落需要移除障碍物的最小数目](https://leetcode.cn/problems/minimum-obstacle-removal-to-reach-corner/), 一道典型例题. 适用于一个图中, 边权值为 0 或 `value` 的最短路径问题.

主要思想是使用双端队列 `deque` 替换传统队列, 权值为 0 的边入队首, 其他权值的边入队尾. 可以将该算法看作是从 BFS 至 Dijkstra 的一份过渡算法. 即, 适用范围从 无权图->无权混单一权重图->非负权图

```c++
vector<int> bfs0_1(vector<vector<int>> &G, int start)
{
    // 记录起点至其他结点最小路径大小
    vector<int> dis(G.size(), 0);
    deque<int> q;
    q.push_back(start);
    dis[start] = 0;
    while (!q.empty())
    {
        auto curNode = q.front();
        q.pop_front();
        for (auto &child : G[curNode])
        {
            // relax dis
            if (dis[curNode] + weight(curNode, child) < dis[child])
            {
                dis[child] = dis[curNode] + weight(curNode, child);
                if (weight(curNode, child) == 0)
                    q.push_front(child);
                else
                    q.push_back(child);
            }
        }
    }
    return dis;
}
```

> 说到底, 0-1 BFS 已经更偏向于 Dijkstra 了(因为遍历深度已经不代表路径长度, 路径长度和边权有关了), 针对一条边已经需要多次松弛最短路. 但同时, 它又保有 BFS 的一定特征, 即, 队首插入可以使得遍历优先走最短路, 这使得原本偏向暴力 Dijkstra 的时间复杂度获得了优化
{: .prompt-info }

2290的题解如下

```c++
class Solution {
public:
    static constexpr int dirs[4][2] = { {0, 1}, {0, -1}, {1, 0}, {-1, 0} };
    int minimumObstacles(vector<vector<int>>& grid)
    {
        vector<vector<int>> dist(grid.size(), vector<int>(grid[0].size(), 0x3f3f3f3f));
        deque<pair<int,int>> q;
        q.emplace_back(0, 0);
        dist[0][0] = 0;
        while(!q.empty())
        {
            auto [x, y] = q.front();
            q.pop_front();
            for(auto &[dx, dy] : dirs)
            {
                int nextX = x + dx, nextY = y + dy;
                if(nextX >= 0 && nextX < grid.size() && nextY >= 0 && nextY < grid[0].size())
                {
                    if(dist[x][y] + grid[nextX][nextY] < dist[nextX][nextY])
                    {
                        dist[nextX][nextY] = dist[x][y] + grid[nextX][nextY];
                        if(grid[nextX][nextY])
                            q.emplace_back(nextX, nextY);
                        else
                            q.emplace_front(nextX, nextY);
                    }
                }
            }
        }
        return dist[grid.size() - 1][grid[0].size() - 1];
    }
};
```

## 优先队列 BFS

如果将原始 BFS 的普通队列换成优先队列, 我们可以按照一定优先度遍历有权重的点(边的仍然无权), 而这就和 [Dijkstra 中的优先队列优化版本](#优先队列-dijkstra)是一模一样的.

## Dijkstra

Dijkstra 解决正权重有向图中的单源最短路径问题, 主要思路可以通过以下伪代码阐述

```c++
void dijkstra(vector<vector<int>> &G, int s)
{
    set<int> S, Q;
    // Q 初始化为图中所有顶点
    initial(Q, G);
    while(!Q.empty())
    {
        // Q 中最小权重的结点
        int u = minNode(Q);
        S.insert(u);
        // 对最小权重结点, 松弛其邻接点
        for(int &child : G[u])
            relax(u, child, G)
    }
}
```

> 上述的松弛操作指的是更新两点的最短距离, 将原有预估值(INT_MAX)松弛至最终值(最短路径)
{: .prompt-tip }

### 暴力法

```c++
vector<int> dijkstra(vector<vector<pair<int, int>>> &G, int s)
{
    int n = G.size();
    vector<int> dis(n, INT_MAX);
    vector<bool> visit(n, false);
    dis[s] = 0;
    for(int i = 0; i < n; ++i)
    {
        // min weight in Q
        int u = n, minWeight = INT_MAX
        for(int j = 0; j < n; ++j)
            if(!visit[j] && dis[j] < minWeight)
                u = j, minWeight = dis[j];
        // Q -> S
        visit[u] = true;
        // relax
        for(auto next : G[u])
            if(dis[next.first] > dis[u] + next.second)
                dis[next.first] = dis[u] + next.second;
    }
    return dis;
}
```

> 要小心 `dis[u] + next.second` 这个如果真的用 INT_MAX 初始化的话, 可能会爆 INT. 可以初始化为0x3f3f3f3f 来近似一个无限大
{: .prompt-warning }

遍历整个图的每个顶点, 时间复杂度 $O(V)$, 找最小元素 $O(V)$, 遍历点的总时间复杂度 $O(V^2)$. 我们对每条边都进行了遍历(松弛), 时间复杂度 $O(E)$, 因此暴力 dijkstra 的总时间复杂度为 $O(V^2+E)$

### 优先队列 Dijkstra

找最小结点时通过优先队列

```c++
vector<int> dijkstra(vector<vector<pair<int, int>>> &G, int s)
{
    int n = G.size();
    auto cmp = [&](const pair<int, int> &left, const pair<int, int> &right)
        { return left.first < right.first; };
    priority_queue<pair<int, int>, vector<pair<int, int>>, decltype(cmp)> pq(cmp);
    vector<int> dis(n, INT_MAX);
    dis[s] = 0;
    vector<bool> visit(n, false);
    pq.emplace(s, 0);
    while(!pq.empty())
    {
        auto u = pq.front();
        pq.pop();
        if(visit[u.first])
            continue;
        visit[u.first] = true;
        for(auto next : G[u.first])
            if(dis[next.first] > dis[u] + next.second)
                dis[next.first] = dis[u] + next.second, pq.emplace(next.first, dis[next.first]);
    }
    return dis;
}
```

每条边都会遍历到以保证正确松弛得到最小路径(多次松弛), 因此最外层遍历时间复杂度 $O(E)$, 针对优先队列的 push/pop , 时间复杂度为 $O(\log{E})$, 总时间复杂度为 $O(E\log{E})$

除此之外, 还有通过二叉堆或斐波那契堆的方法优化, 目前用不上就不管了
