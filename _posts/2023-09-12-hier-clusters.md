---
layout: post
title: hier_clusters
date: 2023-09-12 14:09 +0800
tags: [EDA, Klayout, DB]
categories: [Klayout, DB]
author: Jebearssica
---

> 地狱般的代码, 所以请抱着批判的态度看下去.
{: .prompt-warning }

> 如果你了解[并查集]()的基本概念的话, 对于了解 `hier_clusters` 的部分算法有很大帮助.
{: .prompt-tip }

`hier_clusters` 很难通过几句话能够说出它的作用, 这里只试图探讨构造该数据结构的过程. 根据代码片段, 我们将其分为两部分.

## build local cluster

可以简单概括为, 通过并查集构造相连集合.

```c++
    //  first build all local clusters
    tl::SelfTimer timer (tl::verbosity () > m_base_verbosity + 10, tl::to_string (tr ("Computing local shape clusters")));
    tl::RelativeProgress progress (tl::to_string (tr ("Computing local clusters")), called.size (), 1);

    for (std::set<db::cell_index_type>::const_iterator c = called.begin (); c != called.end (); ++c) {

      //  look for the net label joining spec - for the top cell the "top_cell_index" entry is looked for.
      //  If there is no such entry or the cell is not the top cell, look for the entry by cell index.
      std::map<db::cell_index_type, tl::equivalence_clusters<size_t> >::const_iterator ae;
      const tl::equivalence_clusters<size_t> *ec = 0;
      if (attr_equivalence) {
        if (*c == cell.cell_index ()) {
          ae = attr_equivalence->find (top_cell_index);
          if (ae != attr_equivalence->end ()) {
            ec = &ae->second;
          }
        }
        if (! ec) {
          ae = attr_equivalence->find (*c);
          if (ae != attr_equivalence->end ()) {
            ec = &ae->second;
          }
        }
      }
      // 逐个计算每个 cell 的 cluster
      build_local_cluster (layout, layout.cell (*c), conn, ec, separate_attributes);

      ++progress;

    }
```

这里只简单的介绍一下 `equivalence_clusters`, 因为本文重点在 `hier_clusters` 对于 `merge` 的作用.

`equivalence_clusters` 正如其名, 可以视作 DSU 等价关系的一种实现. 在 `tl` 模块下的源码注释非常清除地描述了这个类的作用.

```c++
/**
 *  @brief A utility class forming clusters based on equivalence of a certain attribute
 *
 *  To use this class, feed it with equivalences using "is_same", e.g.
 *
 *  @code
 *  equivalence_clusters<int> eq;
 *  //  forms two clusters: 1,2,5 and 3,4
 *  eq.same (1, 2);
 *  eq.same (3, 4);
 *  eq.same (1, 5);
 *  @endcode
 *
 *  Self-equivalence is a way of introduction an attribute without
 *  an equivalence:
 *
 *  @code
 *  equivalence_clusters<int> eq;
 *  //  after this, 1 is a known attribute
 *  eq.same (1, 1);
 *  eq.has_attribute (1);  //  ->true
 *  @endcode
 *
 *  Equivalence clusters can be merged, forming new and bigger clusters.
 *
 *  Eventually, each cluster is represented by a non-zero integer ID.
 *  The cluster ID can obtained per attribute value using "cluster_id".
 *  In the above example this will be:
 *
 *  @code
 *  eq.cluster_id (1);  //  ->1
 *  eq.cluster_id (2);  //  ->1
 *  eq.cluster_id (3);  //  ->2
 *  eq.cluster_id (4);  //  ->2
 *  eq.cluster_id (5);  //  ->1
 *  eq.cluster_id (6);  //  ->0  (unknown)
 *  @endcode
 *
 *  "size" will give the maximum cluster ID.
 */
```

### `build_local_cluster`

这个函数实际只是一个非常简单的 wrapper function, 

```c++
template <class T>
void
hier_clusters<T>::build_local_cluster (const db::Layout &layout, const db::Cell &cell, const db::Connectivity &conn, const tl::equivalence_clusters<size_t> *attr_equivalence, bool separate_attributes)
{
  std::string msg = tl::to_string (tr ("Computing local clusters for cell: ")) + std::string (layout.cell_name (cell.cell_index ()));
  if (tl::verbosity () >= m_base_verbosity + 20) {
    tl::log << msg;
  }
  tl::SelfTimer timer (tl::verbosity () > m_base_verbosity + 20, msg);

  connected_clusters<T> &local = m_per_cell_clusters [cell.cell_index ()];
  local.build_clusters (cell, conn, attr_equivalence, true, separate_attributes);
}
```

### `build_clusters`

```c++
template <class T>
void
local_clusters<T>::build_clusters (const db::Cell &cell, const db::Connectivity &conn, const tl::equivalence_clusters<size_t> *attr_equivalence, bool report_progress, bool separate_attributes)
{
  static std::string desc = tl::to_string (tr ("Building local clusters"));

  db::box_scanner<T, std::pair<unsigned int, size_t> > bs (report_progress, desc);
  db::box_convert<T> bc;
  addressable_object_from_shape<T> heap;
  attr_accessor<T> attr;
  db::ShapeIterator::flags_type shape_flags = get_shape_flags<T> () ();
  // 这里可以视为 used layer, 即将该 Cell 中所有连接过的 layer 的 shapes 插入
  for (db::Connectivity::layer_iterator l = conn.begin_layers (); l != conn.end_layers (); ++l) {
    const db::Shapes &shapes = cell.shapes (*l);
    for (db::Shapes::shape_iterator s = shapes.begin (shape_flags); ! s.at_end (); ++s) {
      bs.insert (heap (*s), std::make_pair (*l, attr (*s)));
    }
  }

  cluster_building_receiver<T, box_type> rec (conn, attr_equivalence, separate_attributes);
  // 根据 shape bbox 相交进行 rec.add
  bs.process (rec, 1 /*==touching*/, bc);
  rec.generate_clusters (*this);

  if (attr_equivalence && attr_equivalence->size () > 0) {
    apply_attr_equivalences (*attr_equivalence);
  }
}
```

### `cluster_building_receiver`

用于在 `box_scanner` 中处理 bbox 相交的 shape pair 的类.

