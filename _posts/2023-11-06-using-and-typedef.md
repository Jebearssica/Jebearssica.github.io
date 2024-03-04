---
layout: post
title: using and typedef
date: 2023-11-06 17:26 +0800
tags: [c++, C++14]
categories: [c++, C++14]
author: Jebearssica
---

> 在任何情况下, 都请使用 `using`
{: prompt-tip }

`using` 的使用方式与 `typedef` 一致, 可以帮助另取类型别名我们省略一大段的类型拼写(尤其在各种类嵌套的情况), 同时还能够增加代码的可读性. 一般的理解中, `using` 与 `typedef` 的作用完全一致, 仅仅存在语法上的不同, 如下:

```c++
typedef unstable_box_tree <box_type, vector_type, box_convert_type> box_tree_type;
using box_tree_type = unstable_box_tree <box_type, vector_type, box_convert_type>;
```

然而 `using` 能够做得更多, 对于别名模板 (alias templates) 来说, 相较于传统的 `typedef` 省略了大量代码

```c++
// using version
template <class T>
using myList = std::list<T, myAlloc<T>>;

myList<testClass> ml;
// typedef version
template <class T>
struct myList {
  typedef std::list<T, myAlloc<T>> type;
};

myList<testClass>::type ml;
```

> 针对依赖类型 (dependent type) 而言, 需要使用 `typename` 告知编译器一个未知标识符是一个类型.
{: prompt-tip }

```c++
template <class T>
class A {
private:
  typename T::type m_a; // treat T::type as a type
};

A<myList> a;
```

- [ ] 类型擦除：显式指出 compare function type，即可在外确定类型，function type 与 lambda 捕获对象无关
