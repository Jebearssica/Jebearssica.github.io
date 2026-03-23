---
layout: post
title: Monotonous Stack
date: 2022-08-20 19:18 +0800
tags: [data structure, monotonous stack]
categories: [Data Structure, Monotonous Stack]
author: Jebearssica
math: true
---

单调栈就是有单调性质的栈, 与之相似的还有单调队列, 主要解决区间最值问题( RMQ: Range Maximum/Minimum Query ). 单调栈能够维护一个数前/后一个, 大于/小于它的数.

以下给出根据一个单调递增栈的实现, 栈中存储数组下标.

```c++
stack<int> constructMonoStack(const vector<int> &nums)
{
    stack<int> monoStk;
    for(int idx = 0; idx < nums.size(); ++idx)
    {
        // 这里比较可以改成单调递减栈
        while(!monoStk.empty() && nums[monoStk.top()] < nums[idx])
            monoStk.pop();
        monoStk.push(idx);
    }
    return monoStk;
}
```

实际应用需要你能够将问题抽象为区间最值问题, 这里给出一些示例

## [Leetcode 654. 最大二叉树](https://leetcode.cn/problems/maximum-binary-tree/)

按照题意易得出一个递归构造, 时间复杂度为 $O(N^2)$ 的解法. 然而仔细分析后可以发现有如下规律:

* 在一个区间内的最大值为根结点, 其父亲结点必定是左右两侧第一个比自己大的值
  * 进一步: 一定是上述俩者中的较小的一个, 较大的一个
