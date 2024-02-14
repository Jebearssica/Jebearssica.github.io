---
layout: post
title: Flatten Cell
date: 2023-11-21 13:54 +0800
tags: [EDA, Klayout, DB]
categories: [Klayout, DB]
author: Jebearssica
---

## 注意事项

解析一下 Klayout 中 Flatten Cell 功能是如何实现的. 与之相似的功能有: Flatten Instance, Flatten Region, 相似功能会在介绍过程中提及. 当然, 在了解如何实现之前, 首先得明确理解这个功能是什么, Klayout 文档中清晰描述了这个:

> The "flatten cell" operation flattens a cell into all of its parents. This basically removes a cell by promoting her shapes and instances up in the hierarchy. Cell flattening can be applied to single instances or cells as a whole. When applied to an instance, the individual instance is resolved into shapes. The instantiated cell will still exist afterwards. When applied to a cell, the cell will disappear and replaced by its contents in all places it is used.

{: prompt-error }

很清楚是吧, 然而是错的! 在当前最新版中 (0.28.12) 选中 Cell 右键执行 Flatten Cell, 选择 first hierarchy level 亦或是自定义一个 level, 最终效果是将该 Cell 的子 instance 以及 shape flatten 至被选择的 Cell.

## 源码定位

一般而言, 在知道一个功能后想要阅读相关代码首先得定位到具体代码. 这一步对于一个拥有庞大源代码的项目而言并非易事. 但通常解决方法也比较简单, **问人**. 是的, 最迅速有效的方法就是找个大佬问他, 但我个人比较倾向万事求己不求人, 以下简单记录一下定位代码的流程.

### 理解项目

最最最关键的前置条件, 你需要对项目有个大致了解. 例如, 在当前情况下, 我知道官方文档对于功能的关键词可能会绑定在相关代码的注释或是日志中. 举例, 在之前涉及 density 的学习时, 我知道了 ruby 脚本绑定了一个叫 `_with_density` 的函数如下, 就是通过文档的 API 名称(`with_density`)搜索得到.

```ruby
def with_density(*args)
  self._with_density("with_density", false, *args)
end
```

### 搜索关键字

因此, 我们搜索 Flatten Cell, 排除 xml 的文档文件以及 changelog, 可以发现如下结果:

```c++
// layHierarchyControlPanel.cc:1225
menu_entries.push_back (lay::menu_item ("cm_cell_flatten", "flatten_cell:edit:edit_mode", at, tl::to_string (QObject::tr ("Flatten Cell"))));
// layLayoutViewFunctions.cc:822
manager ()->transaction (tl::to_string (tr ("Flatten cell")));
// layLayoutViewFunctions.cc:2118
menu_entries.push_back (lay::menu_item ("cm_cell_flatten", "flatten_cell:edit:edit_mode", at, tl::to_string (tr ("Flatten Cell"))));
```

### 分析顶层代码

很显然上述文件位于 `layui`, 虽然看不懂前缀 lay 的意思是不是 layout, 但我们至少知道这个是 ui 相关的代码. 如果你有相关 ui 代码(甚至我觉得不需要), 如果你了解按钮事件绑定这类似的东西的话你就应该知道, 按下一个按钮触发一个事件函数. 1, 3两个选项告知了我们下一个关键字应该是 `cm_cell_flatten`. 因此你应该能十分快速定位至下面的代码 `LayoutViewFunctions::cm_cell_flatten ()`

排除与 ui 相关的代码, 比如一些文本框输出之类的玩意儿, 核心部分如下:

```c++
void
LayoutViewFunctions::cm_cell_flatten ()
{
  ...
        db::Layout &layout = cv->layout ();

        std::set<db::cell_index_type> child_cells;
        for (std::vector<HierarchyControlPanel::cell_path_type>::const_iterator p = paths.begin (); p != paths.end (); ++p) {
          if (p->size () > 0) {
            layout.cell (p->back ()).collect_called_cells (child_cells);
          }
        }

        //  don't flatten cells which are child cells of the cells to flatten
        std::set<db::cell_index_type> cells_to_flatten;
        for (std::vector<HierarchyControlPanel::cell_path_type>::const_iterator p = paths.begin (); p != paths.end (); ++p) {
          if (p->size () > 0 && child_cells.find (p->back ()) == child_cells.end ()) {
            cells_to_flatten.insert (p->back ());
          }
        }

        for (std::set<db::cell_index_type>::const_iterator c = cells_to_flatten.begin (); c != cells_to_flatten.end (); ++c) {
          db::Cell &target_cell = layout.cell (*c);
          layout.flatten (target_cell, flatten_insts_levels, prune);
        }

        layout.cleanup ();

        if (supports_undo && manager ()) {
          manager ()->commit ();
        }
  ...
}
```

### 分析核心代码

显然, 我们不需要关注 ui 部分, 恰好我们对 db 有些许了解(你至少需要知道 Layout 是 db 中一个很重要的数据结构), 因此跳转 `db::Layout::flatten`

