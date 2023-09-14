---
layout: post
title: Memory Claim and Reclaim
date: 2022-07-19 20:01 +0800
tags: [c++]
categories: [C++]
author: Jebearssica
---

Java自带一套 Memory Collection, C++ 没有, 了解一下内存分配与回收还是挺重要的. 毕竟旁观室友面 Java 岗, 真的是一直问这个.

## 基础知识

* 局部对象(local member), 内存在栈上, 随着作用域消失而消失回收
* 全局对象(static member), 内存在堆上, 随着整个程序结束而消失回收

## 内存分配

内存分配过程如下, 总是先分配内存空间后调用构造函数

```c++
T *str = new T();
// 实际 c++ 过程: 分配内存空间->类型转换->调用构造
void* mem = operator new(sizeof(T));
str = static_cast<T*>(mem);
str->T::T();
```

- [ ] 有关 `void*` 似乎可以额外开新章学习

### 分配内存空间

计算所需分配内存的空间. 操作系统分配内存块, 将块首尾( `cookies`(首尾的 `cookies` 各占 4 个字节) )的最后一位 bit 置为 1, 表示这块内存已被分配. 最后将块总占用填补至16字节的倍数.

> 由于每个块的大小都是 16 倍数, 因此 `cookies` 的最后一位总为0. 此外 `cookies` 还记录了当前块的大小, 从第二位开始代表当前块一共有多少个 16 字节. 另外在调试模式下, 内存会在上下文分配更多空间. 具体的空间占用与编译器有关.
{: .prompt-info }

## 内存回收

内存回收过程如下, 总是先调用解析函数后回收内存

```c++
delete str;
// 实际 c++ 过程: 调用解析->回收指针内存
T::~T(str);
operator delete(str);
```

### 数组内存声明与回收(Array New & Array Delete)

上述举例的是一般指针的声明与回收, 这里讨论一下数组声明与回收

```c++
T *arr = new T[size];
delete[] arr;
```

> 数组声明与回收必须配套, 否则可能造成内存泄漏(memory leak)
{: .propt-danger }

至于为什么是**可能**, 我们需要详细了解数组内存如何分配. 数组内存分配与前文所述略有不同, 除开多个成员占据的空间之外, 还**包含**一个4字节块( 指针长度 )存放数组长度. 调用 `delete[] arr` 时, 会根据数组长度调用对应次数的析构函数.

当成员不包含指针时, 直接 `delete arr` 可以正常释放内存. 然而存在指针时, 由于只调用了一次析构函数, 剩下的数组成员未调用析构函数, 造成内存泄漏.

> 有关 `sizeof`: 针对一个普通的实例, 它的大小与类的成员有关, 以及是否存在虚函数( 存在即多一个指针大小, `vptr` ). 针对一个数组实例, 在上述考虑普通实例大小以及数组元素个数的情况下, 还额外有一个指针大小存放当前数组元素长度.
{: .prompt-info }

## 重载 `operator new` 与 `operator delete` 操作符

> 注意, 我们重载的是操作符 `new` 与 `delete`. 而非 C++ 中的关键字 `new` 与 `delete`(这个是无法重载的). C++ 的关键字 `new` 与 `delete` 会调用 `operator new` 与 `operator delete`, 也就是上文内存分配与回收的过程
{: .prompt-warning }

有两种方式进行重载, 分别是成员函数重载以及全局函数重载, 以下给出成员函数重载的声明, 同理也可写出同样的操作符重载

> 此处针对操作符的重载用 `static` 进行修饰, 是因为针对所有的类的实例, 都只需要通过类名调用同一个重载函数即可. (与类中数据无关)
{: .prompt-tip }

```c++
class myClass
{
public:
    // 必须给定需要 new 的大小
    static void* operator new(size_t);
    static void* operator new[](size_t);
    // 必须给定需要回收的指针, 可选择的给定回收大小
    static void operator delete(void*, size_t);
    static void operator delete[](void*, size_t);
};
```

> **注意!** 一定要十分小心全局域下的 `new()` 与 `delete()` 的重载. 它的影响十分广泛, 一定要再三考虑其可能产生的后果! 因为对一个成员使用 `new` 或 `delete`, 当其成员函数未进行重载时, 就会调用默认的全局的重载.
{: .prompt-danger }

如果使用者需要绕过已重载的 `new` 和 `delete`, 可以使用如下代码来强制使用全局域的 `new` 与 `delete`

```c++
// 有成员重载优先使用成员重载, 否则使用全局重载
myClass *myclassPtr = new myClass;
delete myclassPtr;
// 强制优先使用全局重载
myClass *myclassPtr = ::new myClass;
::delete myclassPtr;
```

### Placement New & Placement Delete

Placement new 可以说是 `operator new` 的一个细分. 它与原版 `operator new` 的唯一区别就是, 它允许在一块已经分配的内存上创建对象. 因此, 这个常常用于内存池的构造, 在已经开辟的内存池空间中, 去创建新对象. 防止每次 `new` 都要单独申请空间, 造成空间碎片耗时高. 内存池常常用于长时间运行, 对时间敏感的场景.

Placement new 重载时第一个参数必须是 `size_t`. 否则编译有以下报错信息:

```shell
[Error] 'operator new' takes type 'size_t`('unsigned int') as first parameter [-fperssive]
```

Placement delete 被重载后, 不会直接被关键字 `delete` 调用, 只有当通过 `new` 调用的构造函数异常抛出时, 才会调用重载的 placement delete. 它通常用于返还未完全创建的对象所占用的空间( 因为先分配内存, 后调用构造函数 ), 即进行异常处理.

用 STL 中的 `basic_string` 举例, 在[vptr-vtbl](https://jebearssica.github.io/posts/Keywords-const/#简介与基础知识)中, 提到了 STL 中的 `string` 设计, 存在一种 reference counting 机制.

```c++
class basic_string
{
    struct Rep
    {
        // declare
        // 如有需要你还可以声明拥有更多形参的重载, 只需要保证首个类型为 size_t
        inline static void* operator new(size_t s, size_t extra)
        {
            return Allocator::allocate(s + extra * sizeof(charT));
        }
        inline static void operator delete(void*);
        // use
        inline static Rep* create(size_t)
        {
            extra = frob_size(extra + 1);
            // 只传递除第一个参数的剩余实参
            Rep *p = new(extra) Rep;
            // 省略
            return p;
        }
    };
};
```

上述的 `Rep` 就是用于 reference counting, `Rep` 后方就存储具体的字符串内容
