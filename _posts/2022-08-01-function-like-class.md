---
layout: post
title: Function-Like Class
date: 2022-08-01 20:02 +0800
tags: [c++]
categories: [C++]
author: Jebearssica
---

仿函数( Functor ), 就是一个像函数的类, 其中必须重载实现 `operator()` ( operator call ), 以下用 STL 中的 `less` 举例.

```c++
template< class Arg1, class Arg2, class Result>
struct binary_function
{
    typedef Arg1 first_argument_type;
    typedef Arg2 second_argument_type;
    typedef Result result_type;
};
template <class T>
struct less : public binary_funtion<T, T, bool>
{
    bool operator()(const T &x, const T &y) const { return x < y; }
};
```

`less` 的使用场景自不必多说, 用到比较的各种结构或算法都经常见( 如 `priority_queue`). 值得注意的是, 我们可以分析上述两个结构体的大小, 理论上来说大小为 0( 实际 `sizeof(binary_function) == 1 )

通用上来说, 仿函数的作用非常多, 如安全传递函数指针, 依靠函数生成对象, 代码复用, 类间继承关系等等, 属于是一个非常重要, 但又十分难理论分析整体总结的知识点. 大概是我目前这种"纸上谈兵"阶段的痛点, 可能需要在日后生产实践中学习吧.
