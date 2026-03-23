---
layout: post
title: EdgeProcessor
date: 2023-10-11 13:59 +0800
tags: [EDA, Klayout, DB]
categories: [Klayout, DB]
author: Jebearssica
---

一个非常通用的扫描线算法流程, 在 Klayout 内部对许多运算中都用到的运算框架. 要注意的是, 由于 Klayout 的项目几乎通过一个人开发, 为了避免代码膨胀减轻维护压力, 所以有大量的代码复用. 然而这是一个双刃剑, 代码维护轻松了, 但是由于一个流程框架涉及众多命令, 因此势必相较于特例化的算法流程而言更加复杂低效.

很难读懂, 一个看似没什么作用的值重置可能都隐含着一类命令在此引发的问题


初始化先将所有边按照 y 坐标排序

## 交点检测

分块处理，对于每个块（band）内的所有边按照 x 排序后，遍历所有边对进行相交检查，将新生成的交点信息存储至CutPoint。其中分块处理的意义在于缩小需要进行相交检查的边对规模，避免大规模无意义的边相交检查（例如，两条边相隔非常远的情况）

## 分带（or 块）算法优化

这是一种处理思想而非特定算法或数据结构，主要目的是降低算法的常数项系数，毕竟同样的复杂度的算法，在实际工程中的运行时间可能差一个数量级。其核心思想在于对数据集合进行适当划分，并在划分后的每个小集合内进行部分预处理。面对超大规模的数据集合时（如地理/数据库等超大规模的工程），能够在理论时间复杂度确定的情况下大幅度优化运行速度。

对于求边集合的相交情况而言，理论就是需要每对边进行相交检查，从而时间复杂度为 $O(n^2)$ ，但实际应用上可以通过分块的方式降低运算规模。需要确保每个分块中元素不过多（规模过大导致算法常数项系数大），同时元素也不能过少（总分块数过多，排序的常数项系数过大），可能需要一个类似于负载均衡的策略来自动调整填充数量。
从代码实现上分析，可以在外循环内根据 y 坐标划分带坐标y \in \lbrack y, yy\rparen，边集合 $edge \in \lbrack current, future \rparen$ 在内循环中按照 x 坐标划分块（cell）坐标 $x \in \lbrack x, xx\rparen$，边集合$edge \in \lbrack c, f\rparen$，通过相同策略填充每个分块。最终确保相交检查只发生在每个分块内。

* 确定分带边界情况$\lbrack y, yy\rparen$
  * 以 current 作为当前边，future 为第一个未进入该分带的边

