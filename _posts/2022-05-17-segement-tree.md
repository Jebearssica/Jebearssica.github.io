---
layout: post
title: Segement Tree
date: 2022-05-17 15:37 +0800
tags: [data structure, segment tree, old driver tree]
categories: [Data Structure, Segment Tree]
author: Jebearssica
math: true
---

你要不嫌麻烦, 不怕代码量大, 面对几乎一切有关区间信息维护的问题, 你都可以用线段树解决. 毕竟, 它可以实现 $O(\log{N})$ 的区间修改, 区间查询.

> 上面说的区间, 当然也包括单点这个特例. 上面说的查询, 不仅包括区间和, 区间最值也行. 换言之, 只要你维护该区间的信息能够通过左右孩子结点表达, 那么这个信息都能够通过线段树维护.
{: .prompt-tip }

## 大致思想

线段树通常采用堆来存储, 我们通过一个完全二叉堆来构造, 即对于下标 index, 如果它有左右孩子结点
, 那么可以通过 2 \* index 与 2 \* index + 1 来访问其左右孩子结点

![segTree](https://oi-wiki.org/ds/images/segt1.svg)

每个结点都储存着父结点的一半元素和的信息, 即, 每个结点都视作一个区间, 以下通过区间和来举线段树的例子, 当然也可以通过下面的代码写出一个更通用的模板

```c++
// 以建树部分举例
void buildTree(int left, int right, int rootIdx)
{
    if (left == right)
    {
        nums[rootIdx] = oriNums[left];
        return;
    }
    int mid = left + ((right - left) >> 1);
    buildTree(left, mid, rootIdx * 2), buildTree(mid + 1, right, rootIdx * 2 + 1);
    // 用一个通用 operator 函数来表示, 如何通过左右结点构成根结点
    nums[rootIdx] = op(nums[rootIdx * 2], nums[rootIdx * 2 + 1]);
}
```

### 建树

和二分搜索差不多, 找到中点后分成左右区间, 然后后序递归建树, 最后根据左右子树的结果构建当前结点. 应当注意, 线段树里的索引从 1 开始(你 0 就不能通过前面说的方法访问左右孩子结点了)

```c++
void buildTree(int left, int right, int idx)
{
    if (left == right)
    {
        nums[idx] = oriNums[left - 1];
        return;
    }
    int mid = left + ((right - left) >> 1);
    buildTree(left, mid, idx * 2), buildTree(mid + 1, right, idx * 2 + 1);
    // 通过孩子结点更新父结点
    pushup(idx);
}
```

### 区间增加

所有涉及到区间的修改, 我们都需要引入懒惰标记. 如果每次修改我们都及时修改整棵树, 那么时间复杂度与普通数组无异. 我们只有在访问(修改或查询)至父亲结点时, 才将变动更新至子结点, 即懒惰标记下沉. 所以本次修改也算一次访问, 对应区间刚生成的懒惰标记也要下沉.

```c++
/*
    [left, right]: 搜索区间
    [curL, curR]: nums[idx] 所存储的范围
    idx: 当前结点索引
    val: 更改的值
*/
void add(int left, int right, int curL, int curR, int idx, int val)
{
    if (left <= curL && curR <= right)
    {
        nums[idx] += val * (curR - curL + 1);
        addTags[idx] += val;
        return;
    }
    int curM = curL + ((curR - curL) >> 1);
    // 有标记且非叶子结点, 标记下沉至子结点
    if (addTags[idx])
        pushdown(addTags, curL, curR, idx);
    if (left <= curM)
        add(left, right, curL, curM, idx * 2, val);
    if (curM < right)
        add(left, right, curM + 1, curR, idx * 2 + 1, val);
    pushup(idx);
}
```

### 区间替换

与区间增加类似, 不过应当注意的是, 此时我们不能通过标记是否为 0 来确定该区间是否应该被修改, 因为我们可能对区间赋 0, 因此我们需要额外的一个数组存储是否修改.

```c++
/*
    [left, right]: 搜索区间
    [curL, curR]: nums[idx] 所存储的范围
    idx: 当前结点索引
    val: 更改的值
*/
void unitVal(int left, int right, int curL, int curR, int idx, int val)
{
    if (left <= curL && curR <= right)
    {
        nums[idx] = val * (curR - curL + 1);
        replaceTags[idx] = val;
        return;
    }
    int curM = curL + ((curR - curL) >> 1);
    // 有标记且非叶子结点, 标记下沉至子结点
    if (isValidReplace[idx])
        pushdown(replaceTags, curL, curR, idx);
    if (left <= curM)
        unitVal(left, right, curL, curM, idx * 2, val);
    if (curM < right)
        unitVal(left, right, curM + 1, curR, idx * 2 + 1, val);
    pushup(idx);
}
```

### 区间查询

我们在查询前, 应当检查该结点是否修改, 在修改后并将懒惰标记下沉

```c++
/*
    [left, right]: 搜索区间
    [curL, curR]: nums[idx] 所存储的范围
    idx: 当前结点索引
*/
int intervalSum(int left, int right, int curL, int curR, int idx)
{
    if (left <= curL && curR <= right)
        return nums[idx];
    int curM = curL + ((curR - curL) >> 1), sum = 0;
    // 增加和替换通常只进行一个操作, 因为线段树通常不同时涉及两个操作, 有合法修改就下沉懒惰标记
    if (addTags[idx])
        pushdown(addTags, curL, curR, idx);
    else if (isValidReplace[idx])
        pushdown(replaceTags, curL, curR, idx);
    if (left <= curM)
        sum += intervalSum(left, right, curL, curM, idx * 2);
    if (curM + 1 <= right)
        sum += intervalSum(left, right, curM + 1, curR, idx * 2 + 1);
    return sum;
}
```

## 小结

属于是思想很简单, 实现挺困难的数据结构. 个人感觉思想难度低于树状数组. 可以通过[Leetcode 307. 区域和检索 - 数组可修改](https://leetcode.cn/problems/range-sum-query-mutable/)来验证以下代码(不过他不涉及区间修改, 都是单点的)

```c++
#include <vector>
#include <iostream>
using namespace std;

class segTree
{
private:
    vector<int> nums, oriNums, addTags, replaceTags;
    vector<bool> isValidReplace;
    int start, end;
    void pushup(int idx)
    {
        nums[idx] = nums[idx << 1] + nums[idx << 1 | 1];
    }
    void pushdown(int curL, int curR, int idx)
    {
        // 叶子结点不下沉
        if (curL == curR)
            return;
        int curM = curL + ((curR - curL) >> 1);
        // 增标记, 下沉
        if (addTags[idx] != 0)
        {
            nums[idx * 2] += addTags[idx] * (curM - curL + 1), nums[idx * 2 + 1] += addTags[idx] * (curR - curM);
            addTags[idx * 2] += addTags[idx], addTags[idx * 2 + 1] += addTags[idx];
            addTags[idx] = 0;
        }
        // 替标记, 下沉
        else if (isValidReplace[idx])
        {
            nums[idx * 2] = replaceTags[idx] * (curM - curL + 1), nums[idx * 2 + 1] = replaceTags[idx] * (curR - curM);
            replaceTags[idx * 2] = replaceTags[idx], replaceTags[idx * 2 + 1] = replaceTags[idx];
            isValidReplace[idx * 2] = isValidReplace[idx * 2 + 1] = true;
            isValidReplace[idx] = false;
        }
    }
    void buildTree(int left, int right, int idx)
    {
        if (left == right)
        {
            nums[idx] = oriNums[left - 1];
            return;
        }
        int mid = left + ((right - left) >> 1);
        buildTree(left, mid, idx * 2), buildTree(mid + 1, right, idx * 2 + 1);
        pushup(idx);
    }
    int intervalSum(int left, int right, int curL, int curR, int idx)
    {
        if (left <= curL && curR <= right)
            return nums[idx];
        int curM = curL + ((curR - curL) >> 1), sum = 0;
        // 增加和替换通常只进行一个操作, 因为线段树通常不同时涉及两个操作
        pushdown(curL, curR, idx);
        if (left <= curM)
            sum += intervalSum(left, right, curL, curM, idx * 2);
        if (curM + 1 <= right)
            sum += intervalSum(left, right, curM + 1, curR, idx * 2 + 1);
        return sum;
    }
    void add(int left, int right, int curL, int curR, int idx, int val)
    {
        if (left <= curL && curR <= right)
        {
            nums[idx] += val * (curR - curL + 1);
            addTags[idx] += val;
            return;
        }
        int curM = curL + ((curR - curL) >> 1);
        // 有标记且非叶子结点, 标记下沉至子结点
        pushdown(curL, curR, idx);
        if (left <= curM)
            add(left, right, curL, curM, idx * 2, val);
        if (curM < right)
            add(left, right, curM + 1, curR, idx * 2 + 1, val);
        pushup(idx);
    }
    void unitVal(int left, int right, int curL, int curR, int idx, int val)
    {
        if (left <= curL && curR <= right)
        {
            nums[idx] = val * (curR - curL + 1);
            replaceTags[idx] = val;
            return;
        }
        int curM = curL + ((curR - curL) >> 1);
        // 有标记且非叶子结点, 标记下沉至子结点
        pushdown(curL, curR, idx);
        if (left <= curM)
            unitVal(left, right, curL, curM, idx * 2, val);
        if (curM < right)
            unitVal(left, right, curM + 1, curR, idx * 2 + 1, val);
        pushup(idx);
    }

public:
    segTree(vector<int> &initNums) : oriNums(initNums), start(1), end(initNums.size())
    {
        this->nums.resize(end * 4, 0);
        this->addTags.resize(end * 4, 0);
        this->replaceTags.resize(end * 4, 0);
        this->isValidReplace.resize(end * 4, false);
        buildTree(start, end, 1);
    }
    ~segTree() {}
    int intervalSum(int left, int right)
    {
        return this->intervalSum(left, right, start, end, 1);
    }
    void add(int left, int right, int val)
    {
        this->add(left, right, start, end, 1, val);
    }
    void unitVal(int left, int right, int val)
    {
        this->unitVal(left, right, start, end, 1, val);
    }
};

int main()
{
    // leetcode 上找了一个案例
    vector<int> input = {0, 9, 5, 7, 3};
    segTree s(input);
    cout << s.intervalSum(5, 5) << endl;
    cout << s.intervalSum(3, 5) << endl;
    cout << s.intervalSum(4, 4) << endl;
    s.unitVal(5, 5, 5);
    s.unitVal(2, 2, 7);
    s.unitVal(1, 1, 8);
    cout << s.intervalSum(2, 3) << endl;
    s.unitVal(2, 2, 9);
    cout << s.intervalSum(5, 5) << endl;
    s.unitVal(4, 4, 5);
    return 0;
}
```

## 拓展1: 动态开点线段树

无语了, Leetcode 又又又又出线段树的题了, [Leetcode 周赛2276. 统计区间中的整数数目](https://leetcode.cn/problems/count-integers-in-intervals/) 和 [Leetcode 699. 掉落的方块](https://leetcode.cn/problems/falling-squares/). 学之.

众所周知, 上述提前开空间很可能导致超空间( 别算了, leetcode 1e8 的空间复杂度过不了的 ), 这时候需要引入动态开辟空间的思想, 不提前建树, 一边修改遍历一边建树, 即每次 `pushdown` 来动态开辟左右孩子结点. 由于无需提前建树, 也就没有了 `build` 函数. 应当注意的是, 因为没有提前开辟空间, 所以你不能直接用 `idx * 2` 与 `idx * 2 + 1` 来访问左右孩子结点了, 这个时候通过指针访问. 同样的, 你可以使用上述两道题来验证下述代码. 不过针对699, 你需要进行些许修改, 使得结点维护最值.

> 当然, 你可以选择不使用形参来传递当前结点所管辖的区间, 可以直接存在结点中, 不过这个会使得更耗空间, 可能存在某种比较极端的情况, 使得这种写法被卡掉
{: .prompt-tip }
> 还有种预先开辟大块结点池以防止后面分散开辟造成内存碎片影响内存分配的写法. 不过你需要预估结点数量, 这我不擅长, 本人向来喜欢大力出奇迹. —— [reference](https://leetcode.cn/circle/discuss/1cQk4P/)
{: .prompt-info }

```c++
class dynamicSegTree
{
private:
    struct node
    {
        node *left, *right;
        int val;
        int addTag, replaceTag;
        bool isValidReplace;
        node() : left(nullptr), right(nullptr), val(0), addTag(0), replaceTag(0), isValidReplace(false) {}
    };
    node *dst;
    int maxRange;
    void pushup(node *root)
    {
        root->val = root->left->val + root->right->val;
    }
    void pushdown(int curL, int curR, node *root)
    {
        // 叶子结点不下沉
        if (curL == curR)
            return;
        if (root->left == nullptr)
            root->left = new node();
        if (root->right == nullptr)
            root->right = new node();
        int mid = curL + ((curR - curL) >> 1);
        // 增标记, 下沉
        if (root->addTag != 0)
        {
            root->left->addTag += root->addTag;
            root->right->addTag += root->addTag;
            root->left->val += root->addTag * (mid - curL + 1);
            root->right->val += root->addTag * (curR - mid);
            root->addTag = 0;
        }
        // 替标记, 下沉
        else if (root->isValidReplace)
        {
            root->left->isValidReplace = root->right->isValidReplace = true;
            root->left->val = root->replaceTag * (mid - curL + 1);
            root->right->val = root->replaceTag * (curR - mid);
            root->left->replaceTag = root->right->replaceTag = root->replaceTag;
            root->isValidReplace = false;
        }
    }
    void add(int left, int right, int curL, int curR, int val, node *root)
    {
        if (left <= curL && curR <= right)
        {
            root->addTag += val;
            root->val += val * (curR - curL + 1);
            return;
        }
        int curM = curL + ((curR - curL) >> 1);
        // 有标记且非叶子结点, 标记下沉至子结点
        pushdown(curL, curR, root);
        if (left <= curM)
            add(left, right, curL, curM, val, root->left);
        if (curM < right)
            add(left, right, curM + 1, curR, val, root->right);
        pushup(root);
    }
    void unitVal(int left, int right, int curL, int curR, int val, node *root)
    {
        if (left <= curL && curR <= right)
        {
            root->val = val * (curR - curL + 1);
            root->replaceTag = val;
            root->isValidReplace = true;
            return;
        }
        int curM = curL + ((curR - curL) >> 1);
        pushdown(curL, curR, root);
        if (left <= curM)
            unitVal(left, right, curL, curM, val, root->left);
        if (curM < right)
            unitVal(left, right, curM + 1, curR, val, root->right);
        pushup(root);
    }
    int intervalSum(int left, int right, int curL, int curR, node *root)
    {
        if (left <= curL && curR <= right)
            return root->val;
        int curM = curL + ((curR - curL) >> 1), sum = 0;
        pushdown(curL, curR, root);
        if (left <= curM)
            sum += intervalSum(left, right, curL, curM, root->left);
        if (curM + 1 <= right)
            sum += intervalSum(left, right, curM + 1, curR, root->right);
        return sum;
    }

public:
    dynamicSegTree(int N) : maxRange(N)
    {
        this->dst = new node();
    }
    ~dynamicSegTree() {}
    int intervalSum(int left, int right)
    {
        return this->intervalSum(left, right, 1, this->maxRange, this->dst);
    }
    void add(int left, int right, int val)
    {
        this->add(left, right, 1, this->maxRange, val, this->dst);
    }
    void unitVal(int left, int right, int val)
    {
        this->unitVal(left, right, 1, this->maxRange, val, this->dst);
    }
};

int main()
{
    dynamicSegTree DST(1e9);
    cout << DST.intervalSum(1, 1e9) << endl;
    DST.unitVal(8, 43, 1);
    cout << DST.intervalSum(1, 1e9) << endl;
    DST.unitVal(13, 16, 1);
    cout << DST.intervalSum(1, 1e9) << endl;
    DST.unitVal(26, 33, 1);
    DST.unitVal(28, 36, 1);
    DST.unitVal(29, 37, 1);
    cout << DST.intervalSum(1, 1e9) << endl;
    DST.unitVal(34, 46, 1);
    DST.unitVal(10, 23, 1);
    cout << DST.intervalSum(1, 1e9) << endl;
    return 0;
}
```

## 拓展2: 颜色均摊堆(珂朵莉树, 老司机树)

> 名字起源充满二次元气息, 我搞不懂

主要思路是, 通过 set (红黑树) 维护区间信息, 把具有相同信息的区间合并成一个结点存入. 在随机数据中, 随着操作数的增加, 合并的区间会越来越多, 最终剩余的区间数量比较有限, 从而能够实现较好的时间复杂度. 具体证明过程[在这](https://zhuanlan.zhihu.com/p/102786071), 我这种智商基本脱离这玩意儿了

### 分割区间(split)

两个核心操作之一, 用于将一个区间 $[L, R]$ 根据给定点 $idx$ 分割成两个区间 $[L, idx - 1], [idx, R]$, 并返回后者的迭代器. 从而能够将任意针对区间的操作, 转换为有序集合 set 上, 从 `split(L)` 至 `split(R + 1)` 的操作

```c++
// 返回类型 set<node>::iterator
auto split(int idx)
{
    auto iter = odt.lower_bound(node(idx, 0, 0));
    if (iter != odt.end() && iter->left == idx)
        return iter;
    // 从前一个区间进行切割, 因为要获得包含 L 的区间
    --iter;
    const int L = iter->left, R = iter->right, V = iter->value;
    odt.erase(iter);
    odt.insert(node(L, idx - 1, V));
    return odt.insert(node(idx, R, V)).first;
}
```

### 区间赋值(assign)及其他操作

进行区间赋值(统一值)时, 要保证有尽量多的区间被合并, 这才能使得区间数量快速下降, 从而保证时间复杂度. 即搜出迭代器区间, 删除, 然后插入一个合并后的区间

> 一定要先得到右迭代器, 再去搜左迭代器. 当搜索区间被一个结点覆盖时, `split(R + 1)` 会删除该结点再重新插入, 从而使得已经生成好的左迭代器失效. 反之, 搜左迭代器时, 右迭代器已经分割, 左迭代器的右边界一定不超过 `R`
{: .prompt-danger }

```c++
void assign(const int L, const int R, const int V)
{
    auto iterR = split(R + 1), iterL = split(L);
    odt.erase(iterL, iterR);
    odt.insert(node(L, R, V));
}
```

同理其他操作与 `assign` 类似, 一切针对区间的操作都可以用下面的代码实现

```c++
void perform(const int L, const int R)
{
    auto iterR = this->split(R + 1), iterL = this->split(L);
    for (; iterL != iterR; ++iterL)
    {
        // perform
    }
}
```

### 珂朵莉树小结

代码量比线段树少好多, 但是得保证是随机数据的情况下才能有较好的时间复杂度, 基于红黑树实现复杂度是 $O(\log{\log{N}})$, 基于链表的是 $O(\log{N})$. 然而, 在非随机数据的情况下, 整个方法复杂度趋向于暴力法 $O(N^2)$, 肯定没线段树适用范围广以及稳定的. 可以通过[Leetcode 732. 我的日程安排表 III](https://leetcode.cn/problems/my-calendar-iii/)来验证下面代码的正确性.

> 记得看清区间开闭情况, 珂朵莉树的基础结点是左右闭的
{: .prompt-tip }

```c++
class ODT
{
private:
    struct node
    {
        int left, right;
        mutable int val;
        node(const int L, const int R, const int V) : left(L), right(R), val(V) {}
        inline bool operator<(const node &t) const { return this->left < t.left; }
    };
    set<node> odt;
    int maxV;
    auto split(int idx)
    {
        auto iter = odt.lower_bound(node{idx, 0, 0});
        if (iter != odt.end() && iter->left == idx)
            return iter;
        --iter;
        const int L = iter->left, R = iter->right, V = iter->val;
        odt.erase(iter);
        odt.insert(node(L, idx - 1, V));
        return odt.insert(node(idx, R, V)).first;
    }
    int add(const int L, const int R, const int V)
    {
        auto iterR = split(R + 1), iterL = split(L);
        for (; iterL != iterR; ++iterL)
        {
            // cout << "Change: [" << iterL->left << ", " << iterL->right << "] " << iterL->val << endl;
            iterL->val += V;
            maxV = max(maxV, iterL->val);
        }
        return maxV;
    }

public:
    ODT() {}
    ~ODT() {}
    /* 省略一些需要针对特定题目修改的东西 */
};
```