```c++
struct cluster_building_receiver
  : public db::box_scanner_receiver<T, std::pair<unsigned int, size_t> >
{
  typedef typename local_cluster<T>::id_type id_type;
  typedef std::pair<const T *, std::pair<unsigned int, size_t> > shape_value;
  typedef std::vector<shape_value> shape_vector;
  typedef std::set<size_t> global_nets;
  typedef std::pair<shape_vector, global_nets> cluster_value;

  cluster_building_receiver (const db::Connectivity &conn, const tl::equivalence_clusters<size_t> *attr_equivalence, bool separate_attributes)
    : mp_conn (&conn), mp_attr_equivalence (attr_equivalence), m_separate_attributes (separate_attributes)
  {
    if (mp_attr_equivalence) {
      //  cache the global nets per attribute equivalence cluster
      for (size_t gid = 0; gid < conn.global_nets (); ++gid) {
        tl::equivalence_clusters<size_t>::cluster_id_type cl = attr_equivalence->cluster_id (global_net_id_to_attr (gid));
        if (cl > 0) {
          m_global_nets_by_attribute_cluster [cl].insert (gid);
        }
      }
    }
  }

  void generate_clusters (local_clusters<T> &clusters)
  {
    //  build the resulting clusters
    for (typename std::list<cluster_value>::const_iterator c = m_clusters.begin (); c != m_clusters.end (); ++c) {

      //  TODO: reserve?
      local_cluster<T> *cluster = clusters.insert ();
      for (typename shape_vector::const_iterator s = c->first.begin (); s != c->first.end (); ++s) {
        cluster->add (*s->first, s->second.first);
        cluster->add_attr (s->second.second);
      }

      std::set<size_t> global_nets = c->second;

      //  Add the global nets we derive from the attribute equivalence (join_nets of labelled vs.
      //  global nets)
      if (mp_attr_equivalence) {

        for (typename shape_vector::const_iterator s = c->first.begin (); s != c->first.end (); ++s) {

          size_t a = s->second.second;
          if (a == 0) {
            continue;
          }

          tl::equivalence_clusters<size_t>::cluster_id_type cl = mp_attr_equivalence->cluster_id (a);
          if (cl > 0) {
            std::map<size_t, std::set<size_t> >::const_iterator gn = m_global_nets_by_attribute_cluster.find (cl);
            if (gn != m_global_nets_by_attribute_cluster.end ()) {
              global_nets.insert (gn->second.begin (), gn->second.end ());
            }
          }

        }

      }

      cluster->set_global_nets (global_nets);

    }
  }

  void add (const T *s1, std::pair<unsigned int, size_t> p1, const T *s2, std::pair<unsigned int, size_t> p2)
  {
    if (m_separate_attributes && p1.second != p2.second) {
      return;
    }
    if (! mp_conn->interacts (*s1, p1.first, *s2, p2.first)) {
      return;
    }

    typename std::map<const T *, typename std::list<cluster_value>::iterator>::iterator ic1 = m_shape_to_clusters.find (s1);
    typename std::map<const T *, typename std::list<cluster_value>::iterator>::iterator ic2 = m_shape_to_clusters.find (s2);

    if (ic1 == m_shape_to_clusters.end ()) {

      if (ic2 == m_shape_to_clusters.end ()) {

        m_clusters.push_back (cluster_value ());
        typename std::list<cluster_value>::iterator c = --m_clusters.end ();
        c->first.push_back (std::make_pair (s1, p1));
        c->first.push_back (std::make_pair (s2, p2));

        m_shape_to_clusters.insert (std::make_pair (s1, c));
        m_shape_to_clusters.insert (std::make_pair (s2, c));

      } else {

        ic2->second->first.push_back (std::make_pair (s1, p1));
        m_shape_to_clusters.insert (std::make_pair (s1, ic2->second));

      }

    } else if (ic2 == m_shape_to_clusters.end ()) {

      ic1->second->first.push_back (std::make_pair (s2, p2));
      m_shape_to_clusters.insert (std::make_pair (s2, ic1->second));

    } else if (ic1->second != ic2->second) {

      //  join clusters: use the larger one as the target

      if (ic1->second->first.size () < ic2->second->first.size ()) {
        join (ic2->second, ic1->second);
      } else {
        join (ic1->second, ic2->second);
      }

    }
  }

  void finish (const T *s, std::pair<unsigned int, size_t> p)
  {
    //  if the shape has not been handled yet, insert a single cluster with only this shape
    typename std::map<const T *, typename std::list<cluster_value>::iterator>::iterator ic = m_shape_to_clusters.find (s);
    if (ic == m_shape_to_clusters.end ()) {

      m_clusters.push_back (cluster_value ());
      typename std::list<cluster_value>::iterator c = --m_clusters.end ();
      c->first.push_back (std::make_pair (s, p));

      ic = m_shape_to_clusters.insert (std::make_pair (s, c)).first;

    }

    //  consider connections to global nets

    db::Connectivity::global_nets_iterator ge = mp_conn->end_global_connections (p.first);
    for (db::Connectivity::global_nets_iterator g = mp_conn->begin_global_connections (p.first); g != ge; ++g) {

      typename std::map<size_t, typename std::list<cluster_value>::iterator>::iterator icg = m_global_to_clusters.find (*g);

      if (icg == m_global_to_clusters.end ()) {

        ic->second->second.insert (*g);
        m_global_to_clusters.insert (std::make_pair (*g, ic->second));

      } else if (ic->second != icg->second) {

        //  join clusters
        if (ic->second->first.size () < icg->second->first.size ()) {
          join (icg->second, ic->second);
        } else {
          join (ic->second, icg->second);
        }

      }

    }
  }

private:
  const db::Connectivity *mp_conn;
  std::map<const T *, typename std::list<cluster_value>::iterator> m_shape_to_clusters;
  std::map<size_t, typename std::list<cluster_value>::iterator> m_global_to_clusters;
  std::list<cluster_value> m_clusters;
  std::map<size_t, std::set<size_t> > m_global_nets_by_attribute_cluster;
  const tl::equivalence_clusters<size_t> *mp_attr_equivalence;
  bool m_separate_attributes;

  void join (typename std::list<cluster_value>::iterator ic1, typename std::list<cluster_value>::iterator ic2)
  {
    ic1->first.insert (ic1->first.end (), ic2->first.begin (), ic2->first.end ());
    ic1->second.insert (ic2->second.begin (), ic2->second.end ());

    for (typename shape_vector::const_iterator i = ic2->first.begin (); i != ic2->first.end (); ++i) {
      m_shape_to_clusters [i->first] = ic1;
    }
    for (typename global_nets::const_iterator i = ic2->second.begin (); i != ic2->second.end (); ++i) {
      m_global_to_clusters [*i] = ic1;
    }

    m_clusters.erase (ic2);
  }
};
```

## build hierarchy connection

很难描述在干嘛, 我先看看

