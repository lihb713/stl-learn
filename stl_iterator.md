# stl_iterator.h

>文件路径：bits/stl_iterator.h


# 概述

这里只对文件中定义的__gnu_cxx::__normal_iterator进行解释。


# __gnu_cxx::__normal_iterator


__normal_iterator是一个适配器，主要将非迭代器类型转为迭代器（如指针）。


## 定义

```c++
// 这个迭代器适配器是正常的，因为它不会更改迭代器参数的任何运算符的语义。
// 它的主要目的是将一个不是类的迭代器（例如指针）转换为一个类迭代器。
// _Container参数仅存在，因此使用此模板的不同容器可以实例化不同的类型，即使_Iterator参数相同。
using std::iterator_traits;
using std::iterator;
template<typename _Iterator, typename _Container>
class __normal_iterator
{
    //...
}
```


## 数据成员

```c++
protected:
    _Iterator _M_current;

    typedef iterator_traits<_Iterator>		__traits_type;

public:
    // 下面成员类型是std::iterator标准接口
    typedef _Iterator					iterator_type;
    typedef typename __traits_type::iterator_category iterator_category;
    typedef typename __traits_type::value_type  	value_type;
    typedef typename __traits_type::difference_type 	difference_type;
    typedef typename __traits_type::reference 	reference;
    typedef typename __traits_type::pointer   	pointer;
```

这里共有接口部分就是std::iterator定义的标准接口。另外还定义了一个_Iterator类型成员_M_current，显然__normal_iterator作为一个适配器，所有操作将依赖于_Iterator对象。


## 构造函数

```c++
public:
    _GLIBCXX_CONSTEXPR __normal_iterator() _GLIBCXX_NOEXCEPT  //普通构造
        : _M_current(_Iterator()) { }

    explicit
        __normal_iterator(const _Iterator& __i) _GLIBCXX_NOEXCEPT  //拷贝构造
        : _M_current(__i) { }

    // 允许iterator到const_iterator转换
    template<typename _Iter>
    __normal_iterator(const __normal_iterator<_Iter,
        typename __enable_if<
            (std::__are_same<_Iter, typename _Container::pointer>::__value),
            _Container
         >::__type>
        & __i) _GLIBCXX_NOEXCEPT
        : _M_current(__i.base()) { }
```

OK，__normal_iterator的第二个拷贝构造函数很繁琐。我们逐一拆解。首先是are_same，是一个类模板，只有当其两个模板参数相同时其__value才是true_type。下面是__are_same的定义，位于bits/cpp_type_traits.h

```c++
template<typename, typename>
struct __are_same
{
    enum { __value = 0 };
    typedef __false_type __type;
};

template<typename _Tp>
struct __are_same<_Tp, _Tp>
{
    enum { __value = 1 };
    typedef __true_type __type;
};
```

因此__normal_iterator构造函数中只有当_Iter和_Container::pointer类型相同时__are_same::__value才是true_type。也就是__enable_if的第一个模板参数为true。

__enable_if用于编译时判断一个类型是否有效。它有两个模板参数，第一个为bool类型，第二个为任何类型_Tp。如果bool类型传入true，__enable_if的__type才会返回_Tp。否则会编译错误：

```c++
template<bool, typename>
struct __enable_if 
{ };

template<typename _Tp>
struct __enable_if<true, _Tp>
{ typedef _Tp __type; };
```

所以总和上面的分析我们可以明白__normal_iterator第二个拷贝构造函数传入的类型可以是其他迭代器类型_Iter，但是这个_Iter必须和当前迭代器的_Container::pointer类型相同，否则编译错误。换句话说我们可以从与_Iterator不同的迭代器类型_Iter拷贝迭代器，但是_Iter必须是_Container::pointer类型。

>批注：暂时没有见到_Iterator不是_Container::pointer类型的情况。

## 转为const对象

```c++
#if __cplusplus >= 201103L
    __normal_iterator<typename _Container::pointer, _Container>
        _M_const_cast() const noexcept
    {
        using _PTraits = std::pointer_traits<typename _Container::pointer>;
        return __normal_iterator<typename _Container::pointer, _Container>
            (_PTraits::pointer_to(const_cast<typename _PTraits::element_type&>
                (*_M_current)));
    }
#else
    __normal_iterator
        _M_const_cast() const
    {
        return *this;
    }
#endif
```

## 迭代器移动操作

