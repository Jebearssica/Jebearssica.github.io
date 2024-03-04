---
layout: post
title: Prefix Sum and Adjacent Difference
date: 2023-12-14 16:53 +0800
tags: [algorithm]
categories: [algorithm]
author: Jebearssica
math: true
---

理论上来讲前缀和以及差分两种思路应该是算法基础, 是那种堪比排序的基础, 按理说应该掌握得很好才对. 但由于没有系统性学习, 导致后面在学习部分变种树结构的时候非常吃力.

## 前缀和

是一种预处理方法, 能够实现后续相关查询达到 $O(1)$ 的时间, 以下又分几种情况

### 一维前缀和

这个方法通常解决"数组前 n 项之和"这个问题, 如下

```c++
std::vector<int> prefixSum(const std::vector<int> &input) {
  std::vector<int> res;
  res.reserve(input.size());
  res.push_back(input.front());
  for (size_t idx = 1; idx < input.size(); ++idx) {
    res.push_back(input[idx] + res.back());
  }
  return res;
}
// STL 贴心提供了接口, 要善用 STL
std::vector<int> prefixSum(const std::vector<int> &input) {
  std::vector<int> res(input.size(), 0);
  std::partial_sum(input.begin(), input.end(), res.begin());
  return res;
}
```

可以稍微包装一下, 变成一个中等题(事实上还是简单题), 或者变成一个不能直接用上知识需要绕一下的简单题. 如, 求数组的某个连续区间的和.

```c++
// calc sub sum from (start, end]
int sub_sum(const std::vector<int> &input, size_t start, size_t end) {
  auto info = prefixSum(input);
  return info[end] - info[start];
}
```

> 易错点, 在构造前缀和时, 应当考虑后续使用的时候的区间问题, 进一步, 你需要考虑 `prefix[idx]` 代表的前缀和究竟是否包括当前下标指代的元素.
{: prompt-tip }

或者做为一个困难问题中的解法的前置知识. 如, 树状数组等等

### 二维前缀和

进一步, 根据一维前缀和的定义 $sum_i = sum_{i - 1} + a_i$, 借助容斥原理我们可以推广至二维情况 $sum_{i, j} = sum_{i - 1, j} + sum_{i, j - 1} - sum_{i - 1, j - 1} + a_{i, j}$

进而, 对于二维数组的子区间内的和也可以类似计算

```c++
int sub_sum(const std::vector<std::vector<int>> &input, std::pair<size_t, size_t> &start, std::pair<size_t, size_t> &end) {
  auto info = prefixSum(input); // 没具体实现, 但原理类似
  return info[end.first][end.second] - info[start.first][end.second] - info[end.first][start.second] + info[start.first][start.second];  // 依旧基于容斥原理
}
```

### 树上前缀和

更进一步, 我们可以推广应用至多叉树, 很显然, 容斥原理依旧有效, 但我们需要借助最近公共祖先(LCA)来辅助. 此时, 我们将 $sum_{i}$ 定义为结点 `i` 至根结点的权重和, 因此, 我们有:

```c++
// 权重为点上权重
int path_sum_point_weight (Node &i, Node &j) {
  Node lca = calc_lca(i, j);
  auto info_i = prefixSum(i), info_j = prefixSum(j), info_lca = prefixSum(lca), info_lca_parent = prefix(lca->parent());
  return info_i + info_j - info_lca - info_lca_parent;
}
// 权重为边上权重
int path_sum_edge_weight (Node &i, Node &j) {
  Node lca = calc_lca(i, j);
  auto info_i = prefixSum(i), info_j = prefixSum(j), info_lca = prefixSum(lca);
  return info_i + info_j - 2 * info_lca;
}
```

其中 `lca` 代表公共祖先结点, `info_lca_parent` 为结点 `lca` 的父结点

### 基于DP的多维前缀和

> 你需要一点点二进制集合遍历的基础做为前置知识
{: prompt-tip }

当维度进一步上升, 求解多维子空间集合的和如果依旧依赖于容斥原理的话将会变得非常复杂, 因此引入 DP 进行计算. 我们假设这里有个 N 维数组, 我们需要求其前缀和. 可以将 N 维数组的状态以二进制表示, 因此这个 N 维问题会被转换为二进制下的集合遍历问题.

```c++
std::vector<int> prefixSum(1 << n);
for (int mask = 0; mask < (1 << n); ++mask) {
  for (int sub = 0; sub; sub = (sub - 1) & mask) {
    prefix[mask] += input[sub];
  }
}
```

其时间复杂度可以根据每个集合状态 `mask` 被其每个子集各遍历一次来计算. 假设一个 `mask` 状态中有 `k` 个元素, 那么其共有 $2^{n-k}$ 个子集. 整体上, 我们需要处理 `[0, n]` 个元素的所有状态, 那么总时间复杂度为 $O(\Sigma_{0}^{n}(\dbinom{n}{m} \cdot 2^{n-k}))$. 根据二项式定理, 有 $O(3^n)$

> 对于总体时间复杂度的理解, 我们可以认为有 `k` 个元素的状态一共有 `n` 选 `k` 个那么多.
{: prompt-tip }

考虑到上述的朴素解法并没有利用到已经计算完毕的前缀和做为缓存, 即我们可以使用动态规划将一个子集和的前缀和的计算由朴素计算转换为状态转移. 

> 为何能够联想到动态规划? 其实能够想到一个集合 `subset` 可能是多个集合的子集. 那么很显然, 这个子集计算的结果能够直接作用在其父集, 类似记忆化 DFS(实际上就是迭代与递归之间的转换). 说点人话, 高维前缀和是由前一维度的已知状态转移而来, 且状态转移的过程中不会有重复.
{: prompt-info }