```c++
  //  build the hierarchical connections bottom-up and for all cells whose children are computed already

  instance_interaction_cache_type instance_interaction_cache;

  {
    tl::SelfTimer timer (tl::verbosity () > m_base_verbosity + 10, tl::to_string (tr ("Computing hierarchical shape clusters")));
    tl::RelativeProgress progress (tl::to_string (tr ("Computing hierarchical clusters")), called.size (), 1);

    std::set<db::cell_index_type> done;
    std::vector<db::cell_index_type> todo;
    for (db::Layout::bottom_up_const_iterator c = layout.begin_bottom_up (); c != layout.end_bottom_up (); ++c) {

      if (called.find (*c) != called.end ()) {

        bool all_available = true;
        const db::Cell &cell = layout.cell (*c);
        for (db::Cell::child_cell_iterator cc = cell.begin_child_cells (); ! cc.at_end () && all_available; ++cc) {
          all_available = (done.find (*cc) != done.end ());
        }

        if (all_available) {
          todo.push_back (*c);
        } else {
          tl_assert (! todo.empty ());
          build_hier_connections_for_cells (cbc, layout, todo, conn, breakout_cells, progress, instance_interaction_cache, separate_attributes);
          done.insert (todo.begin (), todo.end ());
          todo.clear ();
          todo.push_back (*c);
        }

      }

    }

    build_hier_connections_for_cells (cbc, layout, todo, conn, breakout_cells, progress, instance_interaction_cache, separate_attributes);
  }
```

### `build_hier_connections_for_cells`

仍然是一个 wrapper function

```c++
template <class T>
void
hier_clusters<T>::build_hier_connections_for_cells (cell_clusters_box_converter<T> &cbc, const db::Layout &layout, const std::vector<db::cell_index_type> &cells, const db::Connectivity &conn, const std::set<db::cell_index_type> *breakout_cells, tl::RelativeProgress &progress, instance_interaction_cache_type &instance_interaction_cache, bool separate_attributes)
{
  for (std::vector<db::cell_index_type>::const_iterator c = cells.begin (); c != cells.end (); ++c) {
    build_hier_connections (cbc, layout, layout.cell (*c), conn, breakout_cells, instance_interaction_cache, separate_attributes);
    ++progress;
  }
}
```

### `build_hier_connections`

```c++
template <class T>
void
hier_clusters<T>::build_hier_connections (cell_clusters_box_converter<T> &cbc, const db::Layout &layout, const db::Cell &cell, const db::Connectivity &conn, const std::set<db::cell_index_type> *breakout_cells, instance_interaction_cache_type &instance_interaction_cache, bool separate_attributes)
{
  std::string msg = tl::to_string (tr ("Computing hierarchical clusters for cell: ")) + std::string (layout.cell_name (cell.cell_index ()));
  if (tl::verbosity () >= m_base_verbosity + 20) {
    tl::log << msg;
  }
  tl::SelfTimer timer (tl::verbosity () > m_base_verbosity + 20, msg);

  connected_clusters<T> &local = m_per_cell_clusters [cell.cell_index ()];

  //  NOTE: this is a receiver for both the child-to-child and
  //  local to child interactions.
  std::unique_ptr<hc_receiver<T> > rec (new hc_receiver<T> (layout, cell, local, *this, cbc, conn, breakout_cells, &instance_interaction_cache, separate_attributes));
  cell_inst_clusters_box_converter<T> cibc (cbc);

  //  The box scanner needs pointers so we have to first store the instances
  //  delivered by the cell's iterator (which does not deliver real pointer).

  std::vector<db::Instance> inst_storage;

  //  TODO: there should be a cell.size () for this ...
  size_t n = 0;
  for (db::Cell::const_iterator inst = cell.begin (); ! inst.at_end (); ++inst) {
    n += 1;
  }

  inst_storage.reserve (n);
  for (db::Cell::const_iterator inst = cell.begin (); ! inst.at_end (); ++inst) {
    inst_storage.push_back (*inst);
  }

  //  handle instance to instance connections

  {
    static std::string desc = tl::to_string (tr ("Instance to instance treatment"));
    tl::SelfTimer timer (tl::verbosity () > m_base_verbosity + 30, desc);

    db::box_scanner<db::Instance, unsigned int> bs (true, desc);

    for (std::vector<db::Instance>::const_iterator inst = inst_storage.begin (); inst != inst_storage.end (); ++inst) {
      if (! is_breakout_cell (breakout_cells, inst->cell_index ())) {
        bs.insert (inst.operator-> (), 0);
      }
    }

    bs.process (*rec, 1 /*touching*/, cibc);
  }

  //  handle local to instance connections

  {
    std::list<local_cluster<T> > heap;
    double area_ratio = 10.0;

    static std::string desc = tl::to_string (tr ("Local to instance treatment"));
    tl::SelfTimer timer (tl::verbosity () > m_base_verbosity + 30, desc);

    db::box_scanner2<db::local_cluster<T>, unsigned int, db::Instance, unsigned int> bs2 (true, desc);

    for (typename connected_clusters<T>::const_iterator c = local.begin (); c != local.end (); ++c) {

      //  we do not actually need the original clusters. For a better performance we optimize the
      //  area ratio and split, but we keep the ID the same.
      std::back_insert_iterator<std::list<local_cluster<T> > > iout = std::back_inserter (heap);
      size_t n = c->split (area_ratio, iout);
      if (n == 0) {
        bs2.insert1 (c.operator-> (), 0);
      } else {
        typename std::list<local_cluster<T> >::iterator h = heap.end ();
        while (n-- > 0) {
          bs2.insert1 ((--h).operator-> (), 0);
        }
      }
    }

    for (std::vector<db::Instance>::const_iterator inst = inst_storage.begin (); inst != inst_storage.end (); ++inst) {
      if (! is_breakout_cell (breakout_cells, inst->cell_index ())) {
        bs2.insert2 (inst.operator-> (), 0);
      }
    }

    bs2.process (*rec, 1 /*touching*/, local_cluster_box_convert<T> (), cibc);
  }

  //  join local clusters which got connected by child clusters
  rec->finish_cluster_to_instance_interactions ();

  if (tl::verbosity () >= m_base_verbosity + 20) {
    tl::info << "Cluster build cache statistics (instance to shape cache): size=" << rec->cluster_cache_size () << ", hits=" << rec->cluster_cache_hits () << ", misses=" << rec->cluster_cache_misses ();
  }

  rec.reset (0);

  //  finally connect global nets
  {
    static std::string desc = tl::to_string (tr ("Global net treatment"));
    tl::SelfTimer timer (tl::verbosity () > m_base_verbosity + 30, desc);

    GlobalNetClusterMaker global_net_clusters;

    //  insert the global nets from the subcircuits which need connection

    for (std::vector<db::Instance>::const_iterator inst = inst_storage.begin (); inst != inst_storage.end (); ++inst) {

      const db::connected_clusters<T> &cc = m_per_cell_clusters [inst->cell_index ()];
      for (typename db::connected_clusters<T>::const_iterator cl = cc.begin (); cl != cc.end (); ++cl) {

        if (! cl->get_global_nets ().empty ()) {
          for (db::Instance::cell_inst_array_type::iterator i = inst->begin (); !i.at_end (); ++i) {
            global_net_clusters.add (cl->get_global_nets (), db::ClusterInstance (cl->id (), inst->cell_index (), inst->complex_trans (*i), inst->prop_id ()));
          }
        }

      }
    }

    //  insert the global nets from here

    for (typename db::connected_clusters<T>::const_iterator cl = local.begin (); cl != local.end (); ++cl) {
      const typename db::local_cluster<T>::global_nets &gn = cl->get_global_nets ();
      if (! gn.empty ()) {
        global_net_clusters.add (gn, db::ClusterInstance (cl->id ()));
      }
    }

    //  now global_net_clusters knows what clusters need to be made for the global nets

    for (GlobalNetClusterMaker::entry_iterator ge = global_net_clusters.begin (); ge != global_net_clusters.end (); ++ge) {

      db::local_cluster<T> *gc = local.insert ();
      gc->set_global_nets (ge->first);
      //  NOTE: don't use the gc pointer - it may become invalid during make_path (will also do a local.insert)
      size_t gcid = gc->id ();

      for (std::set<ClusterInstance>::const_iterator i = ge->second.begin (); i != ge->second.end (); ++i) {

        if (! i->has_instance ()) {

          local.join_cluster_with (gcid, i->id ());
          local.remove_cluster (i->id ());

        } else {

          //  ensures the cluster is propagated so we can connect it with another
          size_t other_id = propagate_cluster_inst (layout, cell, *i, cell.cell_index (), false);

          if (other_id == gcid) {
            //  shouldn't happen, but duplicate instances may trigger this
          } else if (other_id) {
            local.join_cluster_with (gcid, other_id);
            local.remove_cluster (other_id);
          } else {
            local.add_connection (gcid, *i);
          }

        }

      }

    }

  }
}
```

