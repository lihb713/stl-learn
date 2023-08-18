# type_traits

>文件路径：std/type_traits


# 概述

type_traits可以翻译为类型萃取器。其中定义了一些类模板，主要用于获取类型特征，比如is_void检查类型是否是void、is_fundamental检查类型是否是基本类型、is_const检查类型是否受const限定，等等。type_traits是C++提供的元编程工具，这些工具帮助我们可以将一些逻辑放在编译期间执行，如对于trivial类型和non-trivial类型编译不同的代码，而不是放在运行期间进行判断，这有助于提高代码执行效率。

type_traits中定义了非常多的类模板和函数模板，这里没有办法一一列举，我们只是摘录一些基本的定义进行解释。


# integral_constant

在type_traits中我们能够看到大量true_type和false_type。这两个类型就是表示真或假，如上面说到的is_void、is_fundamental等实际上都是true_type和false_type的子类。在GNU 2.9版本时，true_type和false_type定义非常简单，只是一个空类型而已：

```c++
struct true_type {};
struct false_type {};
```

但是到了GNU 4.9版本true_type和false_type更加灵活，可以与布尔类型相互转换：

```c++
/// integral_constant
template<typename _Tp, _Tp __v>
struct integral_constant
{
    static constexpr _Tp                  value = __v;
    typedef _Tp                           value_type;
    typedef integral_constant<_Tp, __v>   type;
    constexpr operator value_type() const { return value; }
#if __cplusplus > 201103L
    constexpr value_type operator()() const { return value; }
#endif
};

template<typename _Tp, _Tp __v>
constexpr _Tp integral_constant<_Tp, __v>::value;

/// 用作具有真值的编译时布尔值的类型。
typedef integral_constant<bool, true>     true_type;

/// 用作具有假值的编译时布尔值的类型。
typedef integral_constant<bool, false>    false_type;
```


从上面定义我们看到true_type和false_type也都是一种类型。但它们是由integral_constant类模板生成的。其中true_type可以看作是一种“bool值为true的类型”，false_type看作“bool值为false的类型”。

在integral_constant中我们需要注意的是其中定义了一个value成员，成员类型就为bool类型，值则根据true_type和false_type不同取true或false。另外其中重载了到bool类型的类型转换运算符，这样当我们使用true_type和false_type对象时，可以直接当作bool值使用：

```c++
std::cout << std::true_type() << std::endl;  //输出1
std::cout << std::false_type() << std::endl;  //输出0
```


# is_void

这里我们看一个实际的例子，如果使用true_type和false_type来定义这些类型萃取器的。我们选用is_void：

```c++
template<typename>
struct remove_cv;

template<typename>
struct __is_void_helper
    : public false_type { };

template<>
struct __is_void_helper<void>
    : public true_type { };

/// is_void
template<typename _Tp>
struct is_void
    : public __is_void_helper<typename remove_cv<_Tp>::type>::type
{ };
```

我们看到is_void类型是由__is_void_helper类型决定的。__is_void_helper的类型默认是false_type，同时还有一个特化版本，当类型为void是则为true_type。

另外我们在判断_Tp类型是否is_void时需要先通过typename remove_cv\<_Tp\>::type做类型转换，remove_cv用于去除_Tp类型中的const和volatile修饰符，其定义如下：


```c++
// Const-volatile modifications.

  /// remove_const
template<typename _Tp>
struct remove_const
{
    typedef _Tp     type;
};

template<typename _Tp>
struct remove_const<_Tp const>
{
    typedef _Tp     type;
};

/// remove_volatile
template<typename _Tp>
struct remove_volatile
{
    typedef _Tp     type;
};

template<typename _Tp>
struct remove_volatile<_Tp volatile>
{
    typedef _Tp     type;
};

/// remove_cv
template<typename _Tp>
struct remove_cv
{
    typedef typename
        remove_const<typename remove_volatile<_Tp>::type>::type     type;
};
```


# 条件运算


type_traits中还有一组类用途广泛，那就是其中的条件运算模板类：__and_，__or_，__not_。上面的is_void等模板只能做一些简单的逻辑性判断。但是有了这些条件运算模板我们可以组合任意数量的true_type和false_type，在模板元编程时提供更为复杂的控制。

