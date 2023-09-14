---
layout: post
title: override & overload
date: 2023-08-11 17:29 +0800
tags: [C++]
categories: [C++]
author: Jebearssica
---

对于**重载**这个概念有点混淆, 进一步混淆了如标题所涉及的含义.

## 简介

应当先明确理解以下概念

* override: 重写, 只适用于继承关系下, 派生类对基类虚函数重写
* overload: 重载, 在相同作用域内, 拥有两个或以上的同名函数, 这些同名函数互相重载

### *override*

针对于编程中经常容易发生的覆盖虚函数的情况, 如下:

```c++
struct Base
{
    virtual void * get(int) { std::cout << "VERY UNIQUE OPERATION!"; };
}
struct Derived: public Base
{
    virtual void * get(itn) { std::cout << "I do not know, just very basic one"; };
};
```

通常引入两个问题:

* 实现派生类的时候并没有关注基类是否具备同名函数, 从而导致的无意覆盖. 甚至是不经意的 typo 导致的虚函数重构 (*overload*)
* 修改基类时删除虚函数, 导致子类的方法直接变为普通类方法

面对这两种很有可能破坏原有设计的行为, 可以通过 *override* 与 *final* 两个关键字来避免:

```c++
struct Base
{
    virtual void * get(int);
}
struct Derived: public Base
{
    virtual void * get(int) override; // 合法, 显式告知编译器重载基类的虚函数
    virtual void * get(itn) override; // 非法编译报错, 基类没有对应的虚函数
};
```

> 事实上, 上述例子中子类的 virtual 可以去除 (同样也是守则中提到的建议), 因为 *override* 已经说明了对应的函数继承的是虚函数
{: .prompt-info }

```c++
struct Base
{
    virtual void * get(int) final; // 显式告知编译器不允许子类重载
}
struct Derived1: public Base // 非法, 对应类为 final
{
};
struct Derived2: public Base
{
    virtual void * get(int) override; // 非法, 对应函数为 final
};
struct Derived3 final: public Base // 合法
{
};
```

### *overload*

最基本的多态表现, 但结合虚函数会有一些神奇的情况发生, 看如下示例:

```c++
struct Base
{
    virtual void * get(int);
}
struct Derived: public Base
{
    virtual void * get(itn);
};
```

首先不管是不是 typo, 编译肯定能过的, 但是会报一个 warning `-Woverloaded-virtual`. 其核心原理为, 派生类重载基类虚函数时, 原有基类的接口将被隐藏 (因为编译器在派生类中已经搜索到了同名的虚函数, 因此会直接俄停止向基类继续搜索). 因此我们需要显式声明是否保留原有基类的虚函数, 从而避免由于 typo 产生的重载.

```c++
// 屏蔽
private:
    using Base::get(int);
// 保留
public:
    using Base::get(int);
```

> 实际上, 我们不应该在派生类中试图新重载一个虚函数, 否则会破坏基类设计的语义. 既然都破坏了基类的语义, 为何不直接在基类中增加一个重载的实现? 但凡代码权限管理严格一点, 你根本无法改变基类语义, 建议直接子类新命名一个函数.
{: .prompt-warning }

## 特殊成员函数(special member function)的重载

特殊成员函数指的是编译器会在它们被使用时, 没有显式声明仍然能自动生成的函数, 情况如下:

* 默认构造函数: 当无其他构造函数被显式声明时
* 拷贝构造函数/拷贝赋值运算符: 当无移动构造函数或移动赋值运算符被显式声明时, **注意, 当析构函数被声明时, 拷贝构造函数在未来将不会自动生成**
* 移动构造函数: 当无拷贝构造函数或拷贝赋值运算符被显式声明时
* 析构函数: 当无拷贝构造函数或拷贝赋值运算符被显式声明时
