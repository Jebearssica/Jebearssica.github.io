---
layout: post
title: value category
date: 2023-09-27 15:46 +0800
tags: [c++, Mordern C++]
categories: [c++, Mordern C++]
author: Jebearssica
---

C++ 中的任意表达式都有一个类型(type), 以及值类型(value category). 值类型是理解编译器在表达式计算过程中针对临时变量的创建, 复制和移动所必须遵循的规则的基础.

* glvalue: 一个表达式能够指定一个函数或对象
* prvalue: 一个表达式的计算能够初始化对象或位域, 或是一个内置操作符的计算结果
* xvalue: 是一个能够指定一个对象的资源被复用的 glvalue
* lvalue: 是一个非 xvalue 的 glvalue. ~~历史遗留的含义为赋值符号左侧的值类型~~大清早没了还在看历史
* rvalue: prvalue 或 xvalue

说点人话:

* lvalue: 是一个程序可访问的地址, [cppreference](https://en.cppreference.com/w/cpp/language/value_category) 列举了一屏幕的例子
  * 成员为左值的情况下, 其内部成员类型为左值(member access)
  * 进行强制类型转换为左值的表达式
* prvalue: 程序不可访问, 比如常量(除开字符串常量, 字符串常量是 lvalue, 微软文档没特别指出这个), 或是返回非引用类型的函数调用等等
  * 成员为非静态类型或枚举类型的成员变量
* xvalue: 程序不可访问, 但能够用来初始化右值引用, 比如数组的索引等等
  * 其设计原则是允许 lvalue 和 prvalue 之间进行类型转换, 即 lvalue cast to xvalue to init rvalue

人话说不通, 感觉还是得实用主义优先, 先从使用场景着手


## 临时量实质化 temporary materiliation

针对一个临时变量在栈上进行实质化, 属于一种隐式类型转换.

可以避免针对临时变量的构造与拷贝构造, 转而进行一个某个值类型的拷贝初始化 ==> copy initialization

## 移动语义

```c++
struct test {

}
test move() {
    reutrn test{};
}
int main() {
    auto test_var = move();
    return 0;
}
```

## reference initialization

highly related with std::move and std::bind

## member function overload with ref-qualifier

- [ ] 强制拷贝消除