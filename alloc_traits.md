# alloc_traits.h

>文件路径：ext/alloc_traits.h




# 概述

本文件定义了__alloc_traits模板类。__alloc_traits在许多容器中被用作提取给定内存分配器的属性。因此有必要进行说明。

__alloc_traits在C++11前后的版本变化较大，我们这里只关注C++11之后的版本。

# __alloc_traits


## 定义

```c++
template<typename _Alloc>
struct __alloc_traits
#if __cplusplus >= 201103L
    : std::allocator_traits<_Alloc>
#endif
{
    typedef _Alloc allocator_type;
#if __cplusplus >= 201103L
    //...
#else
    //...
#endif
};
```

可以看到__alloc_traits继承自std::allocator_traits。

>批注：这里暂时没有提供std::allocator_tratis的解释。之后有空补充


## Member type


```c++
public:
    typedef std::allocator_traits<_Alloc>           _Base_type;
    typedef typename _Base_type::value_type         value_type;
    typedef typename _Base_type::pointer            pointer;
    typedef typename _Base_type::const_pointer      const_pointer;
    typedef typename _Base_type::size_type          size_type;
    typedef typename _Base_type::difference_type    difference_type;
    // C++11 allocators do not define reference or const_reference
    typedef value_type& reference;
    typedef const value_type& const_reference;
    using _Base_type::allocate;
    using _Base_type::deallocate;
    using _Base_type::construct;
    using _Base_type::destroy;
    using _Base_type::max_size;
```



这里的member type定义基本和allocator共有接口中的一致。只是这里我们是通过allocator_traits萃取的。


## __is_customer_pointer

```c++
private:
    template<typename _Ptr>
    using __is_custom_pointer
        = std::__and_<std::is_same<pointer, _Ptr>,
        std::__not_<std::is_pointer<_Ptr>>>;
```


这里提供了一个私有的模板类型，实际上了解type_traits的话就理解他就是true_type或false_type。那么具体是true还是false由其中条件决定。如果_Ptr和pointer类型相同，且不是内置pointer类型，那么__is_customer_pointer为true。


## 非标准类型指针构造销毁

```c++
public:
    // 非标准指针类型的重载构造
    template<typename _Ptr, typename... _Args>
    static typename std::enable_if<__is_custom_pointer<_Ptr>::value>::type
        construct(_Alloc& __a, _Ptr __p, _Args&&... __args)
    {
        _Base_type::construct(__a, std::addressof(*__p),
            std::forward<_Args>(__args)...);
    }

    // 非标准指针类型的重载销毁
    template<typename _Ptr>
    static typename std::enable_if<__is_custom_pointer<_Ptr>::value>::type
        destroy(_Alloc& __a, _Ptr __p)
    {
        _Base_type::destroy(__a, std::addressof(*__p));
    }
```

这里使用到了type_traits中的enable_if来判断_Ptr是否是用户自定义指针，如果是的话enable_if的type才被定义，否则会出错。

construct和destroy的实现和new_allocator类似，不再赘述。


## 容器copy/move/swap相关属性


下面是容器相关的几个非常重要属性，决定了容器在copy/move/swap时的行为。
- propagate_on_container_copy_assignment
- propagate_on_container_move_assignment
- propagate_on_container_swap

这三个类型都为true_type或false_type。它们表示的含义是当propagate_on_container_XXX为true_type时我们在copy/move/swap容器时也会直接copy/move/swap本身。因此我们不应该将propagate_on_container_XXX理解为分配器的属性，propagate_on_container_XXX代表的真正含义是分配器是否属于容器“值”的一部分，这个概念总是与容器绑定的，只有分配器在容器的copy/move/swap时才需要考虑。propagate_on_container_XXX为true，那么代表分配器确实是容器“值”的一部分，所以此时当我们将一个容器copy/move/swap到另一个容器时必须将分配器也copy/move/swap过去。

这里表述可能过于抽象。但是我们想象一种场景，当我们初始化两个使用不同的allocator的容器A和B，在之后的代码中，我们将其中一个容器赋值给另一个，如A=B。那么此时A中的分配器应该使用B的吗？如果propagate_on_container_XXX为true，那么应该使用。

如果这些属性是false_type，那么我们就不会直接copy/move/swap分配器。这带来的后果是如果A=B，那么我们希望将B中元素拷贝到A中，但是两者可能拥有不同的分配器，此时我们不得不一个元素一个元素的copy/move。当然这里需要多一步判断，那就是两者分配器真的相同吗，如果相同的话我们起始可以避免一个元素一个元素拷贝。这个问题由__allocator_always_compares_equal来回答。

```c++
public:

    //如果可能则返回分配器__a的拷贝构造
    static _Alloc _S_select_on_copy(const _Alloc& __a)
    {
        return _Base_type::select_on_container_copy_construction(__a);
    }

    static void _S_on_swap(_Alloc& __a, _Alloc& __b)
    {
        std::__alloc_on_swap(__a, __b);
    }

    static constexpr bool _S_propagate_on_copy_assign()
    {
        return _Base_type::propagate_on_container_copy_assignment::value;
    }

    static constexpr bool _S_propagate_on_move_assign()
    {
        return _Base_type::propagate_on_container_move_assignment::value;
    }

    static constexpr bool _S_propagate_on_swap()
    {
        return _Base_type::propagate_on_container_swap::value;
    }

    static constexpr bool _S_always_equal()
    {
        return __allocator_always_compares_equal<_Alloc>::value;
    }

    static constexpr bool _S_nothrow_move()
    {
        return _S_propagate_on_move_assign() || _S_always_equal();
    }

    static constexpr bool _S_nothrow_swap()
    {
        using std::swap;
        return !_S_propagate_on_swap()
            || noexcept(swap(std::declval<_Alloc&>(), std::declval<_Alloc&>()));
    }

    template<typename _Tp>
    struct rebind
    {
        typedef typename _Base_type::template rebind_alloc<_Tp> other;
    };
```



























