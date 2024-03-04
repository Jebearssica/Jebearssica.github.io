---
layout: post
title: Klayout DB
date: 2023-09-19 16:09 +0800
---

这玩意儿也太大了, 根本就不可能概括.

首先要记住以下概念:

* child instance: 指的是当前 cell/instance 的下一个 hierarchy level 的 instance
  * 注意: child instance 与 parent instance 是一对概念, 就是把 hierarchy tree 的结点换成 instance.
    * 注意中的注意: 给我忘记能够 Cell 的子 instance 的概念, 你根本没法通过 Cell 找到其对应的所有 instance
* child cell: 同理与 parent cell 相对应, 就是把 hierarchy tree 的结点换成 Cell
* CellInstArray: 同一 hierarchy level 中一个 Cell 的部分(或所有) instance 集合

一切 hierarchy 层级上 `begin` 获得的迭代器的含义都是获取 child list

## Cell & Instance

`Cell` 为一个实体, 在 hierarchy 上内部存有 child instance 信息与当前 Cell 内 shape 信息.

`Instances` 为一个抽象, 视为一个实体 `CellInstArray` 的代理. 一个 `CellInstArray` 的含义为, 一些 instance 集合(instance 对应的 `Cell` 为该 array 中存储的 `cell_index` 指代的 `Cell`). 举例如下:

* top
  * C1 x2
  * C2 x3

top 下有两个 Cell, C1 与 C2. 因此 top 下**至少**有两个 `CellInstArray`, 分别代表两个 C1 instance 构成的集合与三个 C2 instance 构成的集合.

> 此处我们应当理解**至少**的含义. 两个 C1 instance 可能存在一个 `CellInstArray` 中(例如, 它们可以构成一个 `regular_array`), 同理三个 C2 instance 也可能存储在至少一个至多三个`CellInstArray` 中. 但是, C1 与 C2 的 instance 一定不会在同一个 `CellInstArray` 中.
{: prompt-tip }

`CellInstArray` 的存储信息可以视作一个 `cell_index` 指明所代表的 `Cell`, 以及一系列 transform, 每个 transform 的含义为从当前 instance 转换至其 parent_cell 的转换. 可以认定, 在 Cell 层级上, 一个 Cell 有多少个子 Cell, 就代表着一个 Cell 下至少有多少个 `CellInstArray`. 那么下面举一个遍历一个 Cell 所有直连子 instance transform 的例子

```c++
const db::Cell &topCell = layout.cell(*layout.begin_top_down());
for (auto childInst = top_cell.begin(); !childInst.at_end(); ++childInst) {
  const db::CellInstArray &instArray= childInst->cell_inst();
  for (auto inst = instArray.begin(); !inst.at_end(); ++inst) {
    db::ICplxTrans curInstTrans = instArray.complex_trans(*inst);
  }
}
```

`Cell::begin_parent_cells()` 是获取从当前 Cell 的 parent_cell 的迭代器, 对于有多个 parent cell 的情况而言可以很快获取当前 Cell 对应的所有 parent_cell. 同样的, 我们举例如下:

* top
  * C1
    * C2
  * C2

我们通过下列代码能够遍历所有 C2 的 parent_cell, 即 Cell top 与 Cell C1

```c++
// assume we get cell_index representing Cell C2
const db::Cell &current_cell = layout.cell(cell_index);
for (auto pc = current_cell.begin_parent_cells(); pc != current_cell.end_parent_cells(); ++pc) {
  // now we get Cell C1 & Cell top
  const db::Cell &parent_cell = layout.cell(*pc);
}
```

## `CellInstArray`

下面的代码揭示了 `CellInstArray` 的真实含义. 其中 `CellInst` 只存了一个 cell_index.

```c++
typedef db::array <db::CellInst, db::Trans> CellInstArray;

struct array {
  ...
private:
  Obj m_obj;
  trans_type m_trans;
  basic_array <coord_type> *mp_base;
};
```

核心部分实质是 `basic_array`, 做为一个抽象类提供了一些遍历方式

### `regular_array`

### `iterated_array`

### `single_complex_inst`
