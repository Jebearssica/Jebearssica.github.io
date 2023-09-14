---
layout: post
title: Dynamic Program
date: 2022-04-30 13:21 +0800
tags: [algorithm, dp]
categories: [Algorithm, DP]
author: Jebearssica
math: true
---

> 一切动态规划问题, 都是有穷自动机  ——Jebearssica

通常而言, 做动态规划都涉及求**最值**, 一般的解题思路有自上而下递归和自底向上动规. 按照人类思路自上而下递归比较好写, 写出来之后用哈希表剪枝防止重复递归之后的时间复杂度就和动态规划相当了. 再将整个递归函数封装, 转变思路自底向上即可转换成动态规划.

按照正常的思路, 动态规划的解法都可以递归实现, 因此主问题的答案必定与子问题相关, 这也就是**最优子结构( Optimal Substructure )**; 同时, 如果递归中能够通过记录遍历过的状态而剪枝, 那说明其存在**重叠子问题( Overlapping Subproblem )**; 严格意义上, 动态规划适用于存在上述两种性质的问题.

因为一切程序都是有穷自动机, 那么我们可以从 DFA 的定义出发, 来写一个动态规划的模板

## DFA

有穷自动机形式化定义: 五元组$(Q, \Sigma, \delta, q_0, F)$

* $Q:$ 有穷集合, 状态集
* $\Sigma:$ 有穷集合, 字母表
* $\delta: Q\times\Sigma\rightarrow Q$ 转移函数
* $q_0\in Q:$ 起始状态
* $F\subseteq Q:$ 接收状态集

在 DP 问题中, 我们通常只需要考虑状态定义, 转移函数, 起始状态这三个条件即可, 对于字母表我们只需考虑其个数(也就是几维DP)

## DP

根据 DFA 的定义, 我们需要首先做出状态定义, 才能写出转移函数, 然而面对同一个问题, 不同的状态定义会写出不同的转移函数, 一个良好的状态定义会使得转移函数更加简练好写, 然而状态定义这个东西不太好总结. 一个可供参考的思路是:

* 通过终止状态来确定如何定义状态: 求什么, 就定义什么
* 通过字母表来进行定义: 看状态与那些变量有关, 全都列举出来做一个大定义(相当于你写了一个dp function)
* 根据经验定义
* 再通过数学归纳法检测定义是否合适(是否能通过前方已知状态推出未知状态).

简单来说, 可以直接将要求的结果作为定义(以终止状态定义), 不过遇到一些难一点的题目就需要换个方式定义了, 而这常常需要一定的经验(也可以说是积累)

> 但如果你无脑简单地把有关系的字母表都堆上去, 那可能时间复杂度很高且是不可优化的
{: .prompt-tip }

可以通过一个究极经典题举例 [Leetcode 300.最长递增子序列](https://leetcode-cn.com/problems/longest-increasing-subsequence/)

```c++
class Solution {
public:
    int lengthOfLIS(vector<int>& nums)
    {
        // 定义 dp[i]: 为以 nums[i] 结尾的最长递增长子序列的长度
        // 初始状态: 最短的就是它本身, 长度为 1
        vector<int> dp(nums.size(), 1);
        // 终止状态, 线性遍历整个数组结束
        for(int i = 0; i < nums.size(); ++i)
            // 遍历 dp[0:i] 找到满足 nums[i] > nums[j] 中的最长递增子序列长度再 + 1
            for(int j = 0; j < i; ++j)
                if(nums[i] > nums[j])
                    dp[i] = max(dp[i], dp[j] + 1);
        return *max_element(dp.begin(), dp.end());
    }
};
```

BTW, 针对上面这个问题有一个 $O(n\log(n))$ 的**耐心排序( Patience Sort )**方法能够完成上述算法, 我们将线性遍历整个数组, 在遍历过程中构建最大(小)堆, 如果无法构建, 则新开辟堆, 最终, 堆顶元素必定有序, 且每个堆也是有序, 其中最大的堆长度即为答案

```c++
class Solution {
public:
    int lengthOfLIS(vector<int>& nums)
    {
        vector<int> piles;
        for(int i = 0; i < nums.size(); ++i)
        {
            int cur = nums[i];
            // 二分查第一个堆顶元素不小于 cur 的堆
            int idx = lower_bound(piles.begin(), piles.end(), cur) - piles.begin();
            // 如果没有则新建堆
            if(idx >= piles.size())
                piles.push_back(cur);
            // 找到的堆更新堆顶元素
            piles[idx] = cur;
        }
        // 堆的数量就是最长递增子序列长度, 结果为各个堆顶组成的序列(也可根据堆求最长子序列的个数)
        return piles.size();
    }
};
```

我第一次遇到这道题的时候, 并不能做到快速定义状态, 至少凭借我的智力水平只能碰到一类记住一类

* 子序列/子串
  * 一维: dp[i] 代表以 nums[i] 结尾的序列
  * 二维: dp[i][j] 代表
    * 以 nums1[i], nums2[j] 结尾的两个序列
    * nums[i:j] 的一个序列

除开子序列的题目之外, 其他的就遇上一类总结一类来补好了