```C++
    // y 方向排序
    std::sort (mp_work_edges->begin (), mp_work_edges->end (), edge_ymin_compare<db::Coord> ());
    // 初始化当前扫描线 y
    y = edge_ymin ((*mp_work_edges) [0]);
    // future 为未进入当前分带的第一条边，即确定当前分带右侧边界（对于边而言）
    future = mp_work_edges->begin ();
    // 主循环
    for (std::vector <WorkEdge>::iterator current = mp_work_edges->begin (); current != mp_work_edges->end (); ) {
      // 初始化当前分带信息，n 为当前分带坐标为 y 的边数量，yy 为当前分带 y 坐标的右侧边界
      size_t n = 0;
      db::Coord yy = y;

      //  Use as many scanlines as to fetch approx. 50% new edges into the scanline (this
      //  is an empirically determined factor)
      do {
        // 快速推进至下一个 y 坐标，即将当前 y 坐标的所有边加入分带
        while (future != mp_work_edges->end () && edge_ymin (*future) <= yy) {
          ++future;
        }

        if (future != mp_work_edges->end ()) {
          yy = edge_ymin (*future);
        } else {
          yy = std::numeric_limits <db::Coord>::max ();
        }

        if (n == 0) {
          n = std::distance (current, future);
        }
        // 确保当前分带的边的总数量为 1.5 倍 y 坐标下的边数量
      } while (future != mp_work_edges->end () && std::distance (current, future) < long (n * fill_factor));

      bool is90 = true;
      // 分带内进行下一步计算
      if (current != future) {

        for (std::vector <WorkEdge>::iterator c = current; c != future && is90; ++c) {
          if (c->dx () != 0 && c->dy () != 0) {
            is90 = false;
          }
        }

        if (is90) {
          get_intersections_per_band_90 (*mp_cpvector, current, future, y, yy, selects_edges);
        } else {
          get_intersections_per_band_any (*mp_cpvector, current, future, y, yy, selects_edges);
        }

      }

      // 完成当前分带相交判断后推进扫描线，将所有在扫描线之下的边移出
      y = yy;
      for (std::vector <WorkEdge>::iterator c = current; c != future; ++c) {
        //  Hint: we have to keep the edges ending a y (the new lower band limit) in the all angle case because these edges
        //  may receive cutpoints because the enter the -0.5DBU region below the band
        // 当前分带内所有在当前扫描线之下的边移除活动边集合（因为这些边已经在当前扫描线之下必定不可能和即将推进的扫描线及其以上的边相交）
        if ((!is90 && edge_ymax (*c) < y) || (is90 && edge_ymax (*c) <= y)) {
          if (current != c) {
            std::swap (*current, *c);
          }
          ++current;
        }
      }

    }
```

  * 当前分带内按照 x 坐标进行分块

```C++

static void
get_intersections_per_band_90(std::vector<CutPoints> &cutpoints,
                              std::vector<WorkEdge>::iterator current,
                              std::vector<WorkEdge>::iterator future,
                              db::Coord y, db::Coord yy, bool with_h) {
  std::sort(current, future, edge_xmin_compare<db::Coord>());

#ifdef DEBUG_EDGE_PROCESSOR
  printf("y=%d..%d (90 degree)\n", y, yy);
  printf("edges:");
  for (std::vector<WorkEdge>::iterator c1 = current; c1 != future; ++c1) {
    printf(" %s", c1->to_string().c_str());
  }
  printf("\n");
#endif
  // 初始化 x
  db::Coord x = edge_xmin(*current);

  std::vector<WorkEdge>::iterator f = current;
  // 相同策略在当前分带内进行块划分
  for (std::vector<WorkEdge>::iterator c = current; c != future;) {

    size_t n = 0;
    db::Coord xx = x;

    //  fetch as many cells as to fill in roughly 50% more
    //  (this is an empirical performance improvement factor)
    do {

      while (f != future && edge_xmin(*f) <= xx) {
        ++f;
      }

      if (f != future) {
        xx = edge_xmin(*f);
      } else {
        xx = std::numeric_limits<db::Coord>::max();
      }

      if (n == 0) {
        n = std::distance(c, f);
      }

    } while (f != future && std::distance(c, f) < long(n * fill_factor));

#ifdef DEBUG_EDGE_PROCESSOR
    printf("edges %d..%d:", x, xx);
    for (std::vector<WorkEdge>::iterator c1 = c; c1 != f; ++c1) {
      printf(" %s", c1->to_string().c_str());
    }
    printf("\n");
#endif

    if (std::distance(c, f) > 1) {

      db::Box cell(x, y, xx, yy);
      // 最终在当前分块内进行相交判断
      for (std::vector<WorkEdge>::iterator c1 = c; c1 != f; ++c1) {
      // 省略相交检查代码
    }

    // 右侧边界推进方式同 y 方向
    x = xx;
    for (std::vector<WorkEdge>::iterator cc = c; cc != f; ++cc) {
      if (edge_xmax(*cc) < x) {
        if (c != cc) {
          std::swap(*cc, *c);
        }
        ++c;
      }
    }
  }
}
```

### 水平边特殊处理

* 两条水平边：
  * y 相同：则两天边各自的端点可以作为另一条边的切点，避免正交求交点
  * y不相同：显然不相交