## conditional

首先与或非运算都是基于conditional模板类：

```c++
/// 将typedef成员定义为两种类型之一
template<bool _Cond, typename _Iftrue, typename _Iffalse>
struct conditional
{
    typedef _Iftrue type;
};

// Partial specialization for false.
template<typename _Iftrue, typename _Iffalse>
struct conditional<false, _Iftrue, _Iffalse>
{
    typedef _Iffalse type;
};
```

## __or_

```c++
template<typename...>
struct __or_;

template<>
struct __or_<>
    : public false_type
{ };

template<typename _B1>
struct __or_<_B1>
    : public _B1
{ };

template<typename _B1, typename _B2>
struct __or_<_B1, _B2>
    : public conditional<_B1::value, _B1, _B2>::type
{ };

template<typename _B1, typename _B2, typename _B3, typename... _Bn>
struct __or_<_B1, _B2, _B3, _Bn...>
    : public conditional<_B1::value, _B1, __or_<_B2, _B3, _Bn...>>::type
{ };
```

首先我们先来了解一下__or_是如何使用的。如果我们如下使用。其中type起始就是我们上面提到的true_type或false_type。

```
__or_<type1, type2, type3, ...>::value
```

那么最终结果和下面是等价的：

```
type1::value || type2::value || type3::value ...
```

OK，让我们看一下__or_是如何实现的。我们应该从下向上看。首先对于

```c++
template<typename _B1, typename _B2, typename _B3, typename... _Bn>
struct __or_<_B1, _B2, _B3, _Bn...>
    : public conditional<_B1::value, _B1, __or_<_B2, _B3, _Bn...>>::type
{ };
```

我们传入从1到n，n个“布尔类型”，我们首先判断_B1::value。如果是true，那么按照上面conditional的定义，最终___or_的type实际上就是_B1的type，即也是true。如果为false，那么最终__or_的type则由__or_\<_B2, _B3, _Bn...\>递归的确定。

上面两个模板参数的__or_思考过程是类似的。

如果递归过程中_B2，_B3，...，_Bn一直是false_type。那么最终我们__or_会得到false_type。因此对于没有模板参数的__or_默认是false_type。


## __and_


```c++
template<typename...>
struct __and_;

template<>
struct __and_<>
    : public true_type
{ };

template<typename _B1>
struct __and_<_B1>
    : public _B1
{ };

template<typename _B1, typename _B2>
struct __and_<_B1, _B2>
    : public conditional<_B1::value, _B2, _B1>::type
{ };

template<typename _B1, typename _B2, typename _B3, typename... _Bn>
struct __and_<_B1, _B2, _B3, _Bn...>
    : public conditional<_B1::value, __and_<_B2, _B3, _Bn...>, _B1>::type
{ };
```

了解了__or_之后__and_就非常简单了。同样我们从下向上看。

如果传入从1到n，n个“布尔类型”，我们首先判断_B1::value。如果是true，那么按照上面conditional的定义，最终由__and_\<_B2, _B3, _Bn...\>递归的确定。这里注意到__and_和__or_模板参数顺序的不同。如果为false，那么最终__and_的type就是_B1的type，即false。

同样，如果递归过程中_B2，_B3，...，_Bn一直是true_type。那么最终我们__and_会得到true_type。因此对于没有模板参数的__and_默认是true_type。


## __not_

```c++
template<typename _Pp>
struct __not_
    : public integral_constant<bool, !_Pp::value>
{ };
```


# enable_if


再介绍一个type_traits中经常被使用到的模板类：enable_if。

enable_if的使用方式如下。其中有两个模板参数，第一个参数为bool类型，第二个为任意类型。只有当
```c++
std::enable_if<false>::type  //出错，只有第一个模板为true才有type

std::enable_if<true>::type  //正确，是void类型

std::enable_if<true, int>::type  //正确，是int类型
```

下面是enable_if的实现：

```c++
template<bool, typename _Tp = void>
struct enable_if
{ };

template<typename _Tp>
struct enable_if<true, _Tp>
{
    typedef _Tp type;
};
```




