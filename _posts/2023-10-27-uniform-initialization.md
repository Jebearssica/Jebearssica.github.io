---
layout: post
title: Uniform Initialization
date: 2023-10-27 11:45 +0800
tags: [c++, C++14]
categories: [c++, C++14]
author: Jebearssica
---

## 区分 `{}` 与 `()` 两种构造方式

```c++
// C++的一些构造方式
Object obj_default;     // 默认构造
Object obj_copy = obj1; // 拷贝构造
obj_copy = obj1;        // 赋值, 非构造
Object obj_uniform{};   // 统一初始化
```

C++ 具有非常沉重的历史包袱, 为了向前兼容, 必须要保留 `()` 这种构造方法, 然而针对非静态的成员变量指定默认值时, `()` 会报错

```c++
class TestClass {
private:
    int x{0};   // correct
    int y = 0;  // correct
    int z(0);   // wrong! throw compile error
};
```

针对不可拷贝对象时, 通过 `operator=` 赋值会报错

> 事实上, GCC 13.1 C++14 不能编译通过下面的代码, 然而提升标准至 C++17 及以上时能够编译通过.
{: prompt-error }

- [ ] 可能需要另写一篇专门讲这个的起因?

```c++
std::atomic<int> a1{0};     // correct
std::atomic<int> a2(0);     // correct
std::atomic<int> a3 = 0;    // wrong! throw compile error
```

> 额外注意的是, 通过 `{}` 进行初始化时, 其内部的表达式不会自动进行变窄转换(narrowing conversion), 如果被编译器判定为变窄转换时, 会编译报错.
{: prompt-tip }

```c++
double x = 0, y = 0;
int z1{x + y};      // compile error
int z2(x + y);      // correct
int z3 = x + y;     // correct
```

> 有关 most vexing parse 问题. 即 C++ 编译器会将一切能够解析为声明的情况都解析为声明.
{: prompt-warning }

```c++
class obj;
obj obj1(); // 会被视为一个函数 obj1 声明, 该函数的返回对象为一个 obj
obj obj2{}; // 此时, obj2 被视为一个 obj 对象
```

> 针对 `{}` 进行构造, 编译器会尽可能调用 `initializer_list` 为形参的构造函数, 即使已有的构造函数能够更好匹配参数类型. 甚至是拷贝构造与移动构造.
{: prompt-info }

```c++
class obj {
    obj obj(int a, bool b);
    obj obj(int a, double b);
};
// call the first one
obj obj1(10, true);
obj obj2{10, true};
// call the second one
obj obj3(10, 5.0);
obj obj4{10, 5.0};
class obj_init : public obj {
    obj_init obj_init(std::initializer_list<bool> il);
};
// call the third one
obj_init obj5{10, true};
obj_init obj6{10, 5.0};     // even the second one is better matched.
// call the copy ctor
obj_init obj7(obj5);
// call the move ctor
obj_init obj8(std::move(obj6));
// call the third one
obj_init obj9{objy}, obj10{std::move(obj8)};
```

> 甚至参数会要求变窄转换时, 编译依旧试图匹配
{: prompt-error }

```c++
class obj_init_error : public obj {
    obj_init_error obj_init_error(std::initializer_list<bool> il);
};
// compile error: require narrowing conversion
// will throw an error since {} is not allowed for narrowing conversion
obj_init_error obj1{10, true};
```

> 仅有在无法将 `{}` 初始化中的实参转换为 `std::initializer_list` 时, 编译器才去匹配其他选项
{: prompt-info }

```c++
class obj_init_ignore : public obj {
    obj_init_ignore obj_init_ignore(std::initializer_list<std::string> il); 
};
// call the base class's first ctor
obj_init_ignore obj1{10, true};
```

> 在拥有默认构造函数时, 通过 `{}` 为空集的构造函数并不会调用形参为 `std::initializer_list` 的构造函数, 而是调用默认构造函数. 可以通过创建一个空的 `std::initializer_list` 作为实参来调用形参为 `std::initializer_list` 的构造函数.
{: prompt-tip }

```c++
obj_init_error obj1();  // declare a function named obj1 which returns an "obj_init_error" object
obj_init_error obj2{};  // call default ctor
obj_init_error obj3;    // call default ctor
obj_init_error obj4{{}};// call `std::initializer_list<bool>` ctor
```

## 总结

综上, 我们可以发现重载构造函数时, 如果额外添加了形参为 `std::initializer_list` 的构造函数时, 可能会使得用户在使用这些构造函数时通过 `{}` 构造产生与之前不一样的结果. 因此如果做为一个更加偏向底层给用户提供接口的开发者(就比如我现在是一个苦逼的底层 db 开发人员), 不论是为了自己(~~db 就我一个人, 很少有人用我的接口~~), 还是为了他人(~~迟早我留下的屎山会有人看到的~~), 都应该小心. 可以通过 `std::vector` 来举例:

```c++
std::vector<int> zero_vector_with_ten_length(0, 10);    // a vector full of 0, and its length equals 10
std::vector<int> zero_and_ten_vector{0, 10};            // a vector with 2 elements, 0 and 10
```

特别的, 如果提供了一个模板对象, 那么更应该关注使用者的使用问题, 因为这通常牵扯到一个常见写法, 可变参数模板.

```c++
template<typename T,            //要创建的对象类型
         typename... Ts>        //要使用的实参的类型
void doSomeWork(Ts&&... params)
{
    // create object T with many params
    T obj1(std::forward<Ts>(params)...);
    T obj2{std::forward<Ts>(params)...};
    ...
}
doSomeWork<std::vector<int>>(10, 0);
// obj1 with 10 elements
// obj2 with 2 elements
```

同样的, 做为更上层的开发者, 在使用时应当仔细考量 `{}` 与 `()` 构造对象的差别, 多数情况下使用其中会视为一种默认情况, 这取决于你所处的行业以及周遭环境.
