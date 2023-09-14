---
layout: post
title: CellCounter
date: 2023-08-04 14:31 +0800
tags: [EDA, Klayout, DB]
categories: [Klayout, DB]
author: Jebearssica
---

> A cell multiplicity generator. This class delivers the multiplicity for a cell (number of "as if flat" instances of the cell in all top cells). It is instantiated with a reference to a cell graph object. This object caches cell counts for multiple cells. It is efficient to instantiate this object once and use it as often as possible.
{: .prompt-info }

事实上, 这个类的功能非常简单, 理解的重点也在于何为 **multiplicity**, 接下来通过源码来试图理解这个概念. 另外请一定注意这里做的是针对 Cell 的计算, 而非 Instance 的计算 (Instance 只是作为计算过程中的特征, 你可以看最终的储存结果是按照 cell index 来存储的).

## 定义

检查指定 Cell 结点开始的 Layout 子树, 定义如下

```c++
class DB_PUBLIC CellCounter 
{
public:
  typedef std::map <db::cell_index_type, size_t> cache_t;
  typedef std::set <db::cell_index_type> selection_t;
  typedef selection_t::const_iterator selection_iterator;

  /** 
   *  @brief Instantiate a counter object with a reference to the given cell graph
   */
  CellCounter (const db::Layout *cell_graph);

  /** 
   *  @brief Instantiate a counter object with a reference to the given cell graph
   *
   *  This version allows one to specify a initial (starting) cell where only the cell tree below the
   *  staring cell is considered. Multiplicity refers to the number of instances below the
   *  initial cell.
   */
  CellCounter (const db::Layout *cell_graph, db::cell_index_type starting_cell);

  /**
   *  @brief Determine the instance count of the cell with index "ci"
   *
   *  The instance count is the number of "flat" instances of the cell in all
   *  top cells of the graph. A top cell has a multiplicity of 1.
   */
  size_t weight (db::cell_index_type ci);

  /**
   *  @brief Begin iterator for the cells in the selection
   *
   *  The iterator pair delivers all selected cells (only applicable if an initial cell is specified).
   */
  selection_iterator begin () const 
  {
    return m_selection.begin ();
  }
  
  /**
   *  @brief End iterator for the cells in the selection
   */
  selection_iterator end () const 
  {
    return m_selection.end ();
  }

  /**
   *  @brief Get the selection cone
   */
  const selection_t &selection () const
  {
    return m_selection;
  }

private:
  cache_t m_cache;
  selection_t m_selection; // 当没有指定起始 Cell 结点时为空
  const db::Layout *mp_cell_graph;
};
```

## 权重计算

主要关注计算权重的部分, 事实上是一个自底向上遍历的过程, 代码量很少但是拆分一下有助于更多额外信息的补充

### 终止条件

终止条件具有着非常显著的记忆化 DFS 的特征

```c++
  cache_t::const_iterator c = m_cache.find (ci);

  // 提前终止条件, cache 中已有
  if (c != m_cache.end ()) {
    return c->second;
  // 提前终止条件, 超出边界
  } else if (! m_selection.empty () && m_selection.find (ci) == m_selection.end ()) {
    return 0;
```

### 子任务

DFS 的第二个显著特征, 拆解分发子任务, 将子任务的结果合并作为当前任务的结果.

```c++
  } else {

    const db::Cell *cell = & mp_cell_graph->cell (ci);
    size_t count = 0;
    // 自底向上遍历其所有的 parent instance.
    for (db::Cell::parent_inst_iterator p = cell->begin_parent_insts (); ! p.at_end (); ++p) {
      if (m_selection.empty () || m_selection.find (p->parent_cell_index ()) != m_selection.end ()) {
        count += weight (p->parent_cell_index ()) * p->child_inst ().size ();
      }
    }
  ...
```

> 这里有个十分重要的概念, Instance 是无法在 Klayout GUI 的左侧 Cell tree 上完全体现的, 如果一个 Instance 在同一个 hierarchy tree 位置上重复多次, 此时只会有一个 Instance 在该位置, 对应代码中的 `CellInstArray`. 本文最后一小节会给出具体示例.
{: .prompt-info }
> 额外, 可以看到这里先获得当前 Cell 的所有 parent Instance 再通过 parent Instance 获取 child Instance 的个数. 因此可以断定 `Cell` 中存储 `Instance` 的结构是一个基于双链表的多叉树, 先通过当前 level 的结点向上获取父结点, 再通过父结点计算得到对应 level 的结点的个数.
{: .prompt-info }

### 初始条件以及记忆化处理

```c++
    // 显式规定 TOP 对应的权重为 1, 对应初始条件
    if (count == 0) {
      count = 1;  // top cells have multiplicity 1
    }
    // cache 中插入已经计算的结果
    m_cache.insert (std::make_pair (ci, count));
    return count;

  }
}
```

### 小结与实例分析

```c++
size_t 
CellCounter::weight (db::cell_index_type ci)
{
  cache_t::const_iterator c = m_cache.find (ci);

  if (c != m_cache.end ()) {
    return c->second;
  } else if (! m_selection.empty () && m_selection.find (ci) == m_selection.end ()) {
    return 0;
  } else {

    const db::Cell *cell = & mp_cell_graph->cell (ci);
    size_t count = 0;
    // 遍历当前 cell 的所有 parent instance，就是一个自底向上的过程
    // 要有一个概念，instance 是不能在 klayout 左侧的 hierarchy tree 上完全表现出来的
    // 如一个 instance 可能在同一个 hierarchy 位置上重复多次，那么此时只会有一个 instance 在该位置
    /* 举例如下：
      - top
        - A x2
        - B
          - A x3
    */
    for (db::Cell::parent_inst_iterator p = cell->begin_parent_insts (); ! p.at_end (); ++p) {
      // 当前 instance 对应的父 cell 的 index
      if (m_selection.empty () || m_selection.find (p->parent_cell_index ()) != m_selection.end ()) {
        // 注意，这里遍历的是 parent_inst->child_inst，因此可以认为是当前 instance 的 size 数量，举例如下
        /*
          - parent inst
            - current inst1
            - current inst2
        */
        // 因此我们可以认为 m_instance 的存储结构可以视作一个基于双链表的多叉树
        // 进一步，可以得知这里计算的并不是 cell-tree 的 contour
        // 而更偏向 instance-tree
        count += weight (p->parent_cell_index ()) * p->child_inst ().size ();
      }
    }

    if (count == 0) {
      count = 1;  // top cells have multiplicity 1
    }

    m_cache.insert (std::make_pair (ci, count));
    return count;

  }
}
```

直接给出一个示例的 hierarchy tree, 并给出每个结点的计算过程与结果, 注意 Klayout 左侧的 Cells 界面不会标注数量, 这里列出只是方便理解:

* p3
  * p7*1
  * t1*1
  * t10*3
  * t2*1
    * t1*1
    * t10*1
  * t8*1
    * p7*3
    * t10*4

计算过程如下:

* p7 计算过程:
  * p7 -> t8 -> p3: p3 = 1, t8 = 1 \* 1, p7 += 1 \* 3
  * p7 -> p3: p7 += 1 \* 1
  * p7 = 4
* t10 计算过程:
  * t10 -> t8: t10 += 1 \* 4
  * t10 -> t2 -> p3: t2 = 1 \* 1, t10 += 1 \* 1
  * t10 -> p3: t10 += 1 \* 3
  * t10 = 8
* t1 计算过程:
  * t1 -> t2: t1 += 1 \* 1
  * t1 -> p3: t1 += 1 \* 1
  * t1 = 2
* 计算结束, 所有结点结果计算完毕
