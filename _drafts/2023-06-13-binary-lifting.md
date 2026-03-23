---
layout: post
title: Binary Lifting
date: 2023-06-13 20:17 +0800
tags: [algorithm, binary-lifting]
categories: [Algorithm]
author: Jebearssica
---

起源, [Leetcode 1483. 树节点的第 K 个祖先](https://leetcode.cn/problems/kth-ancestor-of-a-tree-node/description/). 时隔十个月重新做题, 屁都不会, 差不多又快回到草履虫时期了. (事实上是这道题确实需要额外的知识储备)

> BTW, 工作之后了解到现如今中文互联网的语料过少, 可能使得未来 NLP 远离我们. ~~所以我打算在个人博客上增加更多废话~~. 其实需要的是一个开放专业的讨论氛围, 而不是我这种菜鸟放弃碎碎念板着脸一本正经写些垃圾就能变好的.
{ .prompt-tip }

## 简介

一种空间换时间的算法, 通过预处理使得后续操作的时间复杂度大幅度下降. 主要的核心思路可以视作二进制编码, 将一个线性的处理步骤给拆分成多个二进制的步骤. 即, 任意常数 n 可以分解为若干个不同的二次幂.

> 这种思路是不是有点太宽泛了? 什么状态压缩 dp, 各种位运算技巧似乎都可以是这种核心思路
{ .prompt-info }

## 最近公共祖先 (LCA)

朴素算法是以 $O(n)$ 的复杂度完成 n 次当前结点的祖先结点的查询, 而使用倍增思想时, 可以通过预处理每个点的第 $2^k$ 个祖先的位置

更详细点, 可以认为 `f[x][i]` 为结点 `x` 的第 $2^i$ 个祖先结点, 那么有如下转移方程: `f[x][i] = f[f[x][i-1]][i-1]`, 即一个结点 `x` 的第 $2^i$ 个祖先结点为该结点的 第 $2^{i-1}$ 个祖先结点的第 $2^{i-1}$ 个祖先结点

> 事实上是借助倍增思想从而实现一种特殊的动态规划来完成 LCA 的求解
{ .prompt-tip }

```c++
class TreeAncestor {
public:
    TreeAncestor(int n, vector<int>& parent) {

    }
    
    int getKthAncestor(int node, int k) {

    }
};

/**
 * Your TreeAncestor object will be instantiated and called as such:
 * TreeAncestor* obj = new TreeAncestor(n, parent);
 * int param_1 = obj->getKthAncestor(node,k);
 */
```
