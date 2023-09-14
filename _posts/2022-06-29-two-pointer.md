---
layout: post
title: Two Pointer
date: 2022-06-29 15:38 +0800
tags: [algorithm, two pointer, slide window]
categories: [Algorithm, Two Pointer]
author: Jebearssica
math: true
---

本来是一个掌握得挺好的一个知识点的, 但最近做题做下来的感觉总是在纠结当前区间内的元素个数. 那就总结一下这个知识点, 看看能不能通过梳理各种区间情况来让这类题写得更顺畅一点.

> 不知道有多少人和我一样会莫名其妙纠结, 区间 \[L, R\] 之间有多少元素? 再举个例子, 两个确定日期之间有多少天等等类似问题. 不过这种问题其实主要就是在是否清楚了解, 你所搜索的区间开闭情况
{: .prompt-tip }

## 双指针

一般来说, 双指针法是对暴力法的一种优化, 通常用于线性结构的问题中(如数组, 链表), 没什么高大上的主要思想, 就是靠着两个指针的滑动, 以及维护与双指针有关的信息(可以是双指针构成的区间, 也可以是前后缀等等). 由于双指针是一个非常广义的概念, 因此通常需要面对具体题目具体分析, 下面可以给出一些比较典型的例子

> 你以为给出的例子是纯纯的双指针吗? 那肯定是想多了, 有点难度的题目都是双指针结合一下其他方法构成一个题目, 单考一个双指针大概属于简单签到题?
{: .prompt-info }

### 区间信息

[Leetcode 719. 找出第 K 小的数对距离](https://leetcode.cn/problems/find-k-th-smallest-pair-distance/), 一道双指针结合二分的题目

分析一下题目吧, 别被题目里的 `0 <= i < j < nums.length` 这句话给骗了. 实际上只需要确保两个不同索引构成的距离就行了, 因此与原数组顺序无关, 可以直接排序后二分. 当然因为它标了是个困难题, 所以会看一看数据范围, 如果是面试手撕的话, 如果是我我会问数据范围, 笔试的话大部分也有数据范围. 本题我个人能给出的一个**最坏解**是暴力构造数对距离 + hash查重 + 排序或优先队列. 光是想想这个暴力解就不想写......

总之, 先按照上述思路, 二分范围是 \[0, 最大数对距离\], 可以给出以下二分"模板"代码

```c++
class Solution {
public:
    int smallestDistancePair(vector<int>& nums, int k)
    {
        sort(nums.begin(), nums.end());
        int left = 0, right = nums.back() - nums.front();
        while(left <= right)
        {
            int mid = left + (right - left) / 2;
            if(check(nums, mid, k))
                left = mid + 1;
            else
                right = mid - 1;
        }
        return left;
    }
};
```

很显然, `check` 用于根据二分结果 `mid` 判断是否是第 k 小的数对距离, 一个很容易想到的暴力法是, 针对每个确定的数对 `A` , 通过二分搜索不大于 `k-A` 的第二个数, 然后求和得到当前 `mid` 是第几小; 然而个人觉得此处用双指针实质上可能是更容易想到的方向, 针对每个确定的数 `B`, 收缩左侧边界直至双指针所指的数小于目标值, 则此时, 区间 \[left, right\)对应索引 right 构成的数对距离都小于目标值

```c++
bool check(vector<int> &nums, int target, int k)
{
    int res = 0;
    for(int left = 0, right = 0; right < nums.size(); ++right)
    {
        while(left < right && nums[right] - nums[left] > target)
            ++left;
        res += right - left;
    }
    return res < k;
}
```

如果熟悉滑动窗口的话, 会认为上述方法其实就是滑动窗口. 说的没错阿! 双指针和滑动窗口的关系在最后小结给出.

> "主动"纠结一下区间内元素个数吧: 在我确定元素 `B` 的情况下找元素 `A` 且要确保两者不同, 区间\[left, right\) 是满足元素 `B` 等于 `nums[right]` 所有元素 `A` 的可能索引. 因此最终确定的元素个数为 `(right - left)`
{: .prompt-tip }

### 序列信息

[Leetcode 524. 通过删除字母匹配到字典里最长单词](https://leetcode.cn/problems/longest-word-in-dictionary-through-deleting/), 一个典型双指针而非滑窗的题

很显然, 逐个判断时, 通过双指针判断两个序列是否匹配

```c++
class Solution
{
public:
    string findLongestWord(string s, vector<string> &dictionary)
    {
        string res;
        for (auto &word : dictionary)
        {
            int i = 0, j = 0;
            // LCS模板
            for (; i < s.size() && j < word.size(); ++i)
                if (s[i] == word[j])
                    ++j;
            if (j == word.size() && (res.size() < word.size() || (res.size() == word.size() && res > word)))
                res = word;
        }
        return res;
    }
};
```

> 上述代码存在一个 LCS (最长公共子序列( Longest Common Subsequence )) 的模板, 其实也就是这个地方用到了双指针. LCS 也是一个常见的考点.
{: .prompt-info }

### 单向链表中找环起点

典型的双指针中的快慢指针的用法, 具体操作就是一个慢指针每次移动至其 `next`, 另一个快指针每次移动至 `next->next`, 如果二者相遇, 则有环, 否则无环. 当然两者相遇后, 将快指针移至头结点, 两者同步移动至相遇地点即是环起点, 证明过程如下:

> 虽然我数学很烂, 但我一定要通过一个更烂的证明过程来证明我真的不是在摆烂——Jebearssica

$$
设, 环长为 C, 头结点至环起点为 L, 慢指针在环上移动了 m\\
Dis_{slow} = L + m, Dis_{fast} = L + n \times C + m, Dis_{slow} \times 2 = Dis_{fast}\\
\Rightarrow L + m = n \times C
$$

即, 从头结点至相遇点的距离, 为环的整数倍, 证毕

## 滑动窗口

相较于双指针, 我们更加注重左右指针构成的区间内的信息与性质, 并在移动指针的过程中去**维持性质不变**

### [Leetcode 2302. 统计得分小于 K 的子数组数目](https://leetcode.cn/problems/count-subarrays-with-score-less-than-k/)

直接给题上手, 虽然是 hard, 但它是一道**变换后**显而易见的滑动窗口题

> 重点当然是变换后了啊, 不然你为什么觉得它是 hard? hard 不就是这种套娃题嘛.
{: .prompt-warning }

分析一下定义中的数组分数, 就是当前数组元素和乘长度, 考虑一般情况下, 针对原始数组, 每个子数组的元素和如何快速求得. 显然这里运用前缀和能够快速求到, 那么问题就很简单了. 在预处理获得前缀和数组后, 维持一个窗口, 使得其代表的子数组的分数小于 k 即可(维持性质)

```c++
class Solution {
public:
    long long countSubarrays(vector<int>& nums, long long k)
    {
        // 前缀和处理
        vector<long long> prefix(nums.size() + 1, 0);
        for(int idx = 1; idx <= nums.size(); ++idx)
            prefix[idx] = prefix[idx - 1] + nums[idx - 1];
        long long res = 0;
        // 维持[left, right]窗口, 最外层移动right
        for(int left = 1, right = 1; right <= nums.size(); ++right)
        {
            // 维持[left, right]的性质
            while(left <= right && (prefix[right] - prefix[left - 1]) * (right - left + 1) >= k)
                ++left;
            // 得到最终的窗口信息, 根据题意进行统计处理
            res += right - (left - 1);
        }
        return res;
    }
};
```
