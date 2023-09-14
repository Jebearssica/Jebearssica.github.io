---
layout: post
title: 'Keywords: static'
date: 2022-07-23 22:11 +0800
tags: [c++, design pattern]
categories: [C++]
author: Jebearssica
---

## static member data

在变量名前加 `static` 关键字, 使得该变量变为静态变量. 变量存储在全局数据区而非堆栈, 不会随着所在域(函数或成员)的消亡而消亡.

```c++
// 常见于 Leetcode 题目中用于取余的值
static constexpr int MOD = 1e9 + 7;
```

若在成员变量前修饰, 则该成员变量在全局域中只有一份. 应用场景通常为多个成员共享同一份数据. 如 商品成员共享商品税.

```c++
class product
{
public:
    static double tax;
};
```

静态成员变量与普通成员变量调用方式有显著不同. 普通成员变量根据各自成员的地址(指针)进行调用, 而静态成员变量地址单独存放, 因此可以通过类域访问.

```c++
// 在无product成员时进行定义
double product::tax = 0.15
```

## static member function

静态成员函数, 常用于修改静态成员变量. 调用可以通过成员本身调用, 也可通过类域调用.

```c++
class product
{
public:
    static double tax;
    static void set_tax(double t) { tax = t; }
};
// 两种调用方法
product::set_tax(0.3);
product pencil;
pencil.set_tax(0.14);
```

## static member: Singleton

运用上述知识, 我们可以复现单例模式( Singleton ), 它具有以下特点:

* 只有一个实例(实体)对象
* 对象由对象本身创建
* 该对象对外提供一个访问该对象的全局访问点

> 全局访问点实际上就是 static member function
{: .prompt-tip }

```c++
class Singleton
{
private:
    Singleton();
    Singleton(const Singleton &s);
    static Singleton s;

public:
    // 通过 static member function 访问 static member data
    static Singleton &getInstance() { return s; }
};
```

将构造函数放入 `private` 区, 确保第二个特点, 因为只有对象本身能够调用构造函数创建自身. 然而上述写法有所缺陷, 当没人用到 `Singleton` 时, 由于静态成员变量的存在, 它依旧占据了空间. 那么, 我们可以在静态成员函数中, 构造静态变量. 因此我们可以写出下述代码

```c++
class Singleton
{
private:
    Singleton();
    Singleton(const Singleton &s);

public:
    // 通过 static member function 构造 static member data 并返回
    // 因此只有在需要访问实体时, 才会被创建并存在
    static Singleton &getInstance()
    {
        static Singleton s;
        return s;
    }
};
```

优点自然不必多说, 节省内存占用, 优化对资源多重占用. 然而写的过程中其实就能明显感受到其缺点:

* 扩展困难: 接口少难扩展, 一动就要删除原有代码, 迭代困难
* 并发测试困难: 全局只有一个实体, 调试过程中单例未运行完毕则不可产生新对象
* 设计困难: 代码通常集中在一个类上
