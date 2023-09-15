---
layout: post
title: InstanceSetCompareFunction
date: 2023-08-18 15:25 +0800
---


```c++
// -------------------------------------------------------------------------------------
//  Some utility class: a compare function for a instance set of two cells in the context
//  of two layouts and two initial cells.

class InstanceSetCompareFunction 
{
public:
  typedef std::multiset<db::ICplxTrans, db::trans_less_func<db::ICplxTrans> > trans_set_t;

  InstanceSetCompareFunction (const db::Layout &layout_a, db::cell_index_type initial_cell_a, const db::Layout &layout_b, db::cell_index_type initial_cell_b)
    : m_layout_a (layout_a), m_initial_cell_a (initial_cell_a),
      m_layout_b (layout_b), m_initial_cell_b (initial_cell_b),
      m_cell_a (std::numeric_limits<db::cell_index_type>::max ()),
      m_repr_set (false)
  {
    // ..
  }

  bool compare (db::cell_index_type cell_a, const std::set<db::cell_index_type> &selection_cone_a, db::cell_index_type cell_b, const std::set<db::cell_index_type> &selection_cone_b)
  {
    if (cell_a != m_cell_a) { // 对于连续比较 cell_a，可以跳过这段避免重复收集 called cell

      m_cell_a = cell_a;

      m_callers_a.clear ();
      m_layout_a.cell (cell_a).collect_caller_cells (m_callers_a, selection_cone_a, -1); // collect parent cell, since it is a caller not called function
      m_callers_a.insert (cell_a);

      m_trans.clear ();
      insert (m_layout_a, m_initial_cell_a, m_cell_a, m_callers_a, m_trans, db::ICplxTrans ()); // if contains cell_a, then insert trans into m_trans

    }

    m_repr_set = false;

    std::map<db::cell_index_type, db::ICplxTrans>::const_iterator r = m_repr.find (cell_b);
    if (r != m_repr.end ()) {
      m_repr_set = true;
      if (m_trans.find (r->second) == m_trans.end ()) {
        return false;
      }
    }

    std::set<db::cell_index_type> callers_b;
    m_layout_b.cell (cell_b).collect_caller_cells (callers_b, selection_cone_b, -1);
    callers_b.insert (cell_b);

    trans_set_t trans (m_trans);

    double mag = m_layout_b.dbu () / m_layout_a.dbu ();
    if (! compare (m_layout_b, m_initial_cell_b, cell_b, callers_b, trans, db::ICplxTrans (mag), db::ICplxTrans (1.0 / mag))) {
      return false;
    }

    return trans.empty ();
  }

private:
  const db::Layout &m_layout_a;
  db::cell_index_type m_initial_cell_a;
  const db::Layout &m_layout_b;
  db::cell_index_type m_initial_cell_b;
  db::cell_index_type m_cell_a;
  std::set<db::cell_index_type> m_callers_a;
  trans_set_t m_trans;
  std::map<db::cell_index_type, db::ICplxTrans> m_repr;
  bool m_repr_set;

  void insert (const db::Layout &layout, db::cell_index_type current_cell, db::cell_index_type cell, const std::set<db::cell_index_type> &cone, trans_set_t &trans, const db::ICplxTrans &current_trans)
  {
    if (current_cell == cell) { // if caller cell contains 
      trans.insert (current_trans); // WTF? passing a member object in a member function?
    } else {

      const db::Cell &cc = layout.cell (current_cell);
      size_t instances = cc.cell_instances ();
      SortedCellIndexIterator begin (cc, 0);
      SortedCellIndexIterator end (cc, instances);

      SortedCellIndexIterator i = begin;
      for (std::set<db::cell_index_type>::const_iterator c = cone.begin (); c != cone.end () && i != end; ++c) {
        if (*i <= *c) {
          for (i = std::lower_bound (i, end, *c); i != end && *i == *c; ++i) {
            for (db::CellInstArray::iterator arr = i.instance ().begin (); ! arr.at_end (); ++arr) {
              insert (layout, *c, cell, cone, trans, current_trans * i.instance ().complex_trans (*arr));
            }
          }
        }
      }

    }
  }

  bool compare (const db::Layout &layout, db::cell_index_type current_cell, db::cell_index_type cell, const std::set<db::cell_index_type> &cone, trans_set_t &trans, const db::ICplxTrans &current_trans, const db::ICplxTrans &local_trans)
  {
    if (current_cell == cell) {

      db::ICplxTrans eff_trans (current_trans * local_trans);

      if (! m_repr_set) {
        m_repr_set = true;
        m_repr.insert (std::make_pair (cell, eff_trans));
      }

      trans_set_t::iterator t = trans.find (eff_trans);
      if (t == trans.end ()) {
        return false;
      } else {
        trans.erase (t);
        return true;
      }

    } else {

      const db::Cell &cc = layout.cell (current_cell);
      size_t instances = cc.cell_instances ();
      SortedCellIndexIterator begin (cc, 0);
      SortedCellIndexIterator end (cc, instances);

      SortedCellIndexIterator i = begin;
      for (std::set<db::cell_index_type>::const_iterator c = cone.begin (); c != cone.end () && i != end; ++c) {
        if (*i <= *c) {
          for (i = std::lower_bound (i, end, *c); i != end && *i == *c; ++i) {
            for (db::CellInstArray::iterator arr = i.instance ().begin (); ! arr.at_end (); ++arr) {
              if (! compare (layout, *c, cell, cone, trans, current_trans * i.instance ().complex_trans (*arr), local_trans)) {
                return false;
              }
            }
          }
        }
      }

      return true;

    }
  }
};
```
