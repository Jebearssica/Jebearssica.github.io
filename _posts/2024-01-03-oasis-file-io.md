---
layout: post
title: OASIS File IO
date: 2024-01-03 16:02 +0800
---

SEMI 提出的一种文件格式, 相较于 GDSII 能够更好压缩信息, 在 [SEMI](https://web.archive.org/web/20191104140038/http://www.wrcad.com/oasis/oasis-3626-042303-draft.pdf) 处可以下载到有关 OASIS 的手册, 这是本文的主要参考内容. 除开这个文档外, Klayout 有关 OASIS IO 部分的源码(`dbOASISReader`)也一并做为本文的参考内容.

## 一些惯用语

考虑到本人大量接触 Klayout 的源码, 因此难免将 Klayout 中的一些定义带入本文, 未避免~~人老全麻失忆~~混淆在此标注

* Cell: 一个封装实体, 内含一些平面几何信息(如矩形/梯形等等), 以及一些**其他** Cell 的抽象指针摆放(本文称 placement, Klayout 中称为 instance)

> 注意, 一个 Cell 不能包含自己的 instance, 这是语义不允许的. 当然也显而易见会造成循环依赖的问题.
{: prompt-tip }

* Placement: 一个指定了在另一个 Cell 坐标系下的特定位置/旋转/缩放摆放的 Cell 副本. 也被称作 instance, 在 Klayout 的源码中以 `Instance` 存在.

> Klayout 中针对 `Instance` 是通过代理模式实现的, 即实际不存在任何的 Cell 被拷贝, 每个 Instance 可以视作包含了一些变换的指向 Cell 的一个引用.
{: prompt-tip }

* Geometry: 二维几何图形 + layer_number 与 datatype 的属性, 后者提供了这个几何图形在版图中的层级位置(layer)
* Property: 用于标注的信息. 就我个人的经验而言, 这一部分可扩展性非常大, 每个厂商应该会向这里写入一部分数据从而实现一些性能优化. 但我并没有十足的证据证明, 因为一个中间状态的文件, 我直接在表头或表尾增加自己设计的协议也很正常. 举个例子, 一些视频厂商在表头表尾增加一些奇奇怪怪的东西可以防止一些奇奇怪怪的事情发生.
* Record: OASIS 文件中用于分割的字段

## 一些基础
