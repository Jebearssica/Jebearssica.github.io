---
layout: post
title: Disjoint Set Union
date: 2022-07-23 09:48 +0800
tags: [data structure, dsu]
categories: [Data Structure, DSU]
author: Jebearssica
math: true
---

> 我以为我懂了, 实际没总结我就不懂. 当然, 我以为我总结了我就应该懂了, 实际我不知道我懂没懂. ——废话大师本人

起因[Leetcode 2334. 元素值大于变化阈值的子数组](https://leetcode.cn/problems/subarray-with-elements-greater-than-varying-threshold/), 周赛这道无从下手, 根本看不出是并查集. 当然后来第一反应应该是双指针 + 单调栈(?), 但实际单调栈也没怎么看出. 总而言之, 先总结一下并查集

## 朴素并查集大致思想

一个森林结构支持查找与合并, 能够处理不相交集合的合并与查询, 但不支持集合的分离.

### 初始化

给定一个值 `n`, 代表构成该并查集的最终结点总数. 那么初始化时的连通分量为 `n`, 且每颗子树大小为 1, 每个子树根结点的父亲结点为自身(定义)

> 有了该根结点的定义, 那么后续查询就可以根据该结点的父亲结点是否是其本身而判断是否为根结点了.
{: .prompt-info }

```c++
private:
    vector<int> size;
    vector<int> parent;
    int cnt;

public:
    DSU(int n) : size(n, 1), parent(n), cnt(n)
    {
        for (int idx = 0; idx < n; ++idx)
            parent[idx] = idx;
    }
```

### 查找

查询的关键思想在于, 每个结点询问父亲结点, 最终询问至根结点时, 就找到了搜索结点的祖先结点. 那么一个朴素的查找可以如下

```c++
int find(int x)
{
    while(x != parent[x])
        x = parent[x];
    return x;
}
```

然而, 这样查询保留了许多额外信息, 如, 我的父亲是谁, 我的爷爷是谁......以及我的真实祖先是谁. 当我们并不需要上述信息, 仅仅是想要知道自己在哪个"家族"(子树)时, 我们可以"人为选举一个族长"(反正谁当祖先我不在意), 通过**路径压缩**实现快速查询祖先.

> 事实上, 如果不进行路径压缩, 在一些特殊情况下, 子树可能退化成链表, 查找的时间复杂度就很高了
{: .prompt-info }

```c++
int find(int x)
{
    while(x != parent[x])
    {
        parent[x] = parent[parent[x]];
        x = parent[x];
    }
    return x;
}
```

路径压缩使得最终的子树高度为小于等于2, 即根结点连接着所有孩子结点

> 事实上, 在中间合并的时候树高会为3, 然而由于每次查找都会使得树高度变为2, 因此最终高度应该是2(你总不可能插入了不查询吧)
{: .prompt-tip }

### 合并

同样的, 合并时我们也不关注祖先是谁, 因此两个根结点直接连接即可

```c++
bool merge(int x, int y)
{
    x = find(x), y = find(y);
    // check whether in same component
    if(x == y)
        return false;
    parent[y] = x;
    return true;
}
```

然而, 进一步来说, 当我们总是把更小的子树插入至更大子树中, 对于接下来的查询来说, 最坏时间复杂度一定更优(查询链肯定变短). 在[OI WIKI](https://oi-wiki.org/ds/dsu/)提及了所涉及的论文, 且在许多场景下, 我们不能使用前文的路径压缩, 因此我们可以使用**按秩合并**的方法进行优化.

通常来说, 判断"树大小", 可以通过结点数或树深度来评估, 以下给出结点数的实现方法

```c++
bool merge(int x, int y)
{
    x = find(x), y = find(y);
    // check whether in same component
    if(x == y)
        return false;
    // ensure size[x] > size[y]
    if(size[x] < size[y])
        swap(x, y);
    parent[y] = x;
    size[x] += size[y];
    return true;
}
```

### 时间复杂度分析

同时使用路径压缩和按秩合并优化后, 可以视为常数时间复杂度, 证明不会, 详见[这里是地狱](https://oi-wiki.org/ds/dsu-complexity/)

## 并查集应用

> 大量 Hard 题预警! 下方内容本人不敢保证 100% 吃透!
{: .prompt-danger }

### 朴素应用: 图论的合并与查询

[Leetcode 685. 冗余连接 II](https://leetcode.cn/problems/redundant-connection-ii/), 题目大意十分明确, 找出有向图中删除的一条边使其变成有根树(根结点无父结点, 其余结点只有一个父结点). 给出的有向边 \[x, y\] 指的是 x 是 y 的父结点

给出的非合法图只有两种情况, 有环或多父亲. 那么建图过程中可以利用并查集来判断是否成环(并查集的简单应用), 同时维护每个结点的入度来判断是否有多个父亲结点. 然而可能有三种情况出现:

* 有环无冲突: 删除构建成环的边(最后遍历到的边)
* 有冲突无环: 删除构成冲突的边(最后遍历到的边)
* 有环有冲突: 删掉出度为2(冲突结点, 有多个父结点的结点)中, 构成环的边

```c++
#include <vector>
using namespace std;
class UF
{
private:
public:
    vector<int> parent;
    vector<int> indegree;
    int cnt;
    int find(int x)
    {
        while (this->parent[x] != x)
        {
            // this->parent[x] = this->parent[this->parent[x]];
            x = this->parent[x];
        }
        return x;
    }
    int findChild(int start, int target)
    {
        while (this->parent[start] != target)
        {
            // this->parent[start] = this->parent[this->parent[start]];
            // cout << "find child node: " << start << endl;
            start = this->parent[start];
        }
        return start;
    }
    bool merge(int p, int q)
    {
        int rootP = find(p);
        int rootQ = find(q);
        if (rootP == rootQ)
            return false;
        this->parent[p] = q;
        this->indegree[q]++;
        return true;
    }
    bool isConflict(int x)
    {
        return this->indegree[x] == 1;
    }
    UF(int n)
    {
        this->cnt = n;
        this->parent.resize(n);
        this->indegree.resize(n);
        for (int idx = 0; idx < n; idx++)
        {
            this->parent[idx] = idx;
            this->indegree[idx] = 1;
        }
    }
    ~UF() {}
};
class Solution
{
public:
    vector<int> findRedundantDirectedConnection(vector<vector<int>> &edges)
    {
        UF uf(edges.size() + 1);
        int conflict = -1, cycle = -1, idx = 0;
        for (auto edge : edges)
        {
            // this node's indgree is 2
            if (!uf.isConflict(edge[1]))
                conflict = idx;
            else if (!uf.merge(edge[0], edge[1]))
                cycle = idx;
            ++idx;
        }
        if (conflict < 0)
            return edges[cycle];
        else if (cycle < 0)
            return edges[conflict];
        else
            return vector<int>{uf.findChild(edges[cycle][1], edges[conflict][1]), edges[conflict][1]};
    }
};
```

小结, 这类题想到用并查集实现其中某些部分的判断不难, 难点更多在于其中图论的分类讨论以及与图论基础知识的结合.

### 进阶应用: 二元 `==` 等价关系转换为图论问题

[Leetcode 565. 数组嵌套](https://leetcode.cn/problems/array-nesting/), 说实话, 我是因为碰到了[下面](#高阶应用-二元-fx-等价关系转换为图论问题), 才能看出这道题用并查集(果然看不出解法是因为见少了).

大致思想是, 并查集所维护的是动态连通性. 我们将其视作等价关系(自反的, 对称的, 传递的), 那么此题中的 `==` 就很显然也是一种等价关系, 那么就可以使用并查集来进行维护.

题意分析: 重点就是**索引也是值**, 因此合并每个值与其对应索引, 返回最大的连通分量的秩.

```c++
class Solution {
private:
    vector<int> size;
    vector<int> parent;
    int find(int x)
    {
        while(x != parent[x])
        {
            parent[x] = parent[parent[x]];
            x = parent[x];
        }
        return x;
    }
    int merge(int x, int y)
    {
        x = find(x), y = find(y);
        if(x != y)
        {
            if(size[x] < size[y])
                swap(x, y);
            size[x] += size[y];
            parent[y] = x;
        }
        return size[x];
    }
public:
    int arrayNesting(vector<int>& nums)
    {
        int n = nums.size();
        size.resize(n, 1);
        parent.resize(n);
        for(int idx = 0; idx < n; ++idx)
            parent[idx] = idx;
        int res = 1;
        for(int idx = 0; idx < n; ++idx)
            res = max(res, merge(nums[idx], idx));
        return res;
    }
};
```

### 高阶应用: 二元 `f(x)` 等价关系转换为图论问题

[Leetcode 2334. 元素值大于变化阈值的子数组](https://leetcode.cn/problems/subarray-with-elements-greater-than-varying-threshold/)这道题就是典型的, 除非你数学足够好, 要不然第一次见看不出并查集解法的一道题(实际上根据上一章, 我们可以将题目条件抽象为一个函数 `f(x)`).

> 用单调栈显然是一个对于我来说更容易看出的解法, 然而用并查集我个人完全无法得到正确的分析题意的步骤, 只能照猫画虎. 所以实际上看着也不是那么(典型?)
{: .prompt-danger }

当然作为周赛T4, 我一开始是看不出并查集的(现在也很难看出来). 那么进行一些题意分析: 很显然, 有一个贪心原则那就是, 单个元素越大且子数组越长, 越有可能满足题意. 那么我们可以从大到小分析元素, 并逐步合并遍历过的集合.

为了满足题目中的**子数组**条件, 我们需要将遍历至的元素与前后元素连接起来. 实际上, 我们具体实现时可以选择前或后一个方向连接即可.

> 注意, 当前结点的下(上)一结点是还未遍历的结点, 因此计算连通分量的秩时要减一. 此外, 由于我们总是合并一侧结点, 因此我们可以确保连通结点除去右侧结尾点, 其余结点都已遍历(这个非常重要要想清楚)
{: .prompt-tip }

那么我们可以写出以下题解

```c++
class Solution {
private:
    vector<int> size;
    vector<int> parent;
    int find(int x)
    {
        while(x != parent[x])
        {
            parent[x] = parent[parent[x]];
            x = parent[x];
        }
        return x;
    }
    int merge(int x, int y)
    {
        x = find(x), y = find(y);
        if(size[x] < size[y])
            swap(x, y);
        parent[y] = x;
        size[x] += size[y];
        return size[x];
    }
public:
    int validSubarraySize(vector<int>& nums, int threshold)
    {
        int n = nums.size();
        size.resize(n + 1, 1);
        parent.resize(n + 1);
        vector<pair<int, int>> idxAndNums(n);
        // 初始化父亲结点 & 记录索引
        for(int idx = 0; idx < n; ++idx)
            parent[idx] = idx, idxAndNums[idx] = {nums[idx], idx};
        parent[n] = n;
        // 按值降序排序
        sort(idxAndNums.begin(), idxAndNums.end(), [&](const pair<int, int> &a, const pair<int, int> &b) { return a.first > b.first; });
        // 逐个遍历
        for(int idx = 0; idx < n; ++idx)
        {
            // 遍历点与右侧结点连通后, 求当前连通分量的秩
            auto cur = merge(idxAndNums[idx].second, idxAndNums[idx].second + 1)
            // 判断是否满足题意
            if(nums[idxAndNums[idx].second] * (cur - 1) > threshold)
                return cur - 1;
        }
        return -1;
    }
};
```

小结, 这类题切入点就很难, 难点不在于利用并查集去解该问题, 而是如何**将一个看起来与图论无关的问题转换(亦或者它本身就是)图论问题**, 按照本人莽夫特质来说, 一切的不熟悉都是训练量未达到, 可能就是没怎么见多识广所以无法转换?

## 总结

并查集, 简单来说就是检查集合动态连通性. 这里的动态指的是能够动态维护集合连通性. 那么从数学上思考集合这个概念是非常广泛的, 涉及数论与图论, 因此用到的地方也挺广泛的. 一般来说看到图论的连通性的时候会首先想到, 其他时候应该是走投无路才会在非图论情况下用到并查集吧(毕竟转换成图论连通性就不熟)