```c++
public:
    // Forward iterator requirements
    reference
        operator*() const _GLIBCXX_NOEXCEPT
    {
        return *_M_current;
    }

    pointer
        operator->() const _GLIBCXX_NOEXCEPT
    {
        return _M_current;
    }

    __normal_iterator&
        operator++() _GLIBCXX_NOEXCEPT
    {
        ++_M_current;
        return *this;
    }

    __normal_iterator
        operator++(int) _GLIBCXX_NOEXCEPT
    {
        return __normal_iterator(_M_current++);
    }

    // Bidirectional iterator requirements
    __normal_iterator&
        operator--() _GLIBCXX_NOEXCEPT
    {
        --_M_current;
        return *this;
    }

    __normal_iterator
        operator--(int) _GLIBCXX_NOEXCEPT
    {
        return __normal_iterator(_M_current--);
    }

    // Random access iterator requirements
    reference
        operator[](difference_type __n) const _GLIBCXX_NOEXCEPT
    {
        return _M_current[__n];
    }

    __normal_iterator&
        operator+=(difference_type __n) _GLIBCXX_NOEXCEPT
    {
        _M_current += __n; return *this;
    }

    __normal_iterator
        operator+(difference_type __n) const _GLIBCXX_NOEXCEPT
    {
        return __normal_iterator(_M_current + __n);
    }

    __normal_iterator&
        operator-=(difference_type __n) _GLIBCXX_NOEXCEPT
    {
        _M_current -= __n; return *this;
    }

    __normal_iterator
        operator-(difference_type __n) const _GLIBCXX_NOEXCEPT
    {
        return __normal_iterator(_M_current - __n);
    }
```

前两个是成员访问操作符重载，注意这两个方法被定义为const成员函数。const成员函数的含义是迭代器本身是const的，无法前后移动或随机访问。但是迭代器内部指向的数据却可以任何访问和更改。因此理应定义为const。

这一组是迭代器移动函数，不同迭代器类型支持不同范围的移动操作，因此需要进行不同操作符重载。如forward iterator需要递增操作，bidirectional iterator还需要递减操作，random access iterator需要支持向前或向后移动n个元素。另外所有迭代器都需要重载解引用操作符（\*）和指针操作符（->）。


## base()

```c++
public:
    const _Iterator&
        base() const _GLIBCXX_NOEXCEPT
    {
        return _M_current;
    }
```

base()返回当前迭代器的常量引用。


## 比较运算符（非成员方法）

