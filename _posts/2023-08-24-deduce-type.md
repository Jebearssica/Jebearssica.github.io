---
layout: post
title: Deduce Type
date: 2023-08-24 13:56 +0800
tags: [C++14, c++]
categories: [c++, C++14]
author: Jebearssica
---

可以算得上是 Morden Cpp 的基石, 实际应该在了解通用引用之后再熟悉类型推导的, 但我照着 Effective Mordern Cpp 学的, 所以就这样吧

## Template Type Deduction

示例代码如下, 我们将分析 `ParamType` 与 `T` 在不同实参传递情况下的类型推导结果

```c++
// define
template<typename T>
void f(ParamType param);
// call
f(expr);
```

### `ParamType` 为指针或非通用引用的引用

先上结论:

* `expr` 是引用则先忽略引用性(reference-ness)
* 然后进行类型匹配

举例如下

```c++
template<typename T>
void f(T& param);
int x = 0;
const int cx = x;
const int &rx = x;
f(x);  // T 等价于 int, param 为 int &
f(cx); // T 等价于 const int, param 为 const int &
f(rx); // T 等价于 const int, param 为 const int &
template<typename T>
void cf(const T& param);
cf(x);  // T 等价于 int, param 为 const int &
cf(cx); // T 等价于 int, param 为 const int &
cf(rx); // T 等价于 int, param 为 const int &
// 指针是一样的, 因此略过, 你把引用替换成指针即可
```

### `ParamType` 是一个通用引用

同样先上结论:

* `expr` 为左值时, `T` 与 `ParamType` 会被推导为左值引用
* `expr` 为右值时, 按照非通用引用的引用进行类型推导

举例如下:

```c++
template<typename T>
void f(T&& param);
int x = 0;
const int cx = x;
const int &rx = x;
f(x);  // T 等价于 int &, param 为 int &
f(cx); // T 等价于 const int &, param 为 const int &
f(rx); // T 等价于 const int &, param 为 const int &
f(10); // 10 为右值, T 等价于 int, param 为 int &&
```

- [x] 为何 10 没有被推导成 `const int`?
- A: 在类型推导中, 所有的常量性都会被忽略

### `ParamType` 非指针以及引用

结论: 按值传递, 即永远获得实参的拷贝.

* `expr` 为引用时先忽略引用性
* 再忽略 cv-qualified type(const volatile type qualifiers)

```c++
template<typename T>
void f(T param);
int x = 0;
const int cx = x;
const int &rx = x;
// 下列推导 T 与 ParamType 都是 int
f(x);
f(cx);
f(rx);
```

> cv-qualified type 被忽略可以这样理解. 参数传递的本身就是实参的一份拷贝构造, 因此对原实参的限定符本身对形参并没有任何影响. 同样应当注意的, cv-qualified type 当且仅在值传递时被忽略.
{: .prompt-tip }

对于指针而言, 情况略有不同, 常量性只在指针本身被忽略(因为拷贝), 指针指向的对象的常量性并不会被忽略.

```c++
const char* const ptr = "";
f(ptr); // T 与 ParamType 等价于 const char*
```

#### 数组实参

数组被允许退化为指向该数组的第一个元素的指针, 如下:

```c++
const char chararray[] = "";
const char *charptr = chararray;
```

因此在通过值传递的模板参数中进行类型推导, 数组类型会被推导为指针类型:

```c++
const char chararray[] = "";
f(chararray); // T 与 ParamType 等价于 const char*
// 因此如下两个声明会被视为等价
void f(char param[]);
void f(char *param);
```

只有在按一般引用传递时, 实参的数组类型会被传递:

```c++
template<typename T>
void f(T& param);
const char chararray[] = "";
f(chararray); // T 被推导为 const char [0], ParamType 为 const char (&)[0]
```

由于数组长度会通过推导被传递进函数, 因此可以写出如下的返回数组长度的模板函数

```c++
template<typename T, size_t N>
// 由于只关心数组大小, 因此形参无名
constexpr size_t size(T (&)[N]) noexcept {
  return N;
}
```

#### 函数实参

和数组一样, 函数同样被允许退化为指针类型, 事实上函数实参与数组实参在类型推导中的策略一模一样

```c++
void F(int, double);
template<typename T>
void f1(T param);
template<typename T>
void f2(T& param)
// call
f1(F); // ParamType 为 void(*)(int, double)
f2(F); // ParamType 为 void(&)(int, double)
```

## Auto Type Deduction

首先需要知道 `auto` 类型推导的实质形式如下:

