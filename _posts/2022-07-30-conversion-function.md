---
layout: post
title: Conversion Function
date: 2022-07-30 10:50 +0800
tags: [c++]
categories: [C++]
author: Jebearssica
---

## 简介与基础知识

转换函数常用于将一个类转换为另一个类, 具有与一般函数不同的声明注意事项, demo 如下:

```c++
class frac
{
private:
    int numer;
    int deno;
public:
    operator double() const { return (double)(numer / deno); }
    frac(int nu, int de = 1) : numer(nu), deno(de) {}
};
```

* 无返回值:

此时, 针对 `double` 的重载, 与[针对 `<<` 操作符的重载]({% link _posts/2022-07-19-function-design-in-cpp.md %}#重载--操作符的注意事项)不同, 声明时**无返回值无形参**

你可以通过如下形式来实现任意转换函数:

```c++
// 将 anotherClass -> myclass
anotherClass::operator myclass() { ... }
```

* 调用方法:

```c++
frac f(3, 5);
double res = 4 + f;
```

首先, 编译器先寻找有无 `operator +(int, frac)` 的重载, 如果没有, 再寻找有无 `frac::operator double()` 的重载

## 非显式含参构造函数( Non-explicit-argument constructor )

转换函数同样能够通过非显式的形式, 调用**含(实)参**的构造函数来完成. 前文 demo 的构造函数有了一个默认参数 `deno`, 那么在如下情况

```c++
// 重载 +
frac frac::operator+(const frac &f)
{
    ...
}
frac f(3, 5);
// 4 调用 frac(int nu, int de = 1), 其中 4 作为实参传递给 nu
frac res = f + 4;
```

编译器先调用 `frac::operator+`, 然后试图将 4 转换为 `frac`, 从而调用构造函数 `frac::frac(int nu, int de = 1)`

> 歧义( Ambiguous )警告! 当同时声明 `operator double` 与 `frac(int nu, int de = 1)` 并重写加法操作符后. 调用 `frac res = f + 4;` 这段代码就会产生二义性. 编译器无法确定, 将 4 转换为 `frac` 之后做加法 亦或是 将 `f` 转为 `double` 做加法后再转换成 `frac`
{: .prompt-danger }
> 二义性的产生不仅仅需要通过设计来避免, 也应当考虑使用者的使用场景
{: .prompt-info }

## 显式含参构造函数( Explicit-argument constructor )

显式加上 `explicit` 关键字, 可以使得构造函数只能被显式调用

> 这是 `explicit` 关键词几乎所有的应用场景, 还有一咩咩在模板上, 但太少见. 所以可以说这是 `explicit` 的唯一作用: 使得构造函数只能被显式调用
{: .prompt-tip }

```c++
class frac
{
private:
    int numer;
    int deno;
public:
    operator double() const { return (double)(numer / deno); }
    explicit frac(int nu, int de = 1) : numer(nu), deno(de) {}
};
```

因此, 此时通过下述方法, 试图阻止隐式调用构造函数进行转换. 因此编译器只能去找 `operator double()`, 而未找到定义则报错.

```c++
frac f(3, 5);
// error: conversion from double to frac requested
frac res = f + 4;
```
