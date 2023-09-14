---
layout: post
title: Prime Sieve
date: 2022-06-30 10:11 +0800
tags: [algorithm, sieve]
categories: [Algorithm, Sieve]
author: Jebearssica
math: true
---

还在苦恼于面试中只能写出 $O(n\sqrt{n})$ 时间复杂度的素数筛选吗, 快来接触埃氏筛吧

## 暴力枚举

顾名思义, 简单粗暴, 不过要是真写出一个纯暴力 $O(n^2)$, 甚至没用平方根的技巧的话, 那是真的会凉凉

```c++
int sieve(int n)
{
    int res = 0;
    for(int cur = 2; cur <= n; ++cur)
    {
        bool isPrime = true;
        for(int j = 2; j * j <= cur; ++j)
        {
            if(cur % j == 0)
            {
                isPrime = false;
                break;
            }
        }
        res += isPrime;
    }
    return res;
}
```

## 埃氏筛(埃拉托斯特尼筛( Eratosthenes ))

考虑到数之间的关联性, 一个显而易见的事实: 一个正整数的 $x, x > 1$ 的倍数一定是合数, 那么我们可以在筛选至一个数时, 标记其所有倍数, 从而减少筛选次数

```c++
int Eratosthenes(int n)
{
    vector<bool> isPrime(n + 5, true);
    int res = 0;
    for(int cur = 2; cur <= n; ++cur)
    {
        if(isPrime[cur])
        {
            ++res;
            // check validation
            if((long long)cur * cur <= n)
            {
                for(int j = cur * cur; j <= n; j += cur)
                    isPrime[j] = false;
            }
        }
    }
    return res;
}
```

> 上述是经过一定优化后的埃氏筛, 具体在第二重循环中, 从 `cur * cur` 处开始判断, 因为之前的倍数已经被标记过了
{: .prompt-tip }

严格的时间复杂度为 $O(n\log{\log{n}})$, 具体证明过程可参考[OI WIKI](http://oi-wiki.com/math/number-theory/sieve/#_2), 需要运用到一些数论知识, 然而我并不具备XD, ~~但可以现学嘛~~, 现学失败, 需要一定复变函数的知识, 我没学过

还有一个线性筛, 能够实现线性时间复杂度的素数筛选, 能够使得每个合数都只会被标记一次(埃氏筛中仍然有重复标记的情况, 如45: 3 5), 然而其更广泛用于积性函数相关的问题上, 很显然我觉得到此为止差不多了
