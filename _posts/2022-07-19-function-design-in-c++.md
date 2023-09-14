---
layout: post
title: Function Design in C++
date: 2022-07-19 13:40 +0800
tags: [c++, design]
categories: [C++, Design]
author: Jebearssica
---

想到哪写到哪, 杂乱

## `const` 修饰成员函数

一个常见的情形如下:

```c++
// 声明一个 const 类, 并输出其数据
const T m1;
cout << m1.data();
```

一个容易令人忽视的地方在于, 成员函数 `member.data` 的设计会漏写 `const`, 从而导致类型无法转换的报错( 指针引用转为 const 指针 )

```c++
// 隐式参数 const member *this (成员函数自带)
T data() const { return this->data; }
```

> `const` 放在后面表示所有参数都被其修饰
{: .prompt-tip }

## 重载 `<<` 操作符的注意事项

* 一定是非成员函数

原因: 使用场景常见为 `cout << ...`, 与其他操作符不同, `<<` 作用在其左侧的对象中. 因此如果在成员对象中重载, 那么就需要如此调用(通过成员对象调用) `m1 << cout;` 极度诡异.

* 输入流不能被 `const` 修饰

原因: 所有操作交给 `cout` 时, 都对其进行了修改. 因此对该操作符进行重载时, 一定有如下声明

```c++
ostream& operator<< (ostream& os, ...)
```

* 返回值一定是 `cout` 对应类型的指针

原因: 使用场景 `cout << ... << ...`, 如果返回值为 `void`, 从右向左执行时, 会出现调用 `ostream& operator<< (void, ...)` 的情况, 显然与预期不符, 形参类型不匹配报错.

> 考虑到有许多情况下我们都会这样连续赋值, 因此针对 `=` 或 `+=` 之类类似的赋值操作符重载时, 都需要考虑返回值, 避免这样连续调用产生的错误
{: .prompt-tip }

## 含指针成员的拷贝赋值函数

* 针对 `=` 进行重载时, 如果类中含有指针成员, 那么一定要做自我赋值检测

原因: 两个相同的指针指向同一地址, 如果先删除后重新赋值会非法访问指针(原指针已被删除)

```c++
T &T::operator=(const T &addr)
{
    // 
    if (this != &addr)
    {
        delete[] data;
        data = new char[len(addr.data) + 1];
        copy(data, addr.data);
    }
    return *this;
}
```
