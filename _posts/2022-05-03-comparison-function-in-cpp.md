---
layout: post
title: Comparison Function in Cpp
date: 2022-05-03 13:52 +0800
tags: [c++, stl]
categories: [C++, STL]
author: Jebearssica
---

[Leetcode 937. 重新排列日志文件](https://leetcode-cn.com/problems/reorder-data-in-log-files/), 一个简单题但是可以好好品一下 STL sort 里的 Comparison Function

题目很简单, 照着模拟即可, 主要考点就是让你写一个自定义比较函数. 对于 Comparison Function, 我总是很迷惑, 什么时候该写小于, 什么时候该写大于, 什么时候该总是返回 true, 而这又意味着什么.

```c++
class Solution {
public:
    vector<string> reorderLogFiles(vector<string>& logs)
    {
        // 不用 sort 是因为要保持原序
        stable_sort(
            logs.begin(),
            logs.end(),
            [&](const string &a, const string &b)
            {
                int s1 = a.find(' '), s2 = b.find(' ');
                if(isdigit(a[s1 + 1]) && isdigit(b[s2 + 1]))
                    return false;
                else if(isdigit(a[s1 + 1]))
                    return false;
                else if(isdigit(b[s2 + 1]))
                    return true;
                else
                {
                    string content1 = a.substr(s1), content2 = b.substr(s2);
                    if(content1 != content2)
                        return content1 < content2;
                    return a < b;
                }
            }
        );
        return logs;
    }
};
```

然而, 一切疑惑都可以根据这句话解决

> Sorts the elements in the range \[first, last\) in **non-descending order**. —— from [cppreference](https://en.cppreference.com/w/cpp/algorithm/sort); Comparison function object (i.e. an object that satisfies the requirements of Compare) which returns ​true if the first argument is **less than** (i.e. is ordered before) the second.
{: .prompt-tip }

它排序是试图排出一个不递减(非严格递增)的序列, 你的 Comparison Function 就是比较规则, 返回 true 则说明首个元素在前, 那么我们就可以给上述代码写上注释了

```c++
// 如果都是数字序列, 保持原序, 返回 false(相等 == is not less than)
if(isdigit(a[s1 + 1]) && isdigit(b[s2 + 1]))
    return false;
// 只有 a 为数字序列, 字母序列 b 应该在前, 即字母序列永远小于数字序列(返回 false)
else if(isdigit(a[s1 + 1]))
    return false;
// 只有 b 为数字序列, 字母序列 a 应该在前, 即字母序列永远小于数字序列(返回 true)
else if(isdigit(b[s2 + 1]))
    return true;
else
{
    string content1 = a.substr(s1), content2 = b.substr(s2);
    // 内容不同, 比较两者大小(字典序), 为 true 则说明 content1 更小
    if(content1 != content2)
        return content1 < content2;
    // 内容相同, 比较 tag 大小(字典序)
    return a < b;
}
```
