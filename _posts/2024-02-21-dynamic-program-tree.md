---
layout: post
title: 'Dynamic Program: Tree'
date: 2024-02-21 17:15 +0800
tags: [algorithm, dp, tree]
categories: [algorithm, dp]
author: Jebearssica
---

简单学习一下树上 dp

## 基本概念

在给定关系以树形式存在的问题中时, 如果遇到了符合 dp 解法的情况可以使用树上 dp.

举个例子, 经典的快乐指数问题. 给定一个树, 选择其中的结点满足以下规则:

* 结点权重和最大
* 一个结点若被选中, 则其父结点必定不能被选中

很显然, 大体思路是基于 DFS 的 DP. 对于一个结点而言存在三种状态:

* 父结点不在集合内, 自身也不在集合内
* 父结点不在集合内, 自身在集合内
* 父结点在集合内, 自身不在集合内

因此我们可以定义 `dp[node][0]` 与 `dp[node][1]` 分别表示以 `node` 为根结点的子树, 在根结点是否进入集合的最大权重和. 那么很容易得到以下状态转移方程:

```c++
dp[node][1] = node->val; // init
for (auto child = node->begin_child(); child != node->end_child(); ++child) {
  dp[node][0] += max(dp[child][0], dp[child][1]);
  dp[node][1] += dp[child][0];
}
```

看上去是一种十分容易从 cache-dfs 转换成 dp 的方法.

## 换根

前文描述的完成计算后, 得到的结果实质上是 `dp[root][node][0]` 与 `dp[root][node][1]`, 即我们最终得到的结果是基于以 `root` 为根结点这一条件. 如果存在一种情况, 需要我们根据不同的根结点获得不同的 dp 后的结果. 我们将一次树上 DP 视为一次 DFS, 那么朴素一点的完成方法就是完成 N 次 DFS, 其中 N 为树上结点总数. 这显然是十分低效的, 这时候我们引入换根.

我们令 `dp[a]` 为以 `a` 为根结点的树获得的结果. 那么我们试图完成从状态 `dp[a]` 至 `dp[a->child]` 的转换. 为了方便理解, 直接上题目 [Leetcode 834.树中距离之和](https://leetcode-cn.com/problems/sum-of-distances-in-tree)

> 稍稍分析一下, 看似是求结点距离之和, 实际上是求以不同结点为根的结点深度和 + 1 :). 这里 + 1 指的是如果真的算深度根结点深度为 0, 但题目求的是距离之和(实际推广为子树结点和, 因为边权重为 1).
{: prompt-tip }

那么, 当结点 `a->child` 为根结点时, 该子树上的所有结点深度都减一, 而不在该子树上的所有结点深度都加一. 即我们有 `dp[a->child] = dp[a] - nodes(a->child) + (n - nodes(a->child))`. 即我们在只需要额外一次 DFS 遍历的情况下完成了整棵树的换根的状态转移. 综上, 我们只需要在第一次 DFS 过程中进行一些预处理, 如记录每个结点子树的结点总数以及以某个结点(任意结点都可)的 `dp[node]` (或者其他问题中需要的条件), 在第二次 DFS 过程中完成 DP 的计算.

```c++
/**
* @brief 给定无向连通树, 有 n 个
* @param n 结点总数
* @param edges edges[a][b] 表示结点 a b 之间有一条边
* @return 返回数组, res[a] 表示树中第 a 个结点与其他所有结点间的距离和 
*/
vector<int> sumOfDistancesInTree(int n, vector<vector<int>> &edges) {
  // 构造树
  vector<vector<int>> graph(n, vector<int>());
  for (const auto &e : edges) {
    graph[e[0]].push_back(e[1]);
    graph[e[1]].push_back(e[0]);
  }

  vector<int> dp(n);
  vector<int> size(n, 1); // 子树结点数至少为 1, 如果根据前文描述是深度和的话, 这里初始化为 0
  auto dfs = [&](auto self, const int child, const int parent, const int depth) -> void {
    dp[0] += depth;
    for (const auto &next : graph[child]) {
      if (next != parent) {
        self(self, next, child, depth + 1);
        size[child] += size[next];
      }
    }
  };
  
  // 首轮 dfs 记录子树结点数
  dfs(dfs, 0, -1, 0);

  auto dfsdp = [&](auto self, const int child, const int parent) -> void {
    for (const auto &next : graph[child]) {
      if (next != parent) {
        dp[next] = dp[child] - size[next] + (n - size[next]);
        self(self, next, child);
      }
    }
  };

  // 次轮 dfs 推进 dp 计算
  dfsdp(dfsdp, 0, -1);

  return dp;
}
```

再来一道 [Leetcode 2581.统计可能的树根数目](https://leetcode-cn.com/problems/count-number-of-possible-root-nodes). 分析思路大致类似, 在换根处, 我们考虑从结点 `child` 至 `parent` 之间的换根只会对猜想 `(child, parent)` 与 `(parent, child)` 产生影响. 即如果存在前者猜想, 那么满足条件的猜想个数在换根后将会 -1; 如果存在后者猜想, 那么满足条件的个数将会在换根后 +1.

```c++
class Solution2581
{
private:
    /* data */

    struct pair_int_hash
    {
        // template<class T1, class T2>
        size_t operator()(const pair<int, int> &p) const {
            size_t seed = 0;
            auto hash_val = [&](const int &v) -> void { seed ^= hash<int>()(v) + 0x9e3779b9 + (seed << 6) + (seed >> 2); };
            hash_val(p.first), hash_val(p.second); 
            return seed;
        }
    };
    
public:
    int rootCount(vector<vector<int>> &edges, vector<vector<int>> &guesses, int k) {
        int n = edges.size();
        vector<vector<int>> graph(n + 1, vector<int>());
        for (const auto &e : edges) {
            graph[e[0]].push_back(e[1]);
            graph[e[1]].push_back(e[0]);
        }
        
        int correct_cases = 0;
        std::unordered_set<pair<int, int>, pair_int_hash> g_set;
        for (const auto &g : guesses) {
            g_set.insert({g[0], g[1]});
        }
        auto dfs = [&](auto self, const int node, const int parent) -> void {
            if (g_set.find({parent, node}) != g_set.end()) {
                ++correct_cases;
            }
            for (const auto &c : graph[node]) {
                if (c != parent) {
                    self(self, c, node);
                }
            }
        };

        dfs(dfs, 0, -1);
        
        int res = 0;
        auto dfsdp = [&](auto self, const int node, const int parent, const int corrects) -> void {
            if (corrects >= k) {
                ++res;
            }
            for (const auto &c : graph[node]) {
                if (c != parent) {
                    self(self, c, node, corrects + (
                        g_set.find({c, node}) != g_set.end() ? 1 : 0
                    ) - (
                        g_set.find({node, c}) != g_set.end() ? 1 : 0
                    ));
                }
            }
        };

        dfsdp(dfsdp, 0, -1, correct_cases);
        return res;
    }
};
```

## 总结

树上 dp 的要点在于寻求从父结点至子结点(或是相反)的状态转移方程. 当涉及问题涉及不同根结点下的结果时, 其要点还在于从以当前父结点为根结点至以当前子结点为根结点的状态转移方程.