```c++
// 在下面的内容中，允许左侧和右侧迭代器的类型不同（概念上的const volatile限定），
// 因此cv限定迭代器和非cv限定的迭代器之间的比较是有效的。
// 然而，如果我们不提供操作数相同类型的重载，
// std::rel_ops中贪婪且不友好的运算符将使重载解析不明确（当在作用域中时）。
// 有人能提醒我通用编程是关于什么的吗？——Gaby

// Forward iterator requirements
template<typename _IteratorL, typename _IteratorR, typename _Container>
inline bool
    operator==(const __normal_iterator<_IteratorL, _Container>&__lhs,
        const __normal_iterator<_IteratorR, _Container>&__rhs)
    _GLIBCXX_NOEXCEPT
{
    return __lhs.base() == __rhs.base();
}

template<typename _Iterator, typename _Container>
inline bool
    operator==(const __normal_iterator<_Iterator, _Container>&__lhs,
        const __normal_iterator<_Iterator, _Container>&__rhs)
    _GLIBCXX_NOEXCEPT
{
    return __lhs.base() == __rhs.base();
}

template<typename _IteratorL, typename _IteratorR, typename _Container>
inline bool
    operator!=(const __normal_iterator<_IteratorL, _Container>&__lhs,
        const __normal_iterator<_IteratorR, _Container>&__rhs)
    _GLIBCXX_NOEXCEPT
{
    return __lhs.base() != __rhs.base();
}

template<typename _Iterator, typename _Container>
inline bool
    operator!=(const __normal_iterator<_Iterator, _Container>&__lhs,
        const __normal_iterator<_Iterator, _Container>&__rhs)
    _GLIBCXX_NOEXCEPT
{
    return __lhs.base() != __rhs.base();
}

// Random access iterator requirements
template<typename _IteratorL, typename _IteratorR, typename _Container>
inline bool
    operator<(const __normal_iterator<_IteratorL, _Container>&__lhs,
        const __normal_iterator<_IteratorR, _Container>&__rhs)
    _GLIBCXX_NOEXCEPT
{
    return __lhs.base() < __rhs.base();
}

template<typename _Iterator, typename _Container>
inline bool
    operator<(const __normal_iterator<_Iterator, _Container>&__lhs,
        const __normal_iterator<_Iterator, _Container>&__rhs)
    _GLIBCXX_NOEXCEPT
{
    return __lhs.base() < __rhs.base();
}

template<typename _IteratorL, typename _IteratorR, typename _Container>
inline bool
    operator>(const __normal_iterator<_IteratorL, _Container>&__lhs,
        const __normal_iterator<_IteratorR, _Container>&__rhs)
    _GLIBCXX_NOEXCEPT
{
    return __lhs.base() > __rhs.base();
}

template<typename _Iterator, typename _Container>
inline bool
    operator>(const __normal_iterator<_Iterator, _Container>&__lhs,
        const __normal_iterator<_Iterator, _Container>&__rhs)
    _GLIBCXX_NOEXCEPT
{
    return __lhs.base() > __rhs.base();
}

template<typename _IteratorL, typename _IteratorR, typename _Container>
inline bool
    operator<=(const __normal_iterator<_IteratorL, _Container>&__lhs,
        const __normal_iterator<_IteratorR, _Container>&__rhs)
    _GLIBCXX_NOEXCEPT
{
    return __lhs.base() <= __rhs.base();
}

template<typename _Iterator, typename _Container>
inline bool
    operator<=(const __normal_iterator<_Iterator, _Container>&__lhs,
        const __normal_iterator<_Iterator, _Container>&__rhs)
    _GLIBCXX_NOEXCEPT
{
    return __lhs.base() <= __rhs.base();
}

template<typename _IteratorL, typename _IteratorR, typename _Container>
inline bool
    operator>=(const __normal_iterator<_IteratorL, _Container>&__lhs,
        const __normal_iterator<_IteratorR, _Container>&__rhs)
    _GLIBCXX_NOEXCEPT
{
    return __lhs.base() >= __rhs.base();
}

template<typename _Iterator, typename _Container>
inline bool
    operator>=(const __normal_iterator<_Iterator, _Container>&__lhs,
        const __normal_iterator<_Iterator, _Container>&__rhs)
    _GLIBCXX_NOEXCEPT
{
    return __lhs.base() >= __rhs.base();
}
```

这组函数最重要的是开始写的注释，有两点需要说明。首先这组函数允许不同类型迭代器进行比较，即使cv限定不同，所谓cv就是const和volatile修饰符。

但同时对于每个操作符又重载了迭代器类型相同的版本，这是为了和std::rel_ops中的关系运算符不冲突，因为std::rel_ops中关系运算符的运算符对象为相同类型。std::rel_pos是一个命名空间，其中定义了所有关系运算符，这些运算符只用<和==运算符实现，这意味着只需要实现两个运算符，通过std::rel_ops就能获得所有6种运算符。


## 迭代器之间算数运算符

```c++
// _GLIBCXX_RESOLVE_LIB_DEFECTS
// 根据DR179决议，不仅各种比较运算符，而且运算符-必须接受混合iterator/const_iterator参数
template<typename _IteratorL, typename _IteratorR, typename _Container>
#if __cplusplus >= 201103L
// DR 685.
inline auto
    operator-(const __normal_iterator<_IteratorL, _Container>&__lhs,
        const __normal_iterator<_IteratorR, _Container>&__rhs) noexcept
    -> decltype(__lhs.base() - __rhs.base())  //尾置返回类型
#else
inline typename __normal_iterator<_IteratorL, _Container>::difference_type
    operator-(const __normal_iterator<_IteratorL, _Container>&__lhs,
        const __normal_iterator<_IteratorR, _Container>&__rhs)
#endif
{
    return __lhs.base() - __rhs.base();
}

template<typename _Iterator, typename _Container>
inline typename __normal_iterator<_Iterator, _Container>::difference_type
    operator-(const __normal_iterator<_Iterator, _Container>&__lhs,
        const __normal_iterator<_Iterator, _Container>&__rhs)
    _GLIBCXX_NOEXCEPT
{
    return __lhs.base() - __rhs.base();
}

template<typename _Iterator, typename _Container>
inline __normal_iterator<_Iterator, _Container>
    operator+(typename __normal_iterator<_Iterator, _Container>::difference_type
        __n, const __normal_iterator<_Iterator, _Container>&__i)
    _GLIBCXX_NOEXCEPT
{
    return __normal_iterator<_Iterator, _Container>(__i.base() + __n);
}
```


