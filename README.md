# STL源码剖析——前言


# 前言

这是对GNU C++实现的STL源码库代码解读。源代码可以从[GNU C++ Library](https://gcc.gnu.org/onlinedocs/libstdc++/)网站获取。

这里使用的是gcc 4.9版本的源代码，这是2014年公布的版本，因此全面支持的C++11特性。

# STL源文件组织

在gcc 4.9版本源代码中，STL代码位于gcc-4.9.0\libstdc++-v3\include路径下。在下文中没有特别说明，此路径就是我们的默认根目录。

其中有三个重要目录std、bits、ext：

std目录。存放C++标准库头文件，包括容器相关文件vector、list、deque、set、map，内存分配相关文件memory、迭代器相关文件iterator，算法相关文件algorithm、numeric、仿函数相关文件functional等。这里的文件并不存放实际实现代码，只是作为标准库头文件存在，include所有相关头文件以便用户使用。如\<vector\>内容如下：

```c++
/** @file include/vector
 *  This is a Standard C++ Library header.
 */

#ifndef _GLIBCXX_VECTOR
#define _GLIBCXX_VECTOR 1

#pragma GCC system_header

#include <bits/stl_algobase.h>
#include <bits/allocator.h>
#include <bits/stl_construct.h>
#include <bits/stl_uninitialized.h>
#include <bits/stl_vector.h>
#include <bits/stl_bvector.h> 
#include <bits/range_access.h>

#ifndef _GLIBCXX_EXPORT_TEMPLATE
# include <bits/vector.tcc>
#endif

#ifdef _GLIBCXX_DEBUG
# include <debug/vector>
#endif

#ifdef _GLIBCXX_PROFILE
# include <profile/vector>
#endif

#endif /* _GLIBCXX_VECTOR */
```


其中_GLIBCXX_DEBUG表示调试模式，默认不打开。_GLIBCXX_PROFILE表示配置文件模式，默认不打开。

从\<vector\>中导入的头文件能够看到，几乎所有文件都来自bits目录。bits中存放C++标准库的实现源代码。如上面\<vector\>头文件中导入的\<bits/stl_vector.h\>就是vector的实现代码。

另外ext目录存放的是GNU对C++标准库的扩展文件。如GNU提供了各种不同的allocator。

我们剖析源代码主要位于bits目录。

bits目录下的\<c++config\>文件需要格外关注，其中定义了很多标准库可能用到的宏

STL源代码中经常能够看到__cplusplus宏，其指明了C++的版本号。

# 源码剖析目录

- [类型萃取器]
  - [type_traits](type_traits.md)
- [内存分配器]
  - [allocator](allocator.md)
  - [new_allocator](new_allocator.md)
  - [alloc_traits](alloc_traits.md)
- [迭代器]
  - [stl_iterator_base_types.h](stl_iterator_base_types.md)
  - [stl_iterator.h](stl_iterator.md)
- [容器]
  - [stl_vector](stl_vector.md)





































































