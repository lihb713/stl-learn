# stl_iterator_base_types.h

>文件路径：bits/stl_iterator_base_types.h


# 概述

该文件定义了五种基本迭代器类型
- std::input_iterator_tag
- std::output_iterator_tag
- std::forward_iterator_tag
- std::bidirectional_iterator_tag
- std::random_access_iterator_tag

定义了std::iterator。

定义了std::iterator_traits


# 文件头

```c++
/** @file bits/stl_iterator_base_types.h
 *  This is an internal header file, included by other library headers.
 *  Do not attempt to use it directly. @headername{iterator}
 *
 *  This file contains all of the general iterator-related utility types,
 *  such as iterator_traits and struct iterator.
 */

#ifndef _STL_ITERATOR_BASE_TYPES_H
#define _STL_ITERATOR_BASE_TYPES_H 1

#pragma GCC system_header

#include <bits/c++config.h>

#if __cplusplus >= 201103L
# include <type_traits>  // For _GLIBCXX_HAS_NESTED_TYPE, is_convertible
#endif
```


# 迭代器类型 iterator_tags

```c++
/**
    *  @defgroup iterator_tags Iterator Tags 
    *  他们是空类型，用于区分不同迭代器
    *  区别不在于它们包含什么，而是它们是什么
    *  基于不同迭代器类型所支持不同操作，可以使用不同底层算法
*/
//@{ 
///  Marking input iterators.
struct input_iterator_tag { };

///  Marking output iterators.
struct output_iterator_tag { };

/// Forward iterators support a superset of input iterator operations.
struct forward_iterator_tag : public input_iterator_tag { };

/// Bidirectional iterators support a superset of forward iterator
/// operations.
struct bidirectional_iterator_tag : public forward_iterator_tag { };

/// Random-access iterators support a superset of bidirectional
/// iterator operations.
struct random_access_iterator_tag : public bidirectional_iterator_tag { };
//@}
```


# std::iterator


```c++
/**
 *  @brief  普通的iterator类
 *
 *  这个类只定义嵌套的typedefs。迭代器可以从此类继承以节省一些工作
 *  这些typedefs用于专门化和重载
*/
template<typename _Category, typename _Tp, typename _Distance = ptrdiff_t,
    typename _Pointer = _Tp*, typename _Reference = _Tp&>
struct iterator
{
    /// iterator_tags标记类型之一
    typedef _Category  iterator_category;
    /// 迭代器“指向”的类型
    typedef _Tp        value_type;
    /// 迭代器之间的距离表示为这种类型
    typedef _Distance  difference_type;
    /// 表示一个pointer-to-value_type的类型
    typedef _Pointer   pointer;
    /// 表示一个reference-to-value_type的类型
    typedef _Reference reference;
};
```


# std::iterator_traits



```c++
/**
    *  @brief  迭代器的Traits类
    *
    *  这个类只定义嵌套的typedef。一般版本只是转发Iterator参数中的嵌套typedef。
    *  指针和指向常量的指针的特化版本提供了更严格、更正确的语义。
*/
#if __cplusplus >= 201103L

    _GLIBCXX_HAS_NESTED_TYPE(iterator_category)

    template<typename _Iterator,
        bool = __has_iterator_category<_Iterator>::value>
    struct __iterator_traits { };

    template<typename _Iterator>
    struct __iterator_traits<_Iterator, true>
    {
        typedef typename _Iterator::iterator_category iterator_category;
        typedef typename _Iterator::value_type        value_type;
        typedef typename _Iterator::difference_type   difference_type;
        typedef typename _Iterator::pointer           pointer;
        typedef typename _Iterator::reference         reference;
    };

    template<typename _Iterator>
    struct iterator_traits
        : public __iterator_traits<_Iterator> { };
#else
    template<typename _Iterator>
    struct iterator_traits
    {
        typedef typename _Iterator::iterator_category iterator_category;
        typedef typename _Iterator::value_type        value_type;
        typedef typename _Iterator::difference_type   difference_type;
        typedef typename _Iterator::pointer           pointer;
        typedef typename _Iterator::reference         reference;
    };
#endif
```

_GLIBCXX_HAS_NESTED_TYPE(_NTYPE)被定义在std/type_trits中，含义是：使用SFINAE确定类型_Tp是否具有可公开访问的成员类型_NTYPE。

>批注：SFINAE是C++模板方面进阶内容，暂时无法理解，阿巴阿巴

iterator_traits的作用就是从给定迭代器类型_Iterator中提取相应的成员类型。

iterator_traits在C++11版本前定义非常简单，只是对iterator中typedefs的封装。而C++11版本后变得更加复杂，这个变化点主要在于增加了对iterator_category是否存在的判断，因为我们只提供了存在iterator_category的iterator_traits版本，即__has_iterator_category<_Iterator>::value为true的版本。


## 内置指针的特化版本

```c++
/// 内置指针类型的特化版本
template<typename _Tp>
struct iterator_traits<_Tp*>
{
    typedef random_access_iterator_tag iterator_category;
    typedef _Tp                         value_type;
    typedef ptrdiff_t                   difference_type;
    typedef _Tp* pointer;
    typedef _Tp& reference;
};

/// 内置常量指针特化版本
template<typename _Tp>
struct iterator_traits<const _Tp*>
{
    typedef random_access_iterator_tag iterator_category;
    typedef _Tp                         value_type;
    typedef ptrdiff_t                   difference_type;
    typedef const _Tp* pointer;
    typedef const _Tp& reference;
};
```

>批注：注意常量指针特化版本总value_type是Tp，而不是const Tp。

# __iterator_category函数（语法糖）

```c++
/**
    *  这个函数不是C++标准，是仅供内部库使用的语法糖。
*/
template<typename _Iter>
inline typename iterator_traits<_Iter>::iterator_category
    __iterator_category(const _Iter&)
{
    return typename iterator_traits<_Iter>::iterator_category();
}
```

这个函数使用迭代器萃取器获得给定迭代器_Iter的迭代器类型iterator_category。



# 其他

```c++
// If _Iterator has a base returns it otherwise _Iterator is returned
// untouched
    template<typename _Iterator, bool _HasBase>
    struct _Iter_base
    {
        typedef _Iterator iterator_type;
        static iterator_type _S_base(_Iterator __it)
        {
            return __it;
        }
    };

    template<typename _Iterator>
    struct _Iter_base<_Iterator, true>
    {
        typedef typename _Iterator::iterator_type iterator_type;
        static iterator_type _S_base(_Iterator __it)
        {
            return __it.base();
        }
    };

#if __cplusplus >= 201103L
    template<typename _InIter>
    using _RequireInputIter = typename
        enable_if<is_convertible<typename
        iterator_traits<_InIter>::iterator_category,
        input_iterator_tag>::value>::type;
#endif
```


