---
layout: post
title: 'Keywords: auto'
date: 2023-10-08 14:26 +0800
tags: [c++, C++14]
categories: [c++, C++14]
author: Jebearssica
---

通常而言, `auto` 能够根据类型推导规则推导出你想要的类型. 然而, 由于推到规则的限制, 部分情况下可能需要引导编译器推导出你想要的类型.

## prefer `auto` to explicit type declaration

这是一个建议, 事实上众多自带类型推导的语言同样承担了很多大型项目, 这一点几乎就可以确定显式类型推导并不一定是一个大型项目的必要条件. 一个良好的变量命名即可解决这些问题.

## Use the explicitly typed initializer idiom when `auto` deduce undesired type

学到了, C++ 不支持 bit 的引用, 因此诸如下面的代码类型推导会通常不如所愿

```c++
std::vector<bool> test ();
// auto -> std::vector<bool>::reference
auto element = test()[5];
```

> 事实上, `std::vector<bool>` 更接近于一个动态的 `std::bitset`. 后者也有一个类似的不可见的代理类 `std::bitset::reference`
{: prompt-tip }

通常有以下两种解决方案, 这里更加推崇第二种, 它能够更加显著告诉其他人你的语义.

```c++
// 一个常见的解决方案
bool element = test()[5];
// 一个更可读的解决方案, 这样写你摆明了告诉后来的人, 你确确实实知道 test() 的返回值并且你就是想要一个 bool
auto element = static_cast<bool>(test()[5]);
```