先从 wrapper 开始, 我们可以比较好的了解初始状态. 了解一个函数最好的方法就是自己人脑过一遍完整流程(或者 debug 多次). 显然, 我们是从文档起手的, 因此知道形参含义, 但我们不能全信文档, 因为 Klayout 开源力量较弱, 文档这种边角部分可能会有点问题
, 因此看一下头文件里的注释

```c++
/*
   *  @param cell The cell to flatten
   *  @param levels The number of levels to flatten or -1 for all levels (0: nothing, 1: flatten on level of instances etc.)
   *  @param prune If true, remove all cells which became orphans by the flattening
*/
void 
Layout::flatten (db::Cell &cell_to_flatten, int levels, bool prune) 
{
  std::set<db::cell_index_type> direct_children;
  if (prune) {
    //  save direct children
    cell_to_flatten.collect_called_cells (direct_children, 1);
  }

  flatten (cell_to_flatten, cell_to_flatten, db::ICplxTrans (), levels);

  if (prune) {

    //  determine all direct children that are orphans now.
    for (std::set<db::cell_index_type>::iterator dc = direct_children.begin (); dc != direct_children.end (); ) {
      std::set<db::cell_index_type>::iterator dc_next = dc;
      ++dc_next;
      if (cell (*dc).parent_cells () != 0) {
        direct_children.erase (dc);
      }
      dc = dc_next;
    }

    //  and prune them
    prune_cells (direct_children.begin (), direct_children.end (), levels - 1);

  }
}
```

起始状态: src_cell == target_cell, t 为空, level 为指定值

```c++
void
Layout::flatten (const db::Cell &source_cell, db::Cell &target_cell, const db::ICplxTrans &t, int levels)
{
  db::ICplxTrans tt = t;
  // 当 src 与 target 不一致时, src 向 target 进行 shape push
  if (&source_cell != &target_cell) {

    unsigned int nlayers = layers ();
    for (unsigned int l = 0; l < nlayers; ++l) {

      if (is_valid_layer (l)) {

        db::Shapes &target_shapes = target_cell.shapes (l);
        const db::Shapes &source_shapes = source_cell.shapes (l);

        tl::ident_map<db::Layout::properties_id_type> pm1;
        for (db::Shapes::shape_iterator sh = source_shapes.begin (db::ShapeIterator::All); ! sh.at_end (); ++sh) {
          // 对所有 shape 施加 transform 后插入
          target_shapes.insert (*sh, tt, pm1);
        }

      }

    }

  }

  if (levels == 0) {
    // 当 src 与 target 不同时, src 的 child inst 直接插入至 target
    if (&source_cell != &target_cell) {
      for (db::Cell::const_iterator inst = source_cell.begin (); ! inst.at_end (); ++inst) {
        // 插入新 instance
        db::Instance new_inst = target_cell.insert (*inst);
        // 将新的 instance 替换为带有当前 trans 的 instance
        // (实际可以整体理解为插入一个带有当前 trans 的 instance)
        target_cell.transform (new_inst, tt);
      }
    }

  } else if (&target_cell == &source_cell) { // 起始状态下的第一次分支进入

    update ();

    try {

      //  Note: suppressing the update speeds up the flatten process considerably since 
      //  even an iteration of the instances requires an update.
      start_changes ();

      db::Instances old_instances (&target_cell);
      old_instances = target_cell.instances ();
      target_cell.clear_insts ();
      // 遍历 child instances
      for (db::Cell::const_iterator inst = old_instances.begin (); ! inst.at_end (); ++inst) {

        db::CellInstArray cell_inst = inst->cell_inst ();
        // bfs, 针对每个 inst 单独 flatten
        for (db::CellInstArray::iterator a = cell_inst.begin (); ! a.at_end (); ++a) {
          db::ICplxTrans tinst = t * cell_inst.complex_trans (*a);
          flatten (cell (cell_inst.object ().cell_index ()), target_cell, tinst, levels < 0 ? levels : levels - 1);
        }

      }

      end_changes ();

    } catch (...) {
      end_changes ();
      throw;
    }

  } else {
    
    try {

      //  Note: suppressing the update speeds up the flatten process considerably since 
      //  even an iteration of the instances requires an update.
      start_changes ();

      for (db::Cell::const_iterator inst = source_cell.begin (); ! inst.at_end (); ++inst) {

        db::CellInstArray cell_inst = inst->cell_inst ();

        for (db::CellInstArray::iterator a = cell_inst.begin (); ! a.at_end (); ++a) {
          db::ICplxTrans tinst = t * cell_inst.complex_trans (*a);
          flatten (cell (cell_inst.object ().cell_index ()), target_cell, tinst, levels < 0 ? levels : levels - 1);
        }

      }

      end_changes ();

    } catch (...) {
      end_changes ();
      throw;
    }

  }

}
```