```c++
// template type deduction
template<typename T>
void f(ParamType param);
f(expr);
// auto type deduction
template<typename T>
void f_deduce_type(ParamType param);
f_deduce_type(x); // 等价于 ParamType param = x;
```

`auto` 类型推导规则与 `template` 几乎一致, 其一的区别与统一初始化(uniform initialization)有关, `auto` 能够正确推导出 `initializer_list<>`, 而模板不能:

```c++
auto x1 = {1, 2, 3}; // auto -> std::initializer_list<int>
auto x2{1, 2, 3};    // auto -> std::initializer_list<int>
auto x3 = {1, 2, 3.0}; // compile error
template<typename T>
void f(T param);
f({1, 2, 3}); // compile error
template<typename T>
void f(std::initializer_list<T> param);
f({1, 2, 3}); // T -> int, ParamType -> std::initializer_list<int>
```

> 所以使用大括号进行初始化导致类型推导出问题, 这种问题应该很难排查吧? 但请不要因噎废食, 在详细了解 `std::initializer_list` 之后可能情况就不一样了.
{: prompt-tip }

其二的区别与函数返回值的推导有关, 此时 `auto` 按照模板类型推导的方式进行类型推导:

```c++
// 用于函数返回值
auto f() {
  return {1, 2, 3}; // compile error, since initializer_list can not be deducted by template
}
// 用于 lambda 函数形参类型推导
auto lambda_f = [&v](const auto &param) { v = param; };
lambda_f({1, 2, 3}); // compile error
```

总结:

* `auto` 一般情况下的类型推导与 template 中按值传递的推导方式一致
* 仅有在 `std::initializer_list` 与函数返回值或 lambda 形参中与 template 推导不一致

## Decltype

`decltype` 可以根据给定的表达式返回该表达式的类型, 在尾随返回类型中使用:

```c++
// C++11
template<typename Container, typename Index>
// 此时的 auto 并非类型推导, 而是触发函数尾随返回类型的声明
auto f(Container &c, Index idx) -> decltype(c[idx]) {
  return c[idx];
}
```

这使得返回值可以使用形参, 而传统语法使用形参时, 由于形参并未声明, 因此会报错. 在 C++14 中, 对于类型推导从单一 lambda 扩展至所有函数, 因此上述代码可以写成如下形式

```c++
// C++14
template<typename Container, typename Index>
auto f(Container &c, Index idx) {
  return c[idx];
}
```

此时 `auto` 类型推导为 `c[idx]` 的值类型, 然而一般的 stl 容器对于 `operator[]` 都会返回一个引用类型, 这在类型推导中会被忽略. 如果我们需要保留引用类型应当如何实现这个函数呢?

```c++
// C++14
template<typename Container, typename Index>
decltype(auto) f(Container &c, Index idx) {
  return c[idx];
}
```

> 事实上, 上面的声明可以通过通用引用来进行一些优化, 但我不想在这里展开通用引用
{: .prompt-info }

此时, `f` 的返回值与容器 `c` 的 `operator[]` 的返回类型完全一致, 即如果容器返回值则该函数返回值, 容器返回引用则该函数也返回引用.

同理, `decltype(auto)` 这样的用法也可用于变量的类型推导:

```c++
int x;
const int &crx = x;
auto ax = crx;           // ParamType -> int
decltype(auto) dax = crx;// ParamType -> const int &
```

> 接下来, 可能在实际中只有极少数地方会涉及, 可以视作奇技淫巧. 但下面的现象可能是一个大佬写出的代码也很有可能是一个纯菜鸟写出的代码.
{: .prompt-warning }

`decltype` 处理非单纯变量名的左值表达式时(这句话是人话吗?), 一定会推导出该类型的引用, 举例如下:

```c++
int x;
decltype(x);  // -> int
decltype((x));// -> int &
```

> 很显然, 一个简单的括号会产生非常严重的影响, 请千万不要小看这个! 这可能会使得一个函数返回一个临时变量的引用, 这通常是一个未定义的行为, 造成的问题会非常难以排查.
{: .prompt-danger }

```c++
// 演示一个极度危险的代码用例
decltype(auto) f() {
  int res;
  ...
  return res;
}
decltype(auto) danger_f() {
  int res;
  ...
  return (res);
}
```

总结:

* `decltype` 会产生变量或表达式的完整类型(包含 cv-qualifier)
* 对于非单纯变量名的左值表达式, `decltype` 总会推导出引用
* `decltype(auto)` 使用 `decltype` 的规则推导出一个类型, 可以像 `auto` 一样使用