* 水平与垂直：直接求交点，对于 Manhattan 情况，可以简化为坐标比较
* 两条垂直边：处理方法类似两条水平边，只是按照 x 是否相同进行处理

```C++

        bool c1p1_in_cell = cell.contains(c1->p1());
        bool c1p2_in_cell = cell.contains(c1->p2());

        for (std::vector<WorkEdge>::iterator c2 = c; c2 != f; ++c2) {

          if (c1 == c2) {
            continue;
          }

          if (c2->dy() == 0) {

            if ((with_h || c1->dy() != 0) && c1 < c2) {

              if (c1->dy() == 0) {

                //  parallel horizontal edges: produce the end points of each
                //  other edge as cutpoints
                if (c1->p1().y() == c2->p1().y()) {
                  add_hparallel_cutpoints(*c1, *c2, cell, cutpoints);
                  add_hparallel_cutpoints(*c2, *c1, cell, cutpoints);
                }

              } else if (c1->p1() != c2->p1() && c1->p2() != c2->p1() &&
                         c1->p1() != c2->p2() && c1->p2() != c2->p2()) {

                std::pair<bool, db::Point> cp = c1->intersect_point(*c2);
                if (cp.first && cell.contains(cp.second)) {

                  //  add a cut point to c1 and c2 (c2 only if necessary)
                  c1->make_cutpoints(cutpoints)->add(cp.second, &cutpoints,
                                                     true);
                  if (with_h) {
                    c2->make_cutpoints(cutpoints)->add(cp.second, &cutpoints,
                                                       true);
                  }

#ifdef DEBUG_EDGE_PROCESSOR
                  printf("intersection point %s between %s and %s (1).\n",
                         cp.second.to_string().c_str(), c1->to_string().c_str(),
                         c2->to_string().c_str());
#endif
                }
              }
            }

          } else if (c1->dy() == 0) {

            if (c1 < c2 && c1->p1() != c2->p1() && c1->p2() != c2->p1() &&
                c1->p1() != c2->p2() && c1->p2() != c2->p2()) {

              std::pair<bool, db::Point> cp = c1->intersect_point(*c2);
              if (cp.first && cell.contains(cp.second)) {

                //  add a cut point to c1 and c2
                c2->make_cutpoints(cutpoints)->add(cp.second, &cutpoints, true);
                if (with_h) {
                  c1->make_cutpoints(cutpoints)->add(cp.second, &cutpoints,
                                                     true);
                }

#ifdef DEBUG_EDGE_PROCESSOR
                printf("intersection point %s between %s and %s (2).\n",
                       cp.second.to_string().c_str(), c1->to_string().c_str(),
                       c2->to_string().c_str());
#endif
              }
            }

          } else if (c1->p1().x() == c2->p1().x()) {

            //  both edges are coincident - produce the ends of the edges
            //  involved as cut points
            if (c1p1_in_cell && c1->p1().y() > db::edge_ymin(*c2) &&
                c1->p1().y() < db::edge_ymax(*c2)) {
              c2->make_cutpoints(cutpoints)->add(c1->p1(), &cutpoints, true);
            }
            if (c1p2_in_cell && c1->p2().y() > db::edge_ymin(*c2) &&
                c1->p2().y() < db::edge_ymax(*c2)) {
              c2->make_cutpoints(cutpoints)->add(c1->p2(), &cutpoints, true);
            }
          }
        }
      }
```

## 切点处理

一些优化技巧:

* 按照投影排序：这能保证在边上的切点顺序最终是沿着边的顺序
* 避免 Z 型边：事实上是通过点的坐标避免产生新边时出现“折返”情况，这可能造成当前边产生自交或在其他分块中产生新交点
* 原地压缩 workedge（in place）：通过 nw 索引与 n 之间的关系使得尽可能保证内存在原地不进行额外移动与复制

