---
layout: post
title: Cell Mapping
date: 2023-07-27 15:33 +0800
tags: [EDA, Klayout, DB]
categories: [Klayout, DB]
author: Jebearssica
---

> TL;DR: `CellMapping` 在构造一个等价关系表, 从而在对 `Layout` 进行拷贝或对比时能够避免不必要的新 `Cell` 的生成, 从而直接将 `src_layout` 中的 `Cell` 直接拷贝/比较 `dst_layout` 中的 `Cell`.
{: .prompt-info }
> 为了缩减非重点代码, 以及简化写法, 文中涉及代码可能通过C++11以上的特性或直接进行简写, 与源代码不一致. 总之就是因为个人喜好, 有关 Klayout 的源码会有所变动.
{: .prompt-tip }

## 简介

在 Klayout 中通常有如下三种情况来创建两 `Layout` 间的 `CellMapping`:

* `create_single_mapping_full`: 只连接给定的两个 `Cell`, `src_layout` 的其余子 `Cell` 会直接与 `dst_layout` 中创建的新 `Cell` 相连接
* `create_from_names_full`: 连接给定的两个 `Cell`, 对于`src_layout` 的其余子 `Cell` 会试图与 `dst_layout` 中同名的 `Cell` 相连接, 剩余 `Cell` 同样与新创建的 `Cell` 相连接
* `create_from_geometry_full`: 连接给定的两个 `Cell`, 对于 `src_layout` 的其余子 `Cell` 会试图与 `dst_layout` 同路径同累积变换的 `Cell` 相连接, 剩余 `Cell` 同样与新创建的 `Cell` 相连接

### `create_single_mapping_full`

真的很简单, 不信看下面源码

```c++
void 
CellMapping::create_single_mapping (const db::Layout & /*layout_a*/, db::cell_index_type cell_index_a, const db::Layout & /*layout_b*/, db::cell_index_type cell_index_b)
{
  clear ();
  // 简单建立映射关系, 剩下的全都当 missing part 的处理
  map (cell_index_b, cell_index_a);
}
```

### `create_from_names_full`

此处逻辑也较为简单, 不过应该避免混淆 `Cell::collect_called_cells` 与 `Cell::collect_caller_cells`. 前者 DFS 搜集所有的子 Cell(注意是 Cell 而非 instance), 后者向上收集所有父 Cell.

```c++
void
CellMapping::create_from_names (const db::Layout &layout_a, db::cell_index_type cell_index_a, const db::Layout &layout_b, db::cell_index_type cell_index_b)
{
  clear ();

  std::set<db::cell_index_type> called_b;
  layout_b.cell (cell_index_b).collect_called_cells (called_b);

  map (cell_index_b, cell_index_a);

  // 遍历 cell_b 下所有子 Cell, 如果在 layout_a 中能找到同名 Cell 也进行映射
  for (auto const &cell_b_index : called_b) {
    std::pair<bool, db::cell_index_type> ac = layout_a.cell_by_name (layout_b.cell_name (cell_b_index));
    if (ac.first) {
      map (cell_b_index, ac.second);
    }
  }
}
```

### `create_from_geometry_full`

> 逻辑异常复杂, 涉及到的机制也异常多, 需要非常熟悉 Klayout 基本 db, 包括各个类之间的存储关系等等, 因此需要先具备一些知识储备
{: .prompt-warning }

* [`CellCounter`]({% link _posts/2023-08-04-cellcounter.md %})

