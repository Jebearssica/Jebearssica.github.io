---
layout: post
title: Virtual Function
date: 2022-07-28 10:11 +0800
tags: [c++, design pattern]
categories: [C++]
author: Jebearssica
---

## 继承( Inheritance )

虚函数常用于继承之中, 在父类中的函数种类如下:

* 纯虚函数: 父类中未给出默认定义, 子类**必须**重写
  * 当一个父类足够抽象时, 它完全无法预测子类在某些情况下的行为, 此时这些行为应当被父类设计为纯虚函数
* 虚函数: 父类中给出默认定义, 子类**尽量**重写. (你要是希望子类无需重写何必设计成虚函数呢? 用非虚函数不好吗?)
  * 一个父类在某些情况下, 可以给出一个通用的设计时, 而子类的实现细节又各自不同, 这时可以将其设计为虚函数
* 非虚函数: 父类中给出定义, 子类无需(**最好别**)重写. (你子类要重写也行, 但不符合正确的设计理念)
  * 子类的行为完全不影响该函数的行为, 应将其设计为非虚函数

## 模板方法( Template Method )

由于虚函数以及继承的特性, 衍生出 Template Method 这种**设计模式**. 它的主要精髓就是, 父类定义了一个方法框架, 子类能够在不改变该框架的情况下, 重写框架中的特定部分.

最近公开了研究生期间的实验代码, 那就拿深度学习举例(主要是太多人/书用 MFC 打开文件的代码举例了, 老是看老掉牙的东西感觉也不行).

> 当然我做的深度学习研究是真的是~~垃圾~~还有很大的提高空间, 毕竟人均顶会顶刊的领域

以深度学习的训练流程作为父类定义的框架, 子类通过重写虚函数 `loadModel` 来改变其中加载模型的步骤. 在具体执行过程中, 子类通过调用父类的非虚函数 `train`, 在执行至 `loadModel` 时, 由于它是虚函数, 因此执行子类的具体实现.

```c++
// 定义为虚函数或是纯虚函数看设计者的具体思路
virtual Model loadModel() {}
void train()
{
    init();
    auto data = loadData();
    auto model = loadModel();
    train(data, model);
    saveModel();
}
```

适用场景:

* 只希望子类扩展某个步骤, 而不是整个算法框架
* 保持父类结构的完整性
* 多个类的算法大体相似, 但只有一些细微不同时.
  * 注意: 若之后算法发生变化, 所有子类可能都需要更改
  * 具体实现, 可以将重复部分抽象至父类中, 只将不同部分在子类中保留

特点(优缺点):

* 子类只需改动整个结构的小部分, 因此改动对其余部分影响小
* 减少重复代码
* 子类受父类框架限制
  * 因此子类可能试图违反 里氏替换原则( Liskov Substitution Principle )破坏父类框架的设计
* 模板方法中的步骤越多, 越难维护

## 观察者( Observer )

简单概括就是一种订阅模式, 支持对象发生事件时通知多个订阅(观察)对象. 这两者又称发布者-订阅者. 以下是一个基于哈希表的观察者模式的 demo, 其中 `Subscriber` 作为订阅者基类可以派生出更多不同的订阅者.

```c++
class Subscriber
{
public:
    virtual void update(Publisher *p);
};

class Publisher
{
private:
    unordered_set<Subscriber *> subscirberList;

public:
    void attach(Subscriber *sub)
    {
        subscirberList.insert(sub);
    }
    void detach(Subscriber *sub)
    {
        subscirberList.erase(sub);
    }
    void publish()
    {
        for (auto &cur : subscirberList)
            cur->update(this);
    }
};
```

适用场景:

* 一个对象状态的改变需要改变其他对象, 或实际对象是未来(动态)的时候
* 或就是典型订阅模式
  * 具体实现, 发布者有个增添订阅者的接口(因此发布者必定委托于订阅者), 可以考虑发布者中聚合(或组合)订阅者, 方便更新订阅者列表.

优缺点:

* 满足开闭原则( Open/closed Principle ), 无需修改发布者就能简单引入新的订阅者类
* 能够在运行时建立对象的关系
* 发布者进行信息更新可能耗时较长, 且可能存在循环依赖(订阅者<->发布者), 从而产生循环调用使得代码崩溃

> 注意, 满足SOLID五种原则本身便是双刃剑, 有利有弊
{: .prompt-warning }

## 总结

虚函数还有很多应用场景, 但通常我们都视为**继承**与其他应用的结合, 只是其中的继承部分就会用到虚函数. 此外, 虚函数还有许多细节知识, 如与动态联编静态联编有关的知识内容. 然而我个人向来贯彻 C++ 的设计理念——"不为(暂未)不使用到的特性付出代价"(个人魔改版), 因此后面碰到了补上即可