```C++
    //  step 3: create new edges from the ones with cutpoints
    //
    //  Hint: when we create the edges from the cutpoints we use the projection to sort the cutpoints along the
    //  edge. However, we have some freedom to connect the points which we use to avoid "z" configurations which could
    //  create new intersections in a 1x1 pixel box.

    todo = todo_next;
    todo_next += (todo_max - todo) / 5;

    size_t n_work = mp_work_edges->size ();
    // 索引 nw 为目标写入位置（新生成的边的写入位置）
    size_t nw = 0;
    // 遍历每条 workedge
    for (size_t n = 0; n < n_work; ++n) {

      WorkEdge &ew = (*mp_work_edges) [n];
      // 对于当前edge ew，尝试获取切点后，通过赋值 0 断开 cut point 索引
      CutPoints *cut_points = ew.data ? & ((*mp_cpvector) [ew.data - 1]) : 0;
      ew.data = 0;

      if (ew.dy () == 0 && ! selects_edges) {
        // 如果是水平边且 selects_edges false 时不保留水平边
        //  don't care about horizontal edges

      } else if (cut_points) {

        if (cut_points->has_cutpoints && ! cut_points->cut_points.empty ()) {

          db::Edge e = ew;
          property_type p = ew.prop;
          // 所有切点通过投影进行排序，确保顺序为沿着当前边 ew 的顺序
          std::sort (cut_points->cut_points.begin (), cut_points->cut_points.end (), ProjectionCompare (e));
          // 初始化前前点与前点
          db::Point pll = e.p1 ();
          db::Point pl = e.p1 ();
          // 遍历所有切点
          for (std::vector <db::Point>::iterator cp = cut_points->cut_points.begin (); cp != cut_points->cut_points.end (); ++cp) {
            if (*cp != pl) {
              // 当前的侯选边 ne: pl -- cp
              WorkEdge ne = WorkEdge (db::Edge (pl, *cp), p);
              // 避免生成 Z 型边（即确保生成的新边顺序一致且不会互相重合），举例如下：
              // c -------> d
              // c <-- b
              // a --> b
              if (pl.y () == pll.y () && ne.p2 ().x () != pl.x () && ne.p2 ().x () == pll.x ()) {
                // 水平方向上的 z 形边
                // 如果 pll -- pl 与 pl -- cp 的边情况满足如下：
                // cp <-- pl
                // pll --> pl
                // 则将当前 ne 改为从 pll --> cp
                ne = db::Edge (pll, ne.p2 ());
              } else if (pl.x () == pll.x () && ne.p2 ().y () != pl.y () && ne.p2 ().y () == pll.y ()) {
                // 竖直方向上的 z 形边
                ne = db::Edge (ne.p1 (), pll);
              } else {
                pll = pl;
              }
              // 更新前点
              pl = *cp;
              // 判断条件与前面相同，判断是否保留水平边或新边
              if (selects_edges || ne.dy () != 0) {
                if (nw <= n) {
                  (*mp_work_edges) [nw++] = ne;
                } else {
                  mp_work_edges->push_back (ne);
                }
              }
            }
          }
          // 最后一个切点处理，如果不是当前线段的终点，则最后一个切点与终点再构成一条新边
          if (cut_points->cut_points.back () != e.p2 ()) {
            WorkEdge ne = WorkEdge (db::Edge (pl, e.p2 ()), p);
            if (pl.y () == pll.y () && ne.p2 ().x () != pl.x () && ne.p2 ().x () == pll.x ()) {
              ne = db::Edge (pll, ne.p2 ());
            } else if (pl.x () == pll.x () && ne.p2 ().y () != pl.y () && ne.p2 ().y () == pll.y ()) {
              ne = db::Edge (ne.p1 (), pll);
            }
            if (selects_edges || ne.dy () != 0) {
              if (nw <= n) {
                (*mp_work_edges) [nw++] = ne;
              } else {
                mp_work_edges->push_back (ne);
              }
            }
          }

        } else {
          // 如果没有切点则推进 nw，并将原索引 n 对应的 workedge 推进至 nw

          if (nw < n) {
            (*mp_work_edges) [nw] = (*mp_work_edges) [n];
          }
          ++nw;

        }

      } else {
        // 如果没有切点则推进 nw，并将原索引 n 对应的 workedge 推进至 nw
        if (nw < n) {
          (*mp_work_edges) [nw] = (*mp_work_edges) [n];
        }
        ++nw;

      }

    }
    // 循环结束后，将多余元素删除，剩下的为新的 workedge 集合
    if (nw != n_work) {
      mp_work_edges->erase (mp_work_edges->begin () + nw, mp_work_edges->begin () + n_work);
    }

#ifdef DEBUG_EDGE_PROCESSOR
    printf ("Output edges:\n");
    for (std::vector <WorkEdge>::iterator c1 = mp_work_edges->begin (); c1 != mp_work_edges->end (); ++c1) {
      printf ("%s\n", c1->to_string().c_str ());
    }
#endif

```

