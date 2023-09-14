---
layout: post
title: bit-fileds and data alignment
date: 2023-08-17 17:58 +0800
tags: [C++]
categories: [C++]
author: Jebearssica
---

## 简介

> 位域的声明可以指定类或结构体中的数据成员存储位数, 通常用于节省不必要的空间开销. 相邻的成员可能被打包共享一个字节, 也可能被分开横跨多个字节. ~~真是晦涩难懂的定义呢~~
{: .prompt-info }

用法如下: 在所有对**非静态**成员变量的声明后加 `: <constexpr unsigned int>`

```c++
struct S {
    unsigned int a : 3; // 占 3 bits, 因此只能表示 [0, 2^3 - 1]
    unsigned int   : 2; // 占 2 bits, 未使用作为填充
    unsigned int b : 6; // 占 6 bits, 因此只能表示 [0, 2^6 - 1]
    unsigned int c : 5; // 占 5 bits, 因此只能表示 [0, 2^5 - 1]
    unsigned int d : 8; // 占 8 bits, 因此只能表示 [0, 2^8 - 1]
};
```

> 其中有一种特殊位域(无名位域)声明, 特点为不具备名称, 作用是指定一定位为不使用的位(作为填充位), 强制下一个位域在内存分配边界对齐.
{: .prompt-tip }

## 内存对齐

> 然而, 虽然声明了位域, 结构体实际占位并不能通过所有成员之和得到, 因为编译器会给出一定的填充占位 (padding) 从而满足**内存对齐**.
{: .prompt-tip }

> 这里编译器给出的填充占位与无名位域不同, 并非代码中显式声明的部分, 是编译器的一种优化.
{: .prompt-info }

这将从 CPU 结构讲起, 总所周知 (不知道的可以看计算机组成原理, ~~我猜的我真没看过~~), CPU 存在一个结构叫做总线 (bus). 通常而言, 总线获取数据不会逐个位获取 (by bit), 而是多字节获取 (by multiple bytes). 具体是多少个字节取决于总线位宽 (实际就是平台).

```c++
struct S {
    unsigned int a : 3;
    unsigned int b : 6;
    unsigned int c : 5;
    unsigned int d : 8;
};
```

内存分配情况如下:

* 首个双字节: a + b + c + 00, 其中末尾两位为编译器自行填充
* 第二个双字节: d + 00000000, 其中末尾八位为编译器自行填充

接着加入无名位域

```c++
struct S {
    unsigned int a : 3;
    unsigned int   : 2;
    unsigned int b : 6;
    unsigned int c : 5;
    unsigned int d : 8;
};
```

内存分配情况如下:

* 首个双字节: a + 00 + 000..., a 后两位为显式声明的占位, 其余的为编译器自行填充
* 第二个双字节: b + c + 000..., 其余的为编译器自行填充
* 第三个双字节: d + 000..., 其余的为编译器自行填充


## 参考文献

* [Objects and alignment - cppreference.com](https://en.cppreference.com/w/cpp/language/object)
* [C++ Bit Fields | Microsoft Learn](https://learn.microsoft.com/en-us/cpp/cpp/cpp-bit-fields)
* [Bit-field - cppreference.com](https://en.cppreference.com/w/cpp/language/bit_field)