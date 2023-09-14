---
layout: post
title: 'Keywords: const'
date: 2022-08-05 15:40 +0800
tags: [c++, design pattern]
categories: [C++]
author: Jebearssica
---

`const` 的作用: 用于修饰某某东西不变

## 使用 `const` 修饰成员函数

> `const` 关键字, 只有在成员函数中能有如下写法: `retureType functionName(argType &...args) const {}`. 即只有成员函数中能够使用上述的 `const` 用法, 来表明, 该成员函数不会对成员本身的数据进行修改
{: .prompt-info }

一个良好的设计习惯本身就要求, 对所有不会对类进行修改的成员函数使用上述 `const` 进行修饰. 然而, 在这里不仅仅是为了**更好**, 还是为了**不出错**.

> 非常量成员可以调用常量成员函数, 反之, 常量成员调用非常量成员函数会报错
{: .prompt-danger }

```shell
cannot convert 'this' pointer from 'const class 类名` to 'class 类名&'
```

> 额外信息: 成员函数同时存在非常量与常量两种版本时, 常量成员只能调用常量函数, 非常量成员只能调用非常量函数. 而这又给出了更多的信息, 即成员函数的签名包含了后方是否修饰的 `const`
{: .prompt-info }

我们可以考虑一下这样设计的目的, 其中 STL 中针对 `basic_string` 中有如下**重载**代码:

```c++
charT operator[](size_type pos) const
{
    // 省略实现
    // 无需考虑 COW: copy on write
}
reference operator[](size_type pos)
{
    // 省略实现
    // 必须考虑 COW: copy on write
}
```

> STL 中对 `basic_string` 的实现方式是引用计数( Reference-Counting ), 即多个相同的字符串共享同一个引用. 因此如果是一个被 `const` 修饰的静态变量, 对它的操作就无需考虑会对共享的引用进行修改的情况. 反之, 若不是一个静态变量, 则需要考虑**在写的时候拷贝( COW )**一个同样的引用, 从而避免对其他字符串的影响.
{: .prompt-info }