## 边生成

通用的扫描线流程，只在每个交点作为事件驱动点去执行具体的处理

* 预处理：
  * 按照 ymin 进行边排序
  * 初始化起始扫描线y
  * 初始化future作为当前扫描线上遍历到的最后一条有效边，即当前扫描线y>= future->y_min
* 主循环
  * 
* 推进future
* 计算下一事件，即推进y至yy
* 扫描线推进

```C++

  //  step 4: compute the result edges

  gs.start(); // call this as late as possible. This way, input containers can
              // be identical with output containers ("clear" is done after the
              // input is read)

  gs.reset();
  gs.reserve(n_props);

  std::sort(mp_work_edges->begin(), mp_work_edges->end(),
            edge_ymin_compare<db::Coord>());
  // 初始化当前扫描线 y
  y = edge_ymin((*mp_work_edges)[0]);
  // 初始化 future 含义为与当前扫描线相关的边集合的右侧边界
  future = mp_work_edges->begin();
  for (std::vector<WorkEdge>::iterator current = mp_work_edges->begin();
       current != mp_work_edges->end() && !gs.can_stop();) {

    if (m_report_progress) {
      double p = double(std::distance(mp_work_edges->begin(), current)) /
                 double(mp_work_edges->size());
      progress->set(size_t(double(todo_max - todo_next) * p) + todo_next);
    }
    // 推进 future，future y_min 小于当前扫描线
    std::vector<WorkEdge>::iterator f0 = future;
    while (future != mp_work_edges->end() && edge_ymin(*future) <= y) {
      tl_assert(future->data == 0); // HINT: for development
      ++future;
    }
    // 按照 x 在当前扫描线的投影进行排序
    std::sort(f0, future, EdgeXAtYCompare2(y));
    // 计算下一个事件点 yy，在当前区间内 [current, future) 中找到 ymax 大于 y 的最小值，代表着下一次扫描线将会推进至 yy
    db::Coord yy = std::numeric_limits<db::Coord>::max();
    if (future != mp_work_edges->end()) {
      yy = edge_ymin(*future);
    }
    for (std::vector<WorkEdge>::const_iterator c = current; c != future; ++c) {
      if (edge_ymax(*c) > y) {
        yy = std::min(yy, edge_ymax(*c));
      }
    }

    db::Coord ysl = y;
    gs.begin_scanline(y);

    tl_assert(gs.is_reset()); // HINT: for development
    // 确保有新边进入，否则直接跳过实际处理
    if (current != future) {
      // 将新增的边 [f0, future) 合并至已经有序的集合 [current, f0) 中
      std::inplace_merge(current, f0, future, EdgeXAtYCompare2(y));
#ifdef DEBUG_EDGE_PROCESSOR
      printf("y=%d ", y);
      for (std::vector<WorkEdge>::iterator c = current; c != future; ++c) {
        printf("%ld-", long(c->data));
      }
      printf("\n");
#endif
      // 处理当前扫描线的顶点事件
      for (std::vector<WorkEdge>::iterator c = current; c != future;) {
        // Interpret the WorkEdge::data field as a skip-entry index here.
        // If c->data == 0 no skip entry exists for this edge. The
        // EdgeProcessorStates (gs) stores per-interval skip information and
        // returns a small index that can be used to rapidly skip unchanged
        // intervals on subsequent scanlines via `gs.skip_info()`.
        const EdgeProcessorStates::SkipInfo &skip_info = gs.skip_info(c->data);
#ifdef DEBUG_EDGE_PROCESSOR
        printf("X %ld->%d\n", long(c->data), int(skip_info.skip));
#endif
        // skip_info.skip 表示能够跳过的连续边的数量
        if (skip_info.skip != 0 &&
            (c + skip_info.skip >= future || (c + skip_info.skip)->data != 0)) {

          tl_assert(c + skip_info.skip <= future);

          gs.skip_n(skip_info);

          //  skip this interval - has not changed
          c += skip_info.skip;

        } else {
          // 创建一个新的 skip interval
          std::vector<WorkEdge>::iterator c0 = c;
          gs.begin_skip_interval();

          do {

            gs.reset_skip_entry(c->data);

            std::vector<WorkEdge>::iterator f = c + 1;

            //  HINT: "volatile" forces x and xx into memory and disables FPU
            //  register optimisation. That way, we can exactly compare doubles
            //  afterwards.
            // 收集在当前扫描线上具有相同 x 投影的边
            volatile double x = edge_xaty(*c, y);
            // 将具有相同 x 投影的连续边分为一个 coincident 组，对其中每个成员重置其跳过缓存引用
            // 此时 [c, f) 是一个 coincident 组
            while (f != future) {
              volatile double xx = edge_xaty(*f, y);
              if (xx != x) {
                break;
              }
              gs.reset_skip_entry(f->data);
              ++f;
            }

            //  compute edges that occur at this vertex
            // 进入顶点事件
            gs.next_vertex(x);

            //  treat all edges crossing the scanline in a certain point
            // 处理当前 coincident 组内的边
            for (std::vector<WorkEdge>::iterator cc = c; cc != f;) {

              gs.next_coincident();

              std::vector<WorkEdge>::iterator e = mp_work_edges->end();

              std::vector<WorkEdge>::iterator cc0 = cc;

              std::vector<WorkEdge>::iterator fc = cc;
              do {
                ++fc;
              } while (fc != f && EdgeXAtYCompare2(y).equal(*fc, *cc));

              //  sort the coincident edges by property ID - that will
              //  simplify algorithms like "inside" and "outside".
              if (fc - cc > 1) {
                //  for prefer_touch we first deliver the opening edges in
                //  ascending order, in the other case we the other way round so
                //  that the opening edges are always delivered with ascending
                //  property ID order.
                if (prefer_touch) {
                  std::sort(cc, fc, EdgePropCompare());
                } else {
                  std::sort(cc, fc, EdgePropCompareReverse());
                }
              }

              //  treat all coincident edges
              do {

                if (cc->dy() != 0) {
                  // 非水平边
                  if (e == mp_work_edges->end() && edge_ymax(*cc) > y) {
                    // 若没有当前即将进入的活动边且该边与当前扫描线相交
                    // 则初始化当前 e
                    e = cc;
                  }

                  if ((cc->dy() > 0) == prefer_touch) {
                    // 边来自在扫描线上方
                    if (edge_ymax(*cc) > y) {
                      gs.north_edge(prefer_touch, cc->prop);
                    }
                    // 边来自在扫描线下方
                    if (edge_ymin(*cc) < y) {
                      gs.south_edge(prefer_touch, cc->prop);
                    }
                  }
                }

                ++cc;

              } while (cc != fc);

              //  Give the edge selection operator a chance to select edges now
              // 当前区间内交由算子决定如何选择边
              if (selects_edges) {
                for (std::vector<WorkEdge>::iterator sc = cc0; sc != fc; ++sc) {
                  if (edge_ymin(*sc) == y) {
                    gs.select_edge(*sc);
                  }
                }
              }

              //  report the closing or opening edges in the opposite order
              //  than the other ones (see previous loop). Hence we have some
              //  symmetry of events which simplify implementation of the
              //  InteractionDetector for example.
              do {

                --fc;

                if (fc->dy() != 0 && (fc->dy() > 0) != prefer_touch) {
                  if (edge_ymax(*fc) > y) {
                    gs.north_edge(!prefer_touch, fc->prop);
                  }
                  if (edge_ymin(*fc) < y) {
                    gs.south_edge(!prefer_touch, fc->prop);
                  }
                }

              } while (fc != cc0);
              // 结束当前 coincident 组
              gs.end_coincident();

              if (e != mp_work_edges->end()) {
                gs.push_edge(*e);
              }
            }
            // 顶点处理结束
            gs.end_vertex();
            // 推进至下一个顶点事件
            c = f;

          } while (c != future && !gs.is_reset());

          //  TODO: assert that there is no overflow here:
          // Store the skip-entry index returned by gs.end_skip_interval()
          // into the first WorkEdge of this interval. The non-zero returned
          // value denotes a skip entry that can be queried by later calls
          // to `gs.skip_info(data)` and used to skip over unchanged
          // intervals.
          c0->data = gs.end_skip_interval(std::distance(c0, c));
        }
      }
      // 推进下一个扫描线节点
      y = yy;

#ifdef DEBUG_EDGE_PROCESSOR
      for (std::vector<WorkEdge>::iterator c = current; c != future; ++c) {
        printf("%ld-", long(c->data));
      }
      printf("\n");
#endif
      std::vector<WorkEdge>::iterator c0 = current;
      current = future;

      bool valid = true;
      // 更新 work edges 集合，移除已经结束的边，并更新 skip info 引用
      // 将仍然活跃的边移动到 current 前面，由于 current
      // 也相应递减，因此下一个循环的活跃区间依旧为 [current, future)
      for (std::vector<WorkEdge>::iterator c = future; c != c0;) {

        --c;

        size_t data = c->data;
        c->data = 0;

        db::Coord ymax = edge_ymax(*c);
        // 保持活动边在 [current, future) 区间
        if (ymax >= y) {
          --current;
          if (current != c) {
            std::swap(*current, *c);
          }
        }
        // ymax 小于等于当前扫描线，说明该边已经结束（下一个活动区间随着扫描线上升将不再包含该边）
        if (ymax <= y) {
          //  an edge ends now. The interval is not valid, i.e. cannot be
          //  skipped easily.
          valid = false;
        }
        // Update skip-info references as edges are moved/removed. The
        // variable `data` holds a previously stored skip-entry index (or
        // zero). If the interval remains valid we transfer the index to
        // the new position (`current->data = data`) so the skip-optimization
        // can continue to be used. If the interval is invalid (e.g. an
        // edge ended at this scanline) we must release the skip entry via
        // `gs.release_skip_entry(data)` to avoid leaks.
        if (data != 0 && current != future) {
          if (valid) {
            current->data = data;
            data = 0;
          } else {
            current->data = 0;
          }
          valid = true;
        }

        if (data) {
          gs.release_skip_entry(data);
        }
      }
#ifdef DEBUG_EDGE_PROCESSOR
      for (std::vector<WorkEdge>::iterator c = current; c != future; ++c) {
        printf("%ld-", long(c->data));
      }
      printf("\n");
#endif
    }

    tl_assert(gs.is_reset()); // HINT: for development (second)
    // 当前扫描线完成
    gs.end_scanline(ysl);
  }

  gs.flush ();

```