### `hc_receiver`

```c++
/**
 *  @brief The central interaction tester between clusters on a hierarchical level
 *
 *  This receiver is both used for the instance-to-instance and the local-to-instance
 *  interactions. It is employed on cell level for in two box scanners: one
 *  investigating the instance-to-instance interactions and another one investigating
 *  local cluster to instance interactions.
 */
template <class T>
struct hc_receiver
  : public db::box_scanner_receiver<db::Instance, unsigned int>,
    public db::box_scanner_receiver2<local_cluster<T>, unsigned int, db::Instance, unsigned int>
{
public:
  typedef typename hier_clusters<T>::box_type box_type;
  typedef typename local_cluster<T>::id_type id_type;
  typedef std::map<ClusterInstance, id_type> connector_map;

  struct ClusterInstanceInteraction
  {
    ClusterInstanceInteraction (size_t _cluster_id, const ClusterInstance &_other_ci)
      : cluster_id (_cluster_id), other_ci (_other_ci)
    { }

    bool operator== (const ClusterInstanceInteraction &other) const
    {
      return cluster_id == other.cluster_id && other_ci == other.other_ci;
    }

    size_t cluster_id;
    ClusterInstance other_ci;
  };

  /**
   *  @brief Constructor
   */
  hc_receiver (const db::Layout &layout, const db::Cell &cell, db::connected_clusters<T> &cell_clusters, hier_clusters<T> &tree, const cell_clusters_box_converter<T> &cbc, const db::Connectivity &conn, const std::set<db::cell_index_type> *breakout_cells, typename hier_clusters<T>::instance_interaction_cache_type *instance_interaction_cache, bool separate_attributes)
  {
    mp_layout = &layout;
    mp_cell = &cell;
    mp_tree = &tree;
    mp_cbc = &cbc;
    mp_conn = &conn;
    mp_breakout_cells = breakout_cells;
    m_cluster_cache_misses = 0;
    m_cluster_cache_hits = 0;
    mp_instance_interaction_cache = instance_interaction_cache;
    m_separate_attributes = separate_attributes;
    mp_cell_clusters = &cell_clusters;
  }

  /**
   *  @brief Gets the cache size
   */
  size_t cluster_cache_size () const
  {
    db::MemStatisticsSimple ms;
    ms << m_interaction_cache_for_clusters;
    return ms.used ();
  }

  /**
   *  @brief Gets the cache hits
   */
  size_t cluster_cache_hits () const
  {
    return m_cluster_cache_hits;
  }

  /**
   *  @brief Gets the cache misses
   */
  size_t cluster_cache_misses () const
  {
    return m_cluster_cache_misses;
  }

  /**
   *  @brief Receiver main event for instance-to-instance interactions
   */
  void add (const db::Instance *i1, unsigned int /*p1*/, const db::Instance *i2, unsigned int /*p2*/)
  {
    db::ICplxTrans t;

    std::list<std::pair<ClusterInstance, ClusterInstance> > ic;
    consider_instance_pair (box_type::world (), *i1, t, db::CellInstArray::iterator (), *i2, t, db::CellInstArray::iterator (), ic);

    connect_clusters (ic);
  }

  /**
   *  @brief Single-instance treatment - may be required because of interactions between array members
   */
  void finish (const db::Instance *i, unsigned int /*p1*/)
  {
    consider_single_inst (*i);
  }

  /**
   *  @brief Receiver main event for local-to-instance interactions
   */
  void add (const local_cluster<T> *c1, unsigned int /*p1*/, const db::Instance *i2, unsigned int /*p2*/)
  {
    std::list<ClusterInstanceInteraction> ic;

    db::ICplxTrans t;
    consider_cluster_instance_pair (*c1, *i2, t, ic);

    ic.unique ();
    m_ci_interactions.splice (m_ci_interactions.end (), ic, ic.begin (), ic.end ());
  }

  /**
   *  @brief Finally join the clusters in the join set
   *
   *  This step is postponed because doing this while the iteration happens would
   *  invalidate the box trees.
   */
  void finish_cluster_to_instance_interactions ()
  {
    for (typename std::list<ClusterInstanceInteraction>::const_iterator ii = m_ci_interactions.begin (); ii != m_ci_interactions.end (); ++ii) {
      mp_tree->propagate_cluster_inst (*mp_layout, *mp_cell, ii->other_ci, mp_cell->cell_index (), false);
    }

    for (typename std::list<ClusterInstanceInteraction>::const_iterator ii = m_ci_interactions.begin (); ii != m_ci_interactions.end (); ++ii) {

      id_type other = mp_cell_clusters->find_cluster_with_connection (ii->other_ci);
      if (other > 0) {

        //  we found a child cluster that connects two clusters on our own level:
        //  we must join them into one, but not now. We're still iterating and
        //  would invalidate the box trees. So keep this now and combine the clusters later.
        mark_to_join (other, ii->cluster_id);

      } else {
        mp_cell_clusters->add_connection (ii->cluster_id, ii->other_ci);
      }

    }

    for (typename std::list<std::set<id_type> >::const_iterator sc = m_cm2join_sets.begin (); sc != m_cm2join_sets.end (); ++sc) {

      if (sc->empty ()) {
        //  dropped ones are empty
        continue;
      }

      typename std::set<id_type>::const_iterator c = sc->begin ();
      typename std::set<id_type>::const_iterator cc = c;
      for (++cc; cc != sc->end (); ++cc) {
        mp_cell_clusters->join_cluster_with (*c, *cc);
      }

    }
  }

  //  needs explicit implementation because we have two base classes:
  bool stop () const { return false; }
  void initialize () { }
  void finalize (bool) { }

private:
  const db::Layout *mp_layout;
  const db::Cell *mp_cell;
  db::connected_clusters<T> *mp_cell_clusters;
  hier_clusters<T> *mp_tree;
  const cell_clusters_box_converter<T> *mp_cbc;
  const db::Connectivity *mp_conn;
  const std::set<db::cell_index_type> *mp_breakout_cells;
  typedef std::list<std::set<id_type> > join_set_list;
  std::map<id_type, typename join_set_list::iterator> m_cm2join_map;
  join_set_list m_cm2join_sets;
  std::list<ClusterInstanceInteraction> m_ci_interactions;
  instance_interaction_cache<interaction_key_for_clusters<box_type>, std::list<std::pair<size_t, size_t> > > m_interaction_cache_for_clusters;
  size_t m_cluster_cache_misses, m_cluster_cache_hits;
  typename hier_clusters<T>::instance_interaction_cache_type *mp_instance_interaction_cache;
  bool m_separate_attributes;

  /**
   *  @brief Investigate a pair of instances
   *
   *  @param common The common box of both instances
   *  @param i1 The first instance to investigate
   *  @param t1 The parent instances' cumulated transformation
   *  @param i1element selects a specific instance from i1 (unless == db::CellInstArray::iterator())
   *  @param i2 The second instance to investigate
   *  @param t2 The parent instances' cumulated transformation
   *  @param i2element selects a specific instance from i2 (unless == db::CellInstArray::iterator())
   *  @param interacting_clusters_out Receives the cluster interaction descriptors
   *
   *  "interacting_clusters_out" will be cluster interactions in the parent instance space of i1 and i2 respectively.
   *  Cluster ID will be valid in the parent cells containing i1 and i2.
   */
  void consider_instance_pair (const box_type &common,
                               const db::Instance &i1, const db::ICplxTrans &t1, const db::CellInstArray::iterator &i1element,
                               const db::Instance &i2, const db::ICplxTrans &t2, const db::CellInstArray::iterator &i2element,
                               cluster_instance_pair_list_type &interacting_clusters)
  {
    if (is_breakout_cell (mp_breakout_cells, i1.cell_index ()) || is_breakout_cell (mp_breakout_cells, i2.cell_index ())) {
      return;
    }

    box_type bb1 = (*mp_cbc) (i1.cell_index ());
    box_type b1 = i1.cell_inst ().bbox (*mp_cbc).transformed (t1);

    box_type bb2 = (*mp_cbc) (i2.cell_index ());
    box_type b2 = i2.cell_inst ().bbox (*mp_cbc).transformed (t2);

    box_type common_all = common & b1 & b2;

    if (common_all.empty ()) {
      return;
    }

    //  gross shortcut: the cells do no comprise a constellation which will ever interact
    if (! mp_conn->interact (mp_layout->cell (i1.cell_index ()), mp_layout->cell (i2.cell_index ()))) {
      return;
    }

    InstanceToInstanceInteraction ii_key;
    db::ICplxTrans i1t, i2t;
    bool fill_cache = false;

    size_t n1 = i1element.at_end () ? i1.size () : 1;
    size_t n2 = i2element.at_end () ? i2.size () : 1;

    //  Cache is only used for single instances or simple and regular arrays.
    if ((n1 == 1 || ! i1.is_iterated_array ()) &&
        (n2 == 1 || ! i2.is_iterated_array ())) {

      i1t = i1element.at_end () ? i1.complex_trans () : i1.complex_trans (*i1element);
      db::ICplxTrans tt1 = t1 * i1t;

      i2t = i2element.at_end () ? i2.complex_trans () : i2.complex_trans (*i2element);
      db::ICplxTrans tt2 = t2 * i2t;

      db::ICplxTrans cache_norm = tt1.inverted ();

      ii_key = InstanceToInstanceInteraction ((! i1element.at_end () || i1.size () == 1) ? 0 : i1.cell_inst ().delegate (),
                                              (! i2element.at_end () || i2.size () == 1) ? 0 : i2.cell_inst ().delegate (),
                                              cache_norm, cache_norm * tt2);

      const cluster_instance_pair_list_type *cached = mp_instance_interaction_cache->find (i1.cell_index (), i2.cell_index (), ii_key);
      if (cached) {

        //  use cached interactions
        interacting_clusters = *cached;
        for (cluster_instance_pair_list_type::iterator i = interacting_clusters.begin (); i != interacting_clusters.end (); ++i) {
          //  translate the property IDs
          i->first.set_inst_prop_id (i1.prop_id ());
          i->first.transform (i1t);
          i->second.set_inst_prop_id (i2.prop_id ());
          i->second.transform (i2t);
        }

        return;

      }

      //  avoid caching few-to-many interactions as this typically does not contribute anything
      fill_cache = true;

    }

    //  shortcut: if the instances cannot interact because their bounding boxes do no comprise a valid constellation,
    //  reject the pair

    bool any = false;
    for (db::Connectivity::layer_iterator la = mp_conn->begin_layers (); la != mp_conn->end_layers () && ! any; ++la) {
      db::box_convert<db::CellInst, true> bca (*mp_layout, *la);
      box_type bb1 = i1.cell_inst ().bbox (bca).transformed (t1);
      if (! bb1.empty ()) {
        db::Connectivity::layer_iterator lbe = mp_conn->end_connected (*la);
        for (db::Connectivity::layer_iterator lb = mp_conn->begin_connected (*la); lb != lbe && ! any; ++lb) {
          db::box_convert<db::CellInst, true> bcb (*mp_layout, *lb);
          box_type bb2 = i2.cell_inst ().bbox (bcb).transformed (t2);
          any = bb1.touches (bb2);
        }
      }
    }

    if (! any) {
      if (fill_cache) {
        mp_instance_interaction_cache->insert (i1.cell_index (), i2.cell_index (), ii_key);
      }
      return;
    }

    //  array interactions

    db::ICplxTrans t1i = t1.inverted ();
    db::ICplxTrans t2i = t2.inverted ();

    //  TODO: optimize for single inst?
    for (db::CellInstArray::iterator ii1 = i1element.at_end () ? i1.begin_touching (common_all.transformed (t1i), mp_layout) : i1element; ! ii1.at_end (); ++ii1) {

      db::ICplxTrans i1t = i1.complex_trans (*ii1);
      db::ICplxTrans tt1 = t1 * i1t;
      box_type ib1 = bb1.transformed (tt1);

      for (db::CellInstArray::iterator ii2 = i2element.at_end () ? i2.begin_touching (ib1.transformed (t2i), mp_layout) : i2element; ! ii2.at_end (); ++ii2) {

        db::ICplxTrans i2t = i2.complex_trans (*ii2);
        db::ICplxTrans tt2 = t2 * i2t;

        if (i1.cell_index () == i2.cell_index () && tt1 == tt2) {
          //  skip interactions between identical instances (duplicate instance removal)
          if (! i2element.at_end ()) {
            break;
          } else {
            continue;
          }
        }

        box_type ib2 = bb2.transformed (tt2);

        box_type common12 = ib1 & ib2 & common;
        if (! common12.empty ()) {

          const std::list<std::pair<size_t, size_t> > &i2i_interactions = compute_instance_interactions (common12, i1.cell_index (), tt1, i2.cell_index (), tt2);

          for (std::list<std::pair<size_t, size_t> >::const_iterator ii = i2i_interactions.begin (); ii != i2i_interactions.end (); ++ii) {
            ClusterInstance k1 (ii->first, i1.cell_index (), i1t, i1.prop_id ());
            ClusterInstance k2 (ii->second, i2.cell_index (), i2t, i2.prop_id ());
            interacting_clusters.push_back (std::make_pair (k1, k2));
          }

          //  dive into cell of ii2
          const db::Cell &cell2 = mp_layout->cell (i2.cell_index ());
          for (db::Cell::touching_iterator jj2 = cell2.begin_touching (common12.transformed (tt2.inverted ())); ! jj2.at_end (); ++jj2) {

            std::list<std::pair<ClusterInstance, ClusterInstance> > ii_interactions;
            consider_instance_pair (common12, i1, t1, ii1, *jj2, tt2, db::CellInstArray::iterator (), ii_interactions);

            for (std::list<std::pair<ClusterInstance, ClusterInstance> >::iterator i = ii_interactions.begin (); i != ii_interactions.end (); ++i) {
              propagate_cluster_inst (i->second, i2.cell_index (), i2t, i2.prop_id ());
            }

            ii_interactions.unique ();
            interacting_clusters.splice (interacting_clusters.end (), ii_interactions, ii_interactions.begin (), ii_interactions.end ());

          }

          //  dive into cell of ii1
          const db::Cell &cell1 = mp_layout->cell (i1.cell_index ());
          for (db::Cell::touching_iterator jj1 = cell1.begin_touching (common12.transformed (tt1.inverted ())); ! jj1.at_end (); ++jj1) {

            std::list<std::pair<ClusterInstance, ClusterInstance> > ii_interactions;
            consider_instance_pair (common12, *jj1, tt1, db::CellInstArray::iterator (), i2, t2, ii2, ii_interactions);

            for (std::list<std::pair<ClusterInstance, ClusterInstance> >::iterator i = ii_interactions.begin (); i != ii_interactions.end (); ++i) {
              propagate_cluster_inst (i->first, i1.cell_index (), i1t, i1.prop_id ());
            }

            ii_interactions.unique ();
            interacting_clusters.splice (interacting_clusters.end (), ii_interactions, ii_interactions.begin (), ii_interactions.end ());

          }

        }

        if (! i2element.at_end ()) {
          break;
        }

      }

      if (! i1element.at_end ()) {
        break;
      }

    }

    //  remove duplicates (after doing a quick unique before)
    //  NOTE: duplicates may happen due to manifold child/child interactions which boil down to
    //  identical parent cluster interactions.
    std::vector<std::pair<ClusterInstance, ClusterInstance> > sorted_interactions;
    sorted_interactions.reserve (interacting_clusters.size ());
    sorted_interactions.insert (sorted_interactions.end (), interacting_clusters.begin (), interacting_clusters.end ());
    interacting_clusters.clear ();
    std::sort (sorted_interactions.begin (), sorted_interactions.end ());
    sorted_interactions.erase (std::unique (sorted_interactions.begin (), sorted_interactions.end ()), sorted_interactions.end ());

    //  return the list of unique interactions
    interacting_clusters.insert (interacting_clusters.end (), sorted_interactions.begin (), sorted_interactions.end ());

    if (fill_cache && sorted_interactions.size () < instance_to_instance_cache_set_size_threshold) {

      //  normalize transformations for cache
      db::ICplxTrans i1ti = i1t.inverted (), i2ti = i2t.inverted ();
      for (std::vector<std::pair<ClusterInstance, ClusterInstance> >::iterator i = sorted_interactions.begin (); i != sorted_interactions.end (); ++i) {
        i->first.transform (i1ti);
        i->second.transform (i2ti);
      }

      cluster_instance_pair_list_type &cached = mp_instance_interaction_cache->insert (i1.cell_index (), i2.cell_index (), ii_key);
      cached.insert (cached.end (), sorted_interactions.begin (), sorted_interactions.end ());

    }
  }

  /**
   *  @brief Computes a list of interacting clusters for two instances
   */
  const std::list<std::pair<size_t, size_t> > &
  compute_instance_interactions (const box_type &common,
                                 db::cell_index_type ci1, const db::ICplxTrans &t1,
                                 db::cell_index_type ci2, const db::ICplxTrans &t2)
  {
    if (is_breakout_cell (mp_breakout_cells, ci1) || is_breakout_cell (mp_breakout_cells, ci2)) {
      static const std::list<std::pair<size_t, size_t> > empty;
      return empty;
    }

    db::ICplxTrans t1i = t1.inverted ();
    db::ICplxTrans t2i = t2.inverted ();
    db::ICplxTrans t21 = t1i * t2;

    box_type common2 = common.transformed (t2i);

    interaction_key_for_clusters<box_type> ikey (t1i, t21, common2);

    const std::list<std::pair<size_t, size_t> > *ici = m_interaction_cache_for_clusters.find (ci1, ci2, ikey);
    if (ici) {

      return *ici;

    } else {

      const db::Cell &cell2 = mp_layout->cell (ci2);

      const db::local_clusters<T> &cl1 = mp_tree->clusters_per_cell (ci1);
      const db::local_clusters<T> &cl2 = mp_tree->clusters_per_cell (ci2);

      std::list<std::pair<size_t, size_t> > &new_interactions = m_interaction_cache_for_clusters.insert (ci1, ci2, ikey);
      db::ICplxTrans t12 = t2i * t1;

      for (typename db::local_clusters<T>::touching_iterator i = cl1.begin_touching (common2.transformed (t21)); ! i.at_end (); ++i) {

        //  skip the test, if this cluster doesn't interact with the whole cell2
        if (! i->interacts (cell2, t21, *mp_conn)) {
          continue;
        }

        box_type bc2 = (common2 & i->bbox ().transformed (t12));
        for (typename db::local_clusters<T>::touching_iterator j = cl2.begin_touching (bc2); ! j.at_end (); ++j) {

          if ((! m_separate_attributes || i->same_attrs (*j)) && i->interacts (*j, t21, *mp_conn)) {
            new_interactions.push_back (std::make_pair (i->id (), j->id ()));
          }

        }

      }

      return new_interactions;

    }
  }

  /**
   *  @brief Single instance treatment
   */
  void consider_single_inst (const db::Instance &i)
  {
    if (is_breakout_cell (mp_breakout_cells, i.cell_index ())) {
      return;
    }

    box_type bb = (*mp_cbc) (i.cell_index ());
    const db::Cell &cell = mp_layout->cell (i.cell_index ());

    for (db::CellInstArray::iterator ii = i.begin (); ! ii.at_end (); ++ii) {

      db::ICplxTrans tt = i.complex_trans (*ii);
      box_type ib = bb.transformed (tt);

      std::vector<ClusterInstElement> pp;
      pp.push_back (ClusterInstElement (i.cell_index (), i.complex_trans (*ii), i.prop_id ()));

      bool any = false;

      for (db::CellInstArray::iterator ii2 = i.begin_touching (ib, mp_layout); ! ii2.at_end (); ++ii2) {

        db::ICplxTrans tt2 = i.complex_trans (*ii2);
        if (tt.equal (tt2)) {
          //  skip the initial instance
          continue;
        }

        box_type ib2 = bb.transformed (tt2);

        if (ib.touches (ib2)) {

          std::vector<ClusterInstElement> pp2;
          pp2.push_back (ClusterInstElement (i.cell_index (), i.complex_trans (*ii2), i.prop_id ()));

          cluster_instance_pair_list_type interacting_clusters;

          box_type common = (ib & ib2);
          const std::list<std::pair<size_t, size_t> > &i2i_interactions = compute_instance_interactions (common, i.cell_index (), tt, i.cell_index (), tt2);
          for (std::list<std::pair<size_t, size_t> >::const_iterator ii = i2i_interactions.begin (); ii != i2i_interactions.end (); ++ii) {
            ClusterInstance k1 (ii->first, i.cell_index (), tt, i.prop_id ());
            ClusterInstance k2 (ii->second, i.cell_index (), tt2, i.prop_id ());
            interacting_clusters.push_back (std::make_pair (k1, k2));
          }

          for (db::Cell::touching_iterator jj2 = cell.begin_touching (common.transformed (tt2.inverted ())); ! jj2.at_end (); ++jj2) {

            db::ICplxTrans t;

            std::list<std::pair<ClusterInstance, ClusterInstance> > ii_interactions;
            consider_instance_pair (common, i, t, ii, *jj2, tt2, db::CellInstArray::iterator (), ii_interactions);

            for (std::list<std::pair<ClusterInstance, ClusterInstance> >::iterator ii = ii_interactions.begin (); ii != ii_interactions.end (); ++ii) {
              propagate_cluster_inst (ii->second, i.cell_index (), tt2, i.prop_id ());
            }

            interacting_clusters.splice (interacting_clusters.end (), ii_interactions, ii_interactions.begin (), ii_interactions.end ());

          }

          connect_clusters (interacting_clusters);

          any = true;

        }

      }

      //  we don't expect more to happen on the next instance
      if (! any) {
        break;
      }

    }
  }

  /**
   *  @brief Handles a local clusters vs. the clusters of a specific child instance or instance array
   *  @param c1 The local cluster
   *  @param i2 The index of the child cell
   *  @param t2 The accumulated transformation of the path, not including i2
   *  @param interactions_out Delivers the interactions
   */
  void consider_cluster_instance_pair (const local_cluster<T> &c1, const db::Instance &i2, const db::ICplxTrans &t2, std::list<ClusterInstanceInteraction> &interactions_out)
  {
    if (is_breakout_cell (mp_breakout_cells, i2.cell_index ())) {
      return;
    }

    box_type bb2 = (*mp_cbc) (i2.cell_index ());

    const db::Cell &cell2 = mp_layout->cell (i2.cell_index ());

    box_type b1 = c1.bbox ();
    box_type b2 = i2.cell_inst ().bbox (*mp_cbc).transformed (t2);

    if (! b1.touches (b2)) {
      return;
    }

    std::vector<std::pair<size_t, size_t> > c2i_interactions;

    for (db::CellInstArray::iterator ii2 = i2.begin_touching ((b1 & b2).transformed (t2.inverted ()), mp_layout); ! ii2.at_end (); ++ii2) {

      db::ICplxTrans i2t = i2.complex_trans (*ii2);
      db::ICplxTrans tt2 = t2 * i2t;
      box_type ib2 = bb2.transformed (tt2);

      if (b1.touches (ib2) && c1.interacts (cell2, tt2, *mp_conn)) {

        c2i_interactions.clear ();
        compute_cluster_instance_interactions (c1, i2.cell_index (), tt2, c2i_interactions);

        for (std::vector<std::pair<size_t, size_t> >::const_iterator ii = c2i_interactions.begin (); ii != c2i_interactions.end (); ++ii) {
          ClusterInstance k (ii->second, i2.cell_index (), i2t, i2.prop_id ());
          interactions_out.push_back (ClusterInstanceInteraction (ii->first, k));
        }

        //  dive into cell of ii2
        for (db::Cell::touching_iterator jj2 = cell2.begin_touching ((b1 & ib2).transformed (tt2.inverted ())); ! jj2.at_end (); ++jj2) {

          std::list<ClusterInstanceInteraction> ci_interactions;
          consider_cluster_instance_pair (c1, *jj2, tt2, ci_interactions);

          for (typename std::list<ClusterInstanceInteraction>::iterator ii = ci_interactions.begin (); ii != ci_interactions.end (); ++ii) {
            propagate_cluster_inst (ii->other_ci, i2.cell_index (), i2t, i2.prop_id ());
          }

          interactions_out.splice (interactions_out.end (), ci_interactions, ci_interactions.begin (), ci_interactions.end ());

        }

      }

    }
  }

  /**
   *  @brief Handles a local clusters vs. the clusters of a specific child instance
   *  @param c1 The local cluster
   *  @param ci2 The cell index of the cell investigated
   *  @param p2 The instantiation path to the child cell (last element is the instance to ci2)
   *  @param t2 The accumulated transformation of the path
   */
  void
  compute_cluster_instance_interactions (const local_cluster<T> &c1,
                                         db::cell_index_type ci2, const db::ICplxTrans &t2,
                                         std::vector<std::pair<size_t, size_t> > &interactions)
  {
    if (is_breakout_cell (mp_breakout_cells, ci2)) {
      return;
    }

    const db::local_clusters<T> &cl2 = mp_tree->clusters_per_cell (ci2);

    for (typename db::local_clusters<T>::touching_iterator j = cl2.begin_touching (c1.bbox ().transformed (t2.inverted ())); ! j.at_end (); ++j) {

      if ((! m_separate_attributes || c1.same_attrs (*j)) && c1.interacts (*j, t2, *mp_conn)) {
        interactions.push_back (std::make_pair (c1.id (), j->id ()));
      }

    }
  }

  /**
   *  @brief Inserts a pair of clusters to join
   */
  void mark_to_join (id_type a, id_type b)
  {
    if (a == b) {
      //  shouldn't happen, but duplicate instances may trigger this
      return;
    }

    typename std::map<id_type, typename join_set_list::iterator>::const_iterator x = m_cm2join_map.find (a);
    typename std::map<id_type, typename join_set_list::iterator>::const_iterator y = m_cm2join_map.find (b);

    if (x == m_cm2join_map.end ()) {

      if (y == m_cm2join_map.end ()) {

        m_cm2join_sets.push_back (std::set<id_type> ());
        m_cm2join_sets.back ().insert (a);
        m_cm2join_sets.back ().insert (b);

        m_cm2join_map [a] = --m_cm2join_sets.end ();
        m_cm2join_map [b] = --m_cm2join_sets.end ();

      } else {

        y->second->insert (a);
        m_cm2join_map [a] = y->second;

      }

    } else if (y == m_cm2join_map.end ()) {

      x->second->insert (b);
      m_cm2join_map [b] = x->second;

    } else if (x->second != y->second) {

      //  join two superclusters
      typename join_set_list::iterator yset = y->second;
      x->second->insert (yset->begin (), yset->end ());
      for (typename std::set<id_type>::const_iterator i = yset->begin (); i != yset->end (); ++i) {
        m_cm2join_map [*i] = x->second;
      }
      m_cm2join_sets.erase (yset);

    }

#if defined(DEBUG_HIER_NETWORK_PROCESSOR)
    //  concistency check for debugging
    for (typename std::map<id_type, typename join_set_list::iterator>::const_iterator j = m_cm2join_map.begin (); j != m_cm2join_map.end (); ++j) {
      tl_assert (j->second->find (j->first) != j->second->end ());
    }

    for (typename std::list<std::set<id_type> >::const_iterator i = m_cm2join_sets.begin (); i != m_cm2join_sets.end (); ++i) {
      for (typename std::set<id_type>::const_iterator j = i->begin(); j != i->end(); ++j) {
        tl_assert(m_cm2join_map.find (*j) != m_cm2join_map.end ());
        tl_assert(m_cm2join_map[*j] == i);
      }
    }

    //  the sets must be disjunct
    std::set<id_type> all;
    for (typename std::list<std::set<id_type> >::const_iterator i = m_cm2join_sets.begin (); i != m_cm2join_sets.end (); ++i) {
      for (typename std::set<id_type>::const_iterator j = i->begin(); j != i->end(); ++j) {
        tl_assert(all.find (*j) == all.end());
        all.insert(*j);
      }
    }
#endif

  }

  /**
   *  @brief Makes cluster connections from all parents of a cell into this cell and to the given cluster
   *
   *  After calling this method, the cluster instance in ci is guaranteed to have connections from all
   *  parent cells. One of these connections represents the instance ci.
   *
   *  Returns false if the connection was already there in the same place (indicating duplicate instances).
   *  In this case, the cluster instance should be skipped. In the other case, the cluster instance is
   *  updated to reflect the connected cluster.
   */
  void propagate_cluster_inst (ClusterInstance &ci, db::cell_index_type pci, const db::ICplxTrans &trans, db::properties_id_type prop_id) const
  {
    size_t id_new = mp_tree->propagate_cluster_inst (*mp_layout, *mp_cell, ci, pci, true);
    tl_assert (id_new != 0);
    ci = db::ClusterInstance (id_new, pci, trans, prop_id);
  }

  /**
   *  @brief Establishes connections between the cluster instances listed in the argument
   */
  void connect_clusters (const cluster_instance_pair_list_type &interacting_clusters)
  {
    for (cluster_instance_pair_list_type::const_iterator ic = interacting_clusters.begin (); ic != interacting_clusters.end (); ++ic) {

      const ClusterInstance &k1 = ic->first;
      const ClusterInstance &k2 = ic->second;

      //  Note: "with_self" is false as we're going to create a connected cluster anyway
      id_type x1 = mp_tree->propagate_cluster_inst (*mp_layout, *mp_cell, k1, mp_cell->cell_index (), false);
      id_type x2 = mp_tree->propagate_cluster_inst (*mp_layout, *mp_cell, k2, mp_cell->cell_index (), false);

      if (x1 == 0) {

        if (x2 == 0) {

          id_type connector = mp_cell_clusters->insert_dummy ();
          mp_cell_clusters->add_connection (connector, k1);
          mp_cell_clusters->add_connection (connector, k2);

        } else {
          mp_cell_clusters->add_connection (x2, k1);
        }

      } else if (x2 == 0) {

        mp_cell_clusters->add_connection (x1, k2);

      } else if (x1 != x2) {

        //  for instance-to-instance interactions the number of connections is more important for the
        //  cost of the join operation: make the one with more connections the target
        //  TODO: this will be SLOW for STL's not providing a fast size()
        if (mp_cell_clusters->connections_for_cluster (x1).size () < mp_cell_clusters->connections_for_cluster (x2).size ()) {
          std::swap (x1, x2);
        }

        mp_cell_clusters->join_cluster_with (x1, x2);
        mp_cell_clusters->remove_cluster (x2);

      }

    }
  }
};
```