```c++
void 
CellMapping::create_from_geometry (const db::Layout &layout_a, db::cell_index_type cell_index_a, const db::Layout &layout_b, db::cell_index_type cell_index_b)
{
  tl::SelfTimer timer (tl::verbosity () >= 31, tl::to_string (tr ("Cell mapping")));

  if (tl::verbosity () >= 40) {
    tl::info << "Cell mapping - first step: mapping instance count and instance identity";
  }

  clear ();

  db::CellCounter cc_a (&layout_a, cell_index_a);
  db::CellCounter cc_b (&layout_b, cell_index_b);

  std::multimap<size_t, db::cell_index_type> cm_b;
  for (auto const &cell_index : cc_b) {
    cm_b.insert (std::make_pair (cell_index == cell_index_b ? 0 : cc_b.weight (cell_index), cell_index));
  }

  std::multimap<size_t, db::cell_index_type> cm_a;
  for (auto const &cell_index : cc_a) {
    cm_a.insert (std::make_pair (cell_index == cell_index_a ? 0 : cc_a.weight (cell_index), cell_index));
  }

  std::map <db::cell_index_type, std::vector<db::cell_index_type> > candidates; // key = index(a), value = indices(b)

  InstanceSetCompareFunction cmp (layout_a, cell_index_a, layout_b, cell_index_b);

  std::multimap<size_t, db::cell_index_type>::const_iterator a = cm_a.begin (), b = cm_b.begin ();
  while (a != cm_a.end () && b != cm_b.end ()) { // mapping the same-weight cell, from layout_a to layout_b

    size_t w = a->first;
    while (b != cm_b.end () && b->first < w) { // 双指针快速找 instance 数量相同的 cell
      ++b;
    }

    if (b == cm_b.end ()) {
      break;
    } else if (b->first > w) { // 含义为 cell: a->second 对应的相同来自 b 的 cell 队列为空
      candidates.insert (std::make_pair (a->second, std::vector<db::cell_index_type> ()));
      ++a;
    } else { // weight(cell_a) == weight(cell_b)

      if (tl::verbosity () >= 50) { // 正如 debug 信息，这是一个多对多的映射处理，处理内容为相同的 instance count
        size_t na = 0, nb = 0;
        for (auto aa = a; aa != cm_a.end () && aa->first == w; ++aa) {
          ++na;
        }
        for (auto bb = b; bb != cm_b.end () && bb->first == w; ++bb) {
          ++nb;
        }
        tl::info << "Multiplicity group (" << w << " instances) - " << na << " vs. " << nb << " cells";
      }

      unsigned int g = 0; // 可以视作每个处理区间内的 index ?
      std::map <unsigned int, std::vector <db::cell_index_type> > b_group;
      std::map <db::cell_index_type, unsigned int> b_group_of_cell; // 记录 cell_b 对应索引 g，类似 cache 作用

      while (a != cm_a.end () && a->first == w) { // 确保没有步进 set_a, 

        candidates.insert (std::make_pair (a->second, std::vector <db::cell_index_type> ())); // a new entry

        std::set <unsigned int> groups_taken;

        std::multimap<size_t, db::cell_index_type>::const_iterator bb = b;
        while (bb != cm_b.end () && bb->first == w) { // start do real calculation, current calculation interval

          std::map <db::cell_index_type, unsigned int>::const_iterator bg = b_group_of_cell.find (bb->second);
          if (bg != b_group_of_cell.end ()) { // already been calculated

            if (groups_taken.find (bg->second) == groups_taken.end ()) { // if current groups has not been selected yet
              if (cmp.compare (a->second, cc_a.selection (), bb->second, cc_b.selection ())) { // a->second equals bb->second
                candidates [a->second] = b_group [bg->second]; // overwrite the candidates, 存在重复相同的应当覆盖
                groups_taken.insert (bg->second);
              }
            }

          } else { // list_a 与 list_b 进行匹配时，第一轮遍历 list_b 产生新 candidate
                   // 实质此处产生的对应关系如果在当前 weight 情况下不再经过上述 if 分支，则已经认为是一一映射

            if (cmp.compare (a->second, cc_a.selection (), bb->second, cc_b.selection ())) { // a->second equals bb->second
              candidates [a->second].push_back (bb->second); // add more candidates
              b_group_of_cell.insert (std::make_pair (bb->second, g));
              // 无语，b_group[g].push_back(bb->second) 不行吗
              b_group.insert (std::make_pair (g, std::vector <db::cell_index_type> ())).first->second.push_back (bb->second);
            }

          }

          ++bb;

        }

        if (tl::verbosity () >= 60) {
          tl::info << "Checked cell " << layout_a.cell_name (a->second) << ": " << candidates [a->second].size () << " candidates remaining.";
        }
          
        ++a; // a 与 g 同步步进，意味着在每个处理区间内，可以视作 a 与 g 之间一一映射
        ++g; // 即，我们可以通过索引 g 来得知目前处理的是当前区间内的第几个 cell_a

      }

      while (b != cm_b.end () && b->second == w) { // 处理完当前 cell_b 步进 set_b
        ++b;
      }

    }

  }

  while (a != cm_a.end ()) { // collect remaining cell in set_a
    candidates.insert (std::make_pair (a->second, std::vector<db::cell_index_type> ()));
    ++a;
  }

  if (tl::verbosity () >= 60) {
    tl::info << "Mapping candidates:";
    dump_mapping (candidates, layout_a, layout_b);
  }
  // extract one vs one mapping to m_b2a_mapping
  for (std::map <db::cell_index_type, std::vector<db::cell_index_type> >::const_iterator cand = candidates.begin (); cand != candidates.end (); ++cand) {
    extract_unique (cand, m_b2a_mapping, layout_a, layout_b);
  }

  int iteration = 0;

  bool reduction = true;
  while (reduction) {

    reduction = false;
    ++iteration;

    if (tl::verbosity () >= 40) {
      tl::info << "Cell mapping - iteration " << iteration << ": cross-instance cone reduction";
    }

    // This map stores that layout_b cells with the corresponding layout_a cell for such cells which 
    // have their mapping reduced to a unique one 
    std::map <db::cell_index_type, std::pair<db::cell_index_type, int> > unique_candidates;

    std::vector<db::cell_index_type> refined_cand;

    for (std::map <db::cell_index_type, std::vector<db::cell_index_type> >::iterator cand = candidates.begin (); cand != candidates.end (); ++cand) {

      if (cand->second.size () > 1) { // process the multiple mapping cases, since the 1 vs 1(unique) mapping cases have already processed

        refined_cand.clear ();
        refined_cand.insert (refined_cand.end (), cand->second.begin (), cand->second.end ());

        if (tl::verbosity () >= 70) {
          tl::info << "--- Cell: " << layout_a.cell_name (cand->first);
          tl::info << "Before reduction: " << tl::noendl; 
          for (size_t i = 0; i < refined_cand.size (); ++i) { 
            tl::info << " " << layout_b.cell_name(refined_cand[i]) << tl::noendl; 
          } 
          tl::info << "";
        }

        std::set<db::cell_index_type> callers;
        layout_a.cell (cand->first).collect_caller_cells (callers, cc_a.selection (), -1);

        for (std::set<db::cell_index_type>::const_iterator c = callers.begin (); c != callers.end () && refined_cand.size () > 0; ++c) {

          if (*c != cell_index_a) {

            const std::vector<db::cell_index_type> &others = candidates.find (*c)->second;
            if (others.size () == 1) {

              std::set<db::cell_index_type> cross_cone_b;
              layout_b.cell (others.front ()).collect_called_cells (cross_cone_b);

              std::vector<db::cell_index_type>::iterator cout = refined_cand.begin ();
              for (std::vector<db::cell_index_type>::const_iterator cc = refined_cand.begin (); cc != refined_cand.end (); ++cc) {
                if (cross_cone_b.find (*cc) != cross_cone_b.end ()) {
                  *cout++ = *cc;
                }
              }

              if (tl::verbosity () >= 70 && cout != refined_cand.end ()) {
                tl::info << "Reduction because of caller mapping: " << layout_a.cell_name (*c) << " <-> " << layout_b.cell_name (others[0]);
                tl::info << "  -> " << tl::noendl; 
                for (size_t i = 0; i < size_t (cout - refined_cand.begin ()); ++i) { 
                  tl::info << " " << layout_b.cell_name(refined_cand[i]) << tl::noendl; 
                } 
                tl::info << "";
              }

              refined_cand.erase (cout, refined_cand.end ());

            }

          }

        }

        if (refined_cand.size () > 0) {

          std::set<db::cell_index_type> called;
          layout_a.cell (cand->first).collect_called_cells (called);

          for (std::set<db::cell_index_type>::const_iterator c = called.begin (); c != called.end () && refined_cand.size () > 0; ++c) {

            const std::vector<db::cell_index_type> &others = candidates.find (*c)->second;
            if (others.size () == 1) {

              std::set<db::cell_index_type> cross_cone_b;
              layout_b.cell (others.front ()).collect_caller_cells (cross_cone_b, cc_b.selection (), -1);

              std::vector<db::cell_index_type>::iterator cout = refined_cand.begin ();
              for (std::vector<db::cell_index_type>::const_iterator cc = refined_cand.begin (); cc != refined_cand.end (); ++cc) {
                if (cross_cone_b.find (*cc) != cross_cone_b.end ()) {
                  *cout++ = *cc;
                }
              }

              if (tl::verbosity () >= 70 && cout != refined_cand.end ()) {
                tl::info << "Reduction because of callee mapping: " << layout_a.cell_name (*c) << " <-> " << layout_b.cell_name (others[0]);
                tl::info << "  -> " << tl::noendl; 
                for (size_t i = 0; i < size_t (cout - refined_cand.begin ()); ++i) { 
                  tl::info << " " << layout_b.cell_name(refined_cand[i]) << tl::noendl; 
                } 
                tl::info << "";
              }

              refined_cand.erase (cout, refined_cand.end ());

            }

          }

          if (refined_cand.size () == 1) {

            //  The remaining cell is a candidate for layout_b to layout_a mapping
            db::cell_index_type cb = refined_cand[0];
            db::cell_index_type ca = cand->first;

            std::map <db::cell_index_type, std::pair<db::cell_index_type, int> >::iterator uc = unique_candidates.find (cb);
            if (uc != unique_candidates.end ()) {
              if (uc->second.first != ca) {
                int ed = tl::edit_distance (layout_a.cell_name (ca), layout_b.cell_name (cb));
                if (ed < uc->second.second) {
                  uc->second = std::make_pair (ca, ed);
                  if (tl::verbosity () >= 60) {
                    tl::info << "Choosing " << layout_b.cell_name (cb) << " (layout_b) as new unique mapping for " << layout_a.cell_name (ca) << " (layout_a)";
                  }
                }
              }
            } else {
              int ed = tl::edit_distance (layout_a.cell_name (ca), layout_b.cell_name (cb));
              unique_candidates.insert (std::make_pair (cb, std::make_pair (ca, ed)));
              if (tl::verbosity () >= 60) {
                tl::info << "Choosing " << layout_b.cell_name (cb) << " (layout_b) as unique mapping for " << layout_a.cell_name (ca) << " (layout_a)";
              }
            }

          }

        }

      }

    }

    //  realize the proposed unique mapping
    for (std::map <db::cell_index_type, std::pair<db::cell_index_type, int> >::const_iterator uc = unique_candidates.begin (); uc != unique_candidates.end (); ++uc) {

      std::map <db::cell_index_type, std::vector<db::cell_index_type> >::iterator cand = candidates.find (uc->second.first);
      tl_assert (cand != candidates.end ());
      cand->second.clear ();
      cand->second.push_back (uc->first);
      reduction = true;
      extract_unique (cand, m_b2a_mapping, layout_a, layout_b);

    }

    if (tl::verbosity () >= 60) {
      tl::info << "Further refined candidates:";
      dump_mapping (candidates, layout_a, layout_b);
    }
    
    if (tl::verbosity () >= 40) {
      tl::info << "Cell mapping - iteration " << iteration << ": removal of uniquely mapped cells on B side";
    }

    for (std::map <db::cell_index_type, std::vector<db::cell_index_type> >::iterator cand = candidates.begin (); cand != candidates.end (); ++cand) {

      if (cand->second.size () > 1) {

        std::vector<db::cell_index_type> refined_cand;
        for (std::vector<db::cell_index_type>::const_iterator c = cand->second.begin (); c != cand->second.end (); ++c) {
          std::map<db::cell_index_type, db::cell_index_type>::const_iterator um = m_b2a_mapping.find (*c);
          if (um == m_b2a_mapping.end () || um->second == cand->first) {
            refined_cand.push_back (*c);
          }
        }

        if (refined_cand.size () < cand->second.size ()) {
          reduction = true;
          cand->second.swap (refined_cand);
          extract_unique (cand, m_b2a_mapping, layout_a, layout_b);
        }

      }

    }

    if (tl::verbosity () >= 60) {
      tl::info << "After reduction of mapped cells on b side:";
      dump_mapping (candidates, layout_a, layout_b);
    }
    
  }

  if (tl::verbosity () >= 40) {

    int total = 0;
    int not_mapped = 0;
    int unique = 0;
    int non_unique = 0;
    int alternatives = 0;

    for (std::map <db::cell_index_type, std::vector<db::cell_index_type> >::iterator cand = candidates.begin (); cand != candidates.end (); ++cand) {
      ++total;
      if (cand->second.size () == 0) {
        ++not_mapped;
      } else if (cand->second.size () == 1) {
        ++unique;
      } else {
        ++non_unique;
        alternatives += (int) cand->second.size ();
      }
    }

    tl::info << "Geometry mapping statistics:";
    tl::info << "  Total cells = " << total;
    tl::info << "  Not mapped = " << not_mapped;
    tl::info << "  Unique = " << unique;
    tl::info << "  Non unique = " << non_unique << " (total " << alternatives << " of alternatives)";

  }

  //  Resolve mapping according to string match

  if (tl::verbosity () >= 40) {
    tl::info << "Cell mapping - string mapping as last resort";
  }

  for (std::map <db::cell_index_type, std::vector<db::cell_index_type> >::iterator cand = candidates.begin (); cand != candidates.end (); ++cand) {

    if (cand->second.size () > 1) {

      std::string cn_a (layout_a.cell_name (cand->first));

      int min_ed = std::numeric_limits<int>::max ();
      db::cell_index_type min_ed_ci;

      for (std::vector<db::cell_index_type>::const_iterator c = cand->second.begin (); c != cand->second.end (); ++c) {

        if (m_b2a_mapping.find (*c) == m_b2a_mapping.end ()) {

          int ed = tl::edit_distance (cn_a, layout_b.cell_name (*c));
          if (ed < min_ed) {
            min_ed = ed;
            min_ed_ci = *c;
          }

        }

      }

      cand->second.clear ();
      if (min_ed < std::numeric_limits<int>::max ()) {
        cand->second.push_back (min_ed_ci);
        extract_unique (cand, m_b2a_mapping, layout_a, layout_b);
      }

    }

  }

  if (tl::verbosity () >= 40) {

    int total = 0;
    int not_mapped = 0;
    int unique = 0;
    int non_unique = 0;
    int alternatives = 0;

    for (std::map <db::cell_index_type, std::vector<db::cell_index_type> >::iterator cand = candidates.begin (); cand != candidates.end (); ++cand) {
      ++total;
      if (cand->second.size () == 0) {
        if (tl::verbosity () >= 50) {
          tl::info << "Unmapped cell: " << layout_a.cell_name (cand->first);
        }
        ++not_mapped;
      } else if (cand->second.size () == 1) {
        ++unique;
      } else {
        ++non_unique;
        alternatives += (int) cand->second.size ();
      }
    }

    tl::info << "Final mapping statistics:";
    tl::info << "  Total cells = " << total;
    tl::info << "  Not mapped = " << not_mapped;
    tl::info << "  Unique = " << unique;
    tl::info << "  Non unique = " << non_unique << " (total " << alternatives << " of alternatives)";

  }
}
```
