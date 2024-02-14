---
layout: post
title: BoxTree in Klayout
date: 2023-10-17 15:40 +0800
tags: [EDA, Klayout, Multithread, DB]
categories: [Klayout, DB]
author: Jebearssica
---

## `box_tree_node`

一眼四叉树的树结点数据结构

```c++
/**
 *  @brief The node object
 */

template <class Tree>
class box_tree_node
{
public:
  typedef typename Tree::point_type point_type;
  typedef typename Tree::coord_type coord_type;
  typedef typename Tree::box_type box_type;

  box_tree_node (box_tree_node *parent, const point_type &center, const box_type &qbox, unsigned int quad)
  {
    point_type corner;
    if (quad == 0) {
      corner = qbox.upper_right ();
    } else if (quad == 1) {
      corner = qbox.upper_left ();
    } else if (quad == 2) {
      corner = qbox.lower_left ();
    } else if (quad == 3) {
      corner = qbox.lower_right ();
    }

    init (parent, center, corner, quad);
  }

  ~box_tree_node ()
  {
    for (int i = 0; i < 4; ++i) {
      box_tree_node *c = child (i);
      if (c) {
        delete c;
      }
    }
  }

  box_tree_node *clone (box_tree_node *parent = 0, unsigned int quad = 0) const
  {
    box_tree_node *n = new box_tree_node (parent, m_center, m_corner, quad);
    n->m_lenq = m_lenq;
    n->m_len = m_len;
    for (unsigned int i = 0; i < 4; ++i) {
      box_tree_node *c = child (i);
      if (c) {
        c->clone (n, i);
      } else {
        n->m_childrefs [i] = m_childrefs [i];
      }
    }
    return n;
  }

  box_tree_node *child (int i) const
  {
    if ((m_childrefs [i] & 1) == 0) {
      return reinterpret_cast<box_tree_node *> (m_childrefs [i]);
    } else {
      return 0;
    }
  }

  void lenq (int i, size_t l) 
  {
    if (i < 0) {
      m_lenq = l;
    } else {
      box_tree_node *c = child (i);
      if (c) {
        c->m_len = l;
      } else {
        m_childrefs [i] = l * 2 + 1;
      }
    }
  }

  size_t lenq (int i) const
  {
    if (i < 0) {
      return m_lenq;
    } else {
      box_tree_node *c = child (i);
      if (c) {
        return c->m_len;
      } else {
        return m_childrefs [i] >> 1;
      }
    }
  }

  box_tree_node *parent () const
  {
    return (box_tree_node *)((char *) mp_parent - (size_t (mp_parent) & 3));
  }

  int quad () const
  {
    return (int)(size_t (mp_parent) & 3);
  }

  void mem_stat (MemStatistics *stat, MemStatistics::purpose_t purpose, int cat, bool no_self, void *parent)
  {
    if (!no_self) {
      stat->add (typeid (*this), (void *) this, sizeof (*this), sizeof (*this), parent, purpose, cat);
    }
    for (int i = 0; i < 4; ++i) {
      if (child (i)) {
        child (i)->mem_stat (stat, purpose, cat, no_self, parent);
      }
    }
  }

  const point_type &center () const
  {
    return m_center;
  }

  box_type quad_box (int quad) const
  {
    box_type qb = box_type::world ();
    if (parent ()) {
      qb = box_type (m_corner, parent ()->center ());
    }

    switch (quad) {
    case 0: return box_type (m_center, qb.upper_right ());
    case 1: return box_type (m_center, qb.upper_left ());
    case 2: return box_type (m_center, qb.lower_left ());
    case 3: return box_type (m_center, qb.lower_right ());
    default: return qb;
    }
  }

private:
  box_tree_node *mp_parent;
  size_t m_lenq, m_len;
  size_t m_childrefs [4];
  point_type m_center, m_corner;

  box_tree_node (const box_tree_node &d);
  box_tree_node &operator= (const box_tree_node &d);

  box_tree_node (box_tree_node *parent, const point_type &center, const point_type &corner, unsigned int quad)
  {
    init (parent, center, corner, quad);
  }

  void init (box_tree_node *parent, const point_type &center, const point_type &corner, unsigned int quad)
  {
    // 
    m_center = center;
    m_corner = corner;

    m_lenq = m_len = 0;
    for (int i = 0; i < 4; ++i) {
      m_childrefs [i] = 0;
    }

    mp_parent = (box_tree_node *)((char *) parent + quad);
    if (parent) {
      // 请相信现代编译器, 不要关公面前耍大刀, 但凡觉得优化不好有没有可能是编译器版本太低了?
      m_len = (parent->m_childrefs [quad] >> 1);
      parent->m_childrefs [quad] = size_t (this);
    }
  }
};
```

## 题外

这应该是最后一篇有关 Klayout 的文章, 因此做个题外总结. 很显然, Klayout 的整体风格较为传统, 并没有使用到许多现代 C++ 的编程思想以及技巧, 但这并不意味着 Klayout 的代码写得烂. 相反, Klayout 代码整体上写得非常好, 大量的通用框架/流程使得整体代码量大大减少. 几乎一个人维护如此庞大的系统势必使得代码复用的优先级大大提高, 可以说在设计上 Klayout 是非常成功的.

当然 Klayout 依旧存在许多问题, 例如极度通用化的流程使得性能下降以及部分边界情况错误等等问题, 例如作为一个接近二十年的项目使得底层存在重复造轮子的情况, 例如被 EDA 领域裹挟存在大量的历史包袱, 例如显而易见的 C 风格代码(如没有意义的代码端的 trick, 或直接针对指针地址进行操作等等(求求大神们了, 别直接针对指针地址硬编码了, 看都看不懂)).
