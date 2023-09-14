---
layout: post
title: Josephus Problem
date: 2022-05-04 09:42 +0800
tags: [algorithm, josephus problem]
categories: [Algorithm, Josephus Problem]
author: Jebearssica
math: true
---

[Leetcode 1823. 找出游戏的获胜者](https://leetcode-cn.com/problems/find-the-winner-of-the-circular-game/), 经典约瑟夫问题, 大致意思是: "N个人围成一圈, 第一个人从1开始报数, 报M的将被杀掉, 下一个人接着从1开始报. 如此反复, 最后剩下一个, 求最后的胜利者". 如果知道递归公式就可以在 $O(N)$ 的时间内完成计算.

## 队列模拟

队首出列进队尾, 重复 k - 1 次后取出队首, 时间复杂度 $O(NK)$, 要是两者乘积小于 1e9 的话应该都是能过的.

```c++
class Solution {
public:
    int findTheWinner(int n, int k)
    {
        queue<int> q;
        for(int idx = 1; idx <= n; ++idx)
            q.push(idx);
        while(q.size() > 1)
        {
            for(int idx = 1; idx < k; ++k)
                q.push(q.front()), q.pop();
            q.pop();
        }
        return q.front();
    }
};
```

## 递归公式

我们可以根据函数写出这样的递归形式, 即每次递归问题规模减一, 把数组看成循环数组, 每个点的索引值相较于上一轮都前进了 k.

```c++
class Solution {
public:
    int findTheWinner(int n, int k)
    {
        if(n == 1)
            return 1;
        int res = (findTheWinner(n - 1, k) + k) % n;
        return res == 0 ? n : res;
    }
};
```

写出递归式: $F(n, k) = (F(n - 1, k) + k) \% n; F(1, k) = 1$

```c++
class Solution {
public:
    int findTheWinner(int n, int k)
    {
        // 从 0 开始方便取模
        int pos = 0;
        for(int idx = 2; idx <= n; ++idx)
            pos = (pos + k) % idx;
        return pos + 1;
    }
};
```