我们定义 `dp[state]` 为状态 `state` 下的所有子集的和, 为了使得状态互相独立, 我们定义 `dp[state][i]` 为状态 `state` 下前 `i` 位相同的所有子集的和(下标从 1 开始, 由低位至高位计数).

> 状态转移方程中的状态势必是离散的, 即面对一个问题, 如果其状态之间相交, 则应当构造一定的规则将相交状态进行划分.
{: prompt-tip }

我们考虑状态转移方程, 从 `state` 第 `i` 位开始. 如果为 0, 则该状态下的子集情况与 `dp[state][i - 1]` 完全一致. 因此有 `dp[state][i] = dp[state][i - 1]`; 如果为 1, 则该状态下的子集情况还要累加上, 令该位为 0 的子集情况, 即 `dp[state][i] = dp[state][i - 1] + dp[state ^ (1 << i)][i - 1]`. 这个的原理是, 对于一个状态 `state` 的第 `i` 位, 如果为 0, 则其所有子集必定在该位上为 0; 如果为 1, 则子集在该位上的值可以为 0 或 1.

接着考虑初始状态, 很显然, `dp[state][0] = sum(input, state)`

综上, 因此我们有以下代码:

```c++
// n 代表当前维度
// state 为二进制后表示的状态
// dp[state][i]: 从低位开始前 i 位与 state 相同的集合的对状态 state 的前缀和的贡献值, i 属于 [1, n], i 为 0 表示初始状态
// input[state]: 表示输入数组在状态 state 下的和
std::vector<std::vector<int>> dp (1 << n, std::vector<int>(n));
for (int mask = 0; mask < (1 << n); ++mask) {
  // 初始状态: state 本身对 prefix[state] 的贡献为 input[state] 
  dp[mask][0] = input[mask];
  for (int i = 0; i < n; ++i) {
    dp[mask][i] = dp[mask][i - 1];
    if (mask & (1 << i)) {
      dp[mask][i] += dp[mask ^ (1 << i)][i - 1];
    }
  }
}
```

很显然, 可以压缩掉一个维度, 如下:

```c++
std::vector<int> dp(1 << n);
// init
for (size_t state = 0; state < (1 << n); ++state) {
  dp[state] = input[state];
}
// dp
for (size_t i = 0; i < n; ++i) {
  for (size_t mask = 0; mask < (1 << n); ++mask) {
    if (mask & (1 << i)) {
      dp[mask] += dp[mask ^ (1 << i)];
    }
  }
}
```

时间复杂度, 最外层遍历位数 $O(n)$, 最内层遍历所有状态 $O(2^n)$, 总时间复杂度 $ O(n2^n)$

## 差分

可以理解为前缀和的逆运算, 记录邻近元素之间的差值 $diff_{i} = I_{i} - I_{i - 1}, i \in [2, n]$, 另通常定义首个元素为 $diff_{1} = I_{1}$. 此时差分数组具备如下性质:

* 差分数组的前缀和为输入数组本身, 即 $I_{i} = \Sigma_{n=1}^{i}(diff_{n})$
* 输入数组的前缀和可以通过差分数组求得: $\Sigma_{i=1}^{n}(I_{i}) = \Sigma_{i=1}^{n}(\Sigma_{j=1}^{i}(diff_{j})) = \Sigma_{i=1}^{n}((n - i + 1)b_{i})$
* 可用于维护多次对序列上的某区间的所有元素增加(或减少)一个值, 并在修改完毕后查询某一位的值
  * 事实上就是差分数组

### 树上差分

我们将树上任意两个结点之间的路径视为一个序列, 那么在这个序列上进行差分操作即代表着树上差分. 例如, 访问一个路径集合, 对该集合内的每个点都做一次结点上(或边上)权重递增的操作就可以使用树上差分维护这个信息.

当然同样的, 树上差分也分点差分与边差分. 这两者实现上略有区别, 但思路大致相同, 都是借助 LCA 实现的差分.

#### 树上点差分

对于一次从 `s` 至 `e` 的路径 $\delta_{(s,e)}$, 如果按照一般做法 dfs 遍历路径上所有结点并进行递增将会非常耗时(时间复杂度和路径高度有关), 此时如果通过树上差分的方式将会将时间复杂度缩小至常数级(你只需要遍历最多四个结点).

根据树结构令两个结点的最近公共祖先结点为 lca, 我们有 $\delta_{(s,e)} = \delta_{(s, lca)} + \delta_{(e, lca_{child})}$. 我们将拆分后的路径视为之前的一维数组. 那么针对 $\delta_{(s, lca)}$ 的访问的记录, 等价于对结点 `s` 递增, 对结点 `lca` 的父亲结点递减. 综上我们有:

```c++
// s -> lca
node[s]++;
node[lca].parent--;
// e -> lca_child
node[e]++;
node[lca]--;

// equals to
node[s]++, node[e]++, node[lca]--, node[lca].parent--;
```

#### 树上边差分

大致原理类似, 只是由于对边权重进行操作, 因此不涉及对 LCA 结点的父结点进行操作. 省略分析过程如下:

```c++
// 事实上是把边权重绑定在子结点上, 即 a -> b 的权重值在孩子结点 a 上表示
node[s]++, node[e]++, node[lca] -= 2;
```

## 参考

* [oi wiki](https://oi-wiki.org/basic/prefix-sum)
* [codeforces](https://codeforces.com/blog/entry/45223)
