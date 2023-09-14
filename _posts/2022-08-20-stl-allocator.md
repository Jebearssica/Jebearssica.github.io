---
layout: post
title: 'STL: Allocator'
date: 2022-08-20 09:41 +0800
tags: [stl, c++]
categories: [c++, stl]
author: Jebearssica
---

分配器, 通常与容器共同使用而不单独使用, 作用是给容器指定一种分配空间的形式.

> 你硬要用的话毕竟它是个类, 可以直接调用. 但个人认为你不该这样用, 这就好像装机时候的防呆口一样, 防呆不防傻, 人家设计的时候就防止你单独使用从而阻碍你使用了, 你还硬要用.
{: .prompt-tip }

```c++
int *p;
allocator<int> alloc1;
int size;
cin >> size;
p = alloc1.allocate(size);
// 和内存分配不同, 如果你要用分配器, 你必须手动输入要释放的元素个数
alloc1.deallocate(size);
// 其他一些分配器
```

## `allocate` 与 `deallocate`

源码很长, 因此我们逐步分析. 对于分配器来说, 最重要的两个成员函数就是上述的用于分配内存和回收内存.

```c++
// allocate
_NODISCARD _DECLSPEC_ALLOCATOR _Ty *allocate(_CRT_GUARDOVERFLOW const size_t _Count)
{ // allocate array of _Count elements
    // 看到没, 这里一个静态的指针类型转换, 将万能指针转换为容器中元素相同类型的指针, 因此_Allocate() 就是开辟内存的步骤
    return (static_cast<_Ty *>(_Allocate<_New_alignof<_Ty>>(_Get_size_of_n<sizeof(_Ty)>(_Count))));
}

_NODISCARD _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS _DECLSPEC_ALLOCATOR _Ty *allocate(
    _CRT_GUARDOVERFLOW const size_t _Count, const void *)
{ // allocate array of _Count elements, ignore hint
    // 这个就不管, 重载多了一个额外的指针参数, 用于了解你最终需要分配的是 _Count 个什么类型的东西, 最终调用的都是上面那个重载
    return (allocate(_Count));
}
```

前面很长一串的返回类型可以先不管, 总之能看出返回的应该是一个指针, 这与我们的期望也是相同的. 因为分配内存的具体过程是, 开辟内存->指针转换, 这里返回的指针势必能被容器接受从而接受该指针对应的内存块.

深入下去看一下, 一个仿函数调用了全局 `new`, 完事儿, 就是和理论上一样的.

> 你要用 vscode 代码跳转功能去看的话, 它会跳转至和函数形参匹配的对应函数, 你需要找到最初的仿函数或函数模板才能知道它真正在干什么
{: .prompt-tip }

```c++
struct _Default_allocate_traits
{
    _DECLSPEC_ALLOCATOR static void *_Allocate(const size_t _Bytes)
    {
        return (::operator new(_Bytes));
    }

#if _HAS_ALIGNED_NEW
    _DECLSPEC_ALLOCATOR static void *_Allocate_aligned(const size_t _Bytes, const size_t _Align)
    {
        return (::operator new (_Bytes, align_val_t{_Align}));
    }
#endif /* _HAS_ALIGNED_NEW */
};
```

对于 `deallocate`, 我们可以同样分析, 实际调用全局 `delete`

```c++
void deallocate(_Ty *const _Ptr, const size_t _Count)
{ // deallocate object at _Ptr
    // no overflow check on the following multiply; we assume _Allocate did that check
    _Deallocate<_New_alignof<_Ty>>(_Ptr, sizeof(_Ty) * _Count);
}

template <size_t _Align,
          enable_if_t<(!_HAS_ALIGNED_NEW || _Align <= __STDCPP_DEFAULT_NEW_ALIGNMENT__), int> = 0>
inline void _Deallocate(void *_Ptr, size_t _Bytes)
{ // deallocate storage allocated by _Allocate when !_HAS_ALIGNED_NEW || _Align <= __STDCPP_DEFAULT_NEW_ALIGNMENT__
#if defined(_M_IX86) || defined(_M_X64)
    if (_Bytes >= _Big_allocation_threshold)
    { // boost the alignment of big allocations to help autovectorization
        _Adjust_manually_vector_aligned(_Ptr, _Bytes);
    }
#endif /* defined(_M_IX86) || defined(_M_X64) */

    ::operator delete(_Ptr, _Bytes);
}
```

## 源码

MSVC 14.16.27023 中 `<xmemory>`, 实际上能够看成 `new` 与 `delete` 的一层封装. 在实际应用中, 还存在更多中的分配器, 以实现诸如内存池等其他的分配模式.

```c++
template <class _Ty>
class allocator
{ // generic allocator for objects of class _Ty
public:
    static_assert(!is_const_v<_Ty>,
                  "The C++ Standard forbids containers of const elements "
                  "because allocator<const T> is ill-formed.");

    using _Not_user_specialized = void;

    using value_type = _Ty;

    _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS typedef _Ty *pointer;
    _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS typedef const _Ty *const_pointer;

    _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS typedef _Ty &reference;
    _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS typedef const _Ty &const_reference;

    _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS typedef size_t size_type;
    _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS typedef ptrdiff_t difference_type;

    using propagate_on_container_move_assignment = true_type;
    using is_always_equal = true_type;

    template <class _Other>
    struct _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS rebind
    { // convert this type to allocator<_Other>
        using other = allocator<_Other>;
    };

    _NODISCARD _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS _Ty *address(_Ty &_Val) const noexcept
    { // return address of mutable _Val
        return (_STD addressof(_Val));
    }

    _NODISCARD _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS const _Ty *address(const _Ty &_Val) const noexcept
    { // return address of nonmutable _Val
        return (_STD addressof(_Val));
    }

    constexpr allocator() noexcept
    { // construct default allocator (do nothing)
    }

    constexpr allocator(const allocator &) noexcept = default;
    template <class _Other>
    constexpr allocator(const allocator<_Other> &) noexcept
    { // construct from a related allocator (do nothing)
    }

    void deallocate(_Ty *const _Ptr, const size_t _Count)
    { // deallocate object at _Ptr
        // no overflow check on the following multiply; we assume _Allocate did that check
        _Deallocate<_New_alignof<_Ty>>(_Ptr, sizeof(_Ty) * _Count);
    }

    _NODISCARD _DECLSPEC_ALLOCATOR _Ty *allocate(_CRT_GUARDOVERFLOW const size_t _Count)
    { // allocate array of _Count elements
        return (static_cast<_Ty *>(_Allocate<_New_alignof<_Ty>>(_Get_size_of_n<sizeof(_Ty)>(_Count))));
    }

    _NODISCARD _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS _DECLSPEC_ALLOCATOR _Ty *allocate(
        _CRT_GUARDOVERFLOW const size_t _Count, const void *)
    { // allocate array of _Count elements, ignore hint
        return (allocate(_Count));
    }

    template <class _Objty,
              class... _Types>
    _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS void construct(_Objty *const _Ptr, _Types &&..._Args)
    { // construct _Objty(_Types...) at _Ptr
        ::new (const_cast<void *>(static_cast<const volatile void *>(_Ptr)))
            _Objty(_STD forward<_Types>(_Args)...);
    }

    template <class _Uty>
    _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS void destroy(_Uty *const _Ptr)
    { // destroy object at _Ptr
        _Ptr->~_Uty();
    }

    _NODISCARD _CXX17_DEPRECATE_OLD_ALLOCATOR_MEMBERS size_t max_size() const noexcept
    { // estimate maximum array size
        return (static_cast<size_t>(-1) / sizeof(_Ty));
    }
};
```
