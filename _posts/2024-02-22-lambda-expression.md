---
layout: post
title: Lambda Expression
date: 2024-02-22 09:43 +0800
tags: [c++]
categories: [C++]
author: Jebearssica
---

C++11 之后提供了一种方便实现匿名函数对象的方法, lambda. 然而在写树上 dp 时, 常常需要实现一个递归的 lambda 函数, 但我又很不想写完整的 `function` 类型, 此时涉及一些类型推导的事情也一并在本文描述(搞得好像我本来就知道一样).

## 基础用法

一个 lambda 表达式由以下部分构成:

* capture: 描述变量如何捕获
* param: 描述该函数需要显式传递的变量, 可选的
* mutable/exception: 描述 mutable 以及异常抛出的修饰符, 可选的
* trailing return type: 描述返回类型, 可选的
* body: lambda 函数主体部分

### Capture

每个 lambda 函数不仅可以访问传递进函数体的变量, 还能捕获其所在空间的其他变量, 是 lambda 函数的必备选项. 其传递方式根据 `[]` 内的符号决定:

* `[]`: 默认表示在 lambda 闭包内不访问任何其他变量
* `[&]`: 表示 lambda 通过引用的方式访问其他变量
* `[=]`: 表示 lambda 通过值传递的方式访问其他变量

当然你也可以具体指定哪些变量通过引用传递, 哪些变量通过值传递, 如下

```c++
int a, b;
// the below 4 cases are equivalent
auto lam1 = [&a, b] {}; // a by reference, while b by value
auto lam2 = [b, &a] {}; // same as above
auto lam3 = [&, b] {};  // default by reference, while b by value
auto lam4 = [=, &a] {}; // default by value, while a by reference
```

当我们声明了默认捕获方式时, 不能再后续对特定变量声明相同的捕获方式, 即, 我们设置了 `&`, 则后续捕获方式不允许再出现`&param`(即, `[&, &param]`); 设置了 `=`, 则后续不允许出现 `=param`(即, `[=, param]`), 如下

```c++
// assume we are in a class's namespace
int i;
[&, i] {};      // correct, i captured by reference
[&, &i] {};     // error, by-reference capture when by-reference is the default
[=, *this] {};  // correct, capture this by value, since C++17
[=, this] {};   // error, we assume = is default, util C++20
[i, i] {};      // error, i repeated
[this, *this] {}; // error, this repeated
[i](int i) {};  // error, paramter and capture have same name
```

如果出现 `...`, 则会将其视作一个包, 常见于可变参的模板中:

```c++
template <class... Args>
void func(Args... args) {
  auto f1 = [args...] { return f2(args...); };
  f1();
}
```

> 注意, 如果需要在类成员函数中使用 lambda, 需要将 `this` 传递给 lambda capture 从而能够在 lambda 主体内获得访问类成员的权限. 同时应该注意, 如果规定为值捕获 `[=, *this]`, 那么每次函数调用都会拷贝一整个对象, 对于一些需要并行化或异步处理的场景下, 值拷贝是一个很好的避免数据同步问题的解决方案.
{: prompt-info }

### Parameter

lambda 函数处理捕获外部变量, 还可以通过参数列表进行传参, 与一般的函数传参一样. 同样的, 当传入的参数是一个泛型时, 我们可以使用 `auto` 省略其类型. 进一步, 由于 lambda 函数本身是一个对象, 因此我们可以比较方便的实现 lambda 函数的递归嵌套. 如下

```c++
// 传统的 lambda 递归写法, 需要明确类型, 非常麻烦
auto nth_fib = [](int n) {
  std::function<int(int, int, int)>
};
```

### Return

lambda 函数的返回类型会自动推导. 如果主体中不存在返回语句, 那么返回类型会自动推导为 `void`.

### Body

lambda 函数主体部分与普通函数或成员函数的主体部分一致, 对于以下数据拥有访问权

* 捕获的变量
* 传递的参数
* 函数主体局部声明的变量
* 如果定义在类成员内部, 且 `this` 是捕获的变量, 则可访问类成员
* 静态变量
