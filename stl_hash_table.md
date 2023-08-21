# stl_hashtable.h


>批注：这里我们参考的是gcc 3.0版本源码，与gcc 4.9版本差别比较大

>文件路径：ext/hashtable.h


# 概述

gcc 3.0版本的hashtable并没有正是加入stl，而是作为stl扩展实现。但gcc 4.9版本中已经正式加入stl。我们这里对3.0版本源码进行解析。

文件中主要定义了hashtable以及对应的迭代器_Hashtable_iterator和_Hashtable_const_iterator。

在解读源码前我们需要明白gcc中hashtable是通过开链的方式解决哈希冲突的，即将哈希映射后位置相同的元素使用链表连接。


# _Hashtable_node

hashnode表示的就是hashtable中链表节点。并没有使用stl中现有的list实现，而是自己定义了链表节点。

```c++
template <class _Val>
struct _Hashtable_node
{
    _Hashtable_node* _M_next;
    _Val _M_val;
};
```

# _Hashtable_iterator

_Hashtable_iterator是hashtable的iterator。其中_M_cur成员用于指向当前指向的节点，_M_ht成员表示当前迭代器对应的hashtable，用于在遍历到某一个buckets结尾时跳转到下一个buckets。

我们介绍一下下面模板参数的含义：
- _Val：表示键值对，注意hashtable存储的是键值对。_Val表示的是整个键值对，也包含了键
- _Key：表示索引键
- _HashFcn：函数类型，表示哈希函数
- _ExtractKey：函数类型，从_Val对象中提取key的函数
- _EqualKey：函数类型，用于对比key是否相等
- _Alloc：内存分配器

```c++
template <class _Val, class _Key, class _HashFcn, class _ExtractKey, class _EqualKey, class _Alloc>
struct _Hashtable_iterator {
    typedef hashtable<_Val, _Key, _HashFcn, _ExtractKey, _EqualKey, _Alloc>
        _Hashtable;
    typedef _Hashtable_iterator<_Val, _Key, _HashFcn, _ExtractKey, _EqualKey, _Alloc>
        iterator;
    typedef _Hashtable_const_iterator<_Val, _Key, _HashFcn, _ExtractKey, _EqualKey, _Alloc>
        const_iterator;
    typedef _Hashtable_node<_Val> _Node;

    typedef forward_iterator_tag iterator_category;
    typedef _Val value_type;
    typedef ptrdiff_t difference_type;
    typedef size_t size_type;
    typedef _Val& reference;
    typedef _Val* pointer;

    _Node* _M_cur;  //聚合_Hashtable_node以指向当前节点
    _Hashtable* _M_ht;  //聚合hashtable结构以在不同buckets上转移

    _Hashtable_iterator(_Node* __n, _Hashtable* __tab)
        : _M_cur(__n), _M_ht(__tab) {}
    _Hashtable_iterator() {}

    reference operator*() const { return _M_cur->_M_val; }
    pointer operator->() const { return &(operator*()); }

    iterator& operator++();
    iterator operator++(int);

    bool operator==(const iterator& __it) const
    {
        return _M_cur == __it._M_cur;
    }
    bool operator!=(const iterator& __it) const
    {
        return _M_cur != __it._M_cur;
    }
};
```


我们需要注意hashtable的迭代器只有前进操作，不能向后遍历也没有逆向迭代器。因为我们的node节点只有next指针，buckets构成的是单向链表。

下面是_Hashtable_iterator的前进操作。其中_M_bkt_num是hashtable成员函数，用于给定


```c++
template <class _Val, class _Key, class _HF, class _ExK, class _EqK, class _All>
_Hashtable_iterator<_Val, _Key, _HF, _ExK, _EqK, _All>&
    _Hashtable_iterator<_Val, _Key, _HF, _ExK, _EqK, _All>::operator++()
{
    const _Node* __old = _M_cur;
    _M_cur = _M_cur->_M_next;
    if (!_M_cur) {
        size_type __bucket = _M_ht->_M_bkt_num(__old->_M_val);
        while (!_M_cur && ++__bucket < _M_ht->_M_buckets.size())
            _M_cur = _M_ht->_M_buckets[__bucket];
    }
    return *this;
}

template <class _Val, class _Key, class _HF, class _ExK, class _EqK, class _All>
inline _Hashtable_iterator<_Val, _Key, _HF, _ExK, _EqK, _All>
    _Hashtable_iterator<_Val, _Key, _HF, _ExK, _EqK, _All>::operator++(int)
{
    iterator __tmp = *this;
    ++* this;
    return __tmp;
}
```

_Hashtable_const_iterator和_Hashtable_iterator定义几乎一致，不再赘述。

# hashtable



## buckets大小

STL中的buckets大小总是被设置为质数。之所以这么选择是如果我们的哈希函数不够好，索引键映射后的值不够随机。那么一个质数的buckets大小可以让数据分布更均匀。如果我们哈希函数足够好，实际上buckets取什么值并不重要。

下面是28个质数，我们在设置buckets数量时总是从下面质数中选择。

```c++
// Note: 假设long类型最少32bit
enum { __stl_num_primes = 28 };

static const unsigned long __stl_prime_list[__stl_num_primes] =
{
    53ul,         97ul,         193ul,       389ul,       769ul,
    1543ul,       3079ul,       6151ul,      12289ul,     24593ul,
    49157ul,      98317ul,      196613ul,    393241ul,    786433ul,
    1572869ul,    3145739ul,    6291469ul,   12582917ul,  25165843ul,
    50331653ul,   100663319ul,  201326611ul, 402653189ul, 805306457ul,
    1610612741ul, 3221225473ul, 4294967291ul
};

inline unsigned long __stl_next_prime(unsigned long __n)
{
    const unsigned long* __first = __stl_prime_list;
    const unsigned long* __last = __stl_prime_list + (int)__stl_num_primes;
    const unsigned long* pos = lower_bound(__first, __last, __n);  //lower_bound是泛型算法
    return pos == __last ? *(__last - 1) : *pos;
}
```

## define


```c++
template <class _Val, class _Key, class _HashFcn, class _ExtractKey, class _EqualKey, class _Alloc>
class hashtable {
    //...
};
```


## 类型成员

这是hashtable公有类型成员的定义。

```c++
public:
    typedef _Key key_type;
    typedef _Val value_type;
    typedef _HashFcn hasher;
    typedef _EqualKey key_equal;

    typedef size_t            size_type;
    typedef ptrdiff_t         difference_type;
    typedef value_type* pointer;
    typedef const value_type* const_pointer;
    typedef value_type& reference;
    typedef const value_type& const_reference;

public:
    typedef _Hashtable_iterator<_Val, _Key, _HashFcn, _ExtractKey, _EqualKey, _Alloc>
        iterator;
    typedef _Hashtable_const_iterator<_Val, _Key, _HashFcn, _ExtractKey, _EqualKey,  _Alloc>
        const_iterator;

    friend struct
        _Hashtable_iterator<_Val, _Key, _HashFcn, _ExtractKey, _EqualKey, _Alloc>;
    friend struct
        _Hashtable_const_iterator<_Val, _Key, _HashFcn, _ExtractKey, _EqualKey, _Alloc>;
```


## 数据成员

这是hashtable的数据成员。除了模板参数中涉及到的_HashFcn、_ExtractKey、_EqualKey、_Alloc我们分别定义了对应的数据成员外。hashtable还定义了_M_buckets用于数据存储，可以看到_M_buckets是元素类型为node指针的vector容器。容器中每一个元素实际上都是一条链表。另外_M_num_elements表示当前hashtable中的元素数量，我们需要记录元素数量以用于判断是否应该对buckets扩充。

```c++
private:
    hasher                _M_hash;
    key_equal             _M_equals;
    _ExtractKey           _M_get_key;
    vector<_Node*, _Alloc> _M_buckets;
    size_type             _M_num_elements;

    typename _Alloc_traits<_Node, _Alloc>::allocator_type _M_node_allocator;
```


## --

下面_M_get_node和_M_put_node用于分配和释放node内存。

```c++
public:
    hasher hash_funct() const { return _M_hash; }
    key_equal key_eq() const { return _M_equals; }

private:
    typedef _Hashtable_node<_Val> _Node;

public:
    typedef typename _Alloc_traits<_Val, _Alloc>::allocator_type allocator_type;
    allocator_type get_allocator() const { return _M_node_allocator; }

private:
    _Node* _M_get_node() { return _M_node_allocator.allocate(1); }
    void _M_put_node(_Node* __p) { _M_node_allocator.deallocate(__p, 1); }
```


## ctor/dtor/operator=/swap

下面是hashtable构造、拷贝构造、析构以及operator=运算符重载函数。函数本身没有太多需要说明的。其中使用到的_M_initialize_buckets用来初始化hashtable内存，以存储至少__n个元素。_M_copy_from用来从另一个hashtable拷贝元素到当前hashtable。

```c++
public:
    hashtable(size_type __n,
        const _HashFcn& __hf,
        const _EqualKey& __eql,
        const _ExtractKey& __ext,
        const allocator_type& __a = allocator_type())
        : _M_node_allocator(__a),
        _M_hash(__hf),
        _M_equals(__eql),
        _M_get_key(__ext),
        _M_buckets(__a),
        _M_num_elements(0)
    {
        _M_initialize_buckets(__n);
    }

    hashtable(size_type __n,
        const _HashFcn& __hf,
        const _EqualKey& __eql,
        const allocator_type& __a = allocator_type())
        : _M_node_allocator(__a),
        _M_hash(__hf),
        _M_equals(__eql),
        _M_get_key(_ExtractKey()),
        _M_buckets(__a),
        _M_num_elements(0)
    {
        _M_initialize_buckets(__n);
    }

    hashtable(const hashtable& __ht)
        : _M_node_allocator(__ht.get_allocator()),
        _M_hash(__ht._M_hash),
        _M_equals(__ht._M_equals),
        _M_get_key(__ht._M_get_key),
        _M_buckets(__ht.get_allocator()),
        _M_num_elements(0)
    {
        _M_copy_from(__ht);
    }

    hashtable& operator= (const hashtable& __ht)
    {
        if (&__ht != this) {
            clear();
            _M_hash = __ht._M_hash;
            _M_equals = __ht._M_equals;
            _M_get_key = __ht._M_get_key;
            _M_copy_from(__ht);
        }
        return *this;
    }

    ~hashtable() { clear(); }

    void swap(hashtable& __ht)
    {
        std::swap(_M_hash, __ht._M_hash);
        std::swap(_M_equals, __ht._M_equals);
        std::swap(_M_get_key, __ht._M_get_key);
        _M_buckets.swap(__ht._M_buckets);
        std::swap(_M_num_elements, __ht._M_num_elements);
    }
```


下面是_M_initialize_buckets函数，从我们给定的28个质数中选择合适的的作为hashtable的大小，并开辟对应大小的buckets空间。注意我们的__n指的是hashtable的元素数量，但这里我们用__n初始化了对应数量的buckets。实际上尽管我们每个buckets可以存储一系列的“碰撞”元素，但是在hashtable插入过程中我们希望尽量保证每个buckets中只有一个元素，这样查询效率最高。在后面hashtable的调整函数中能够验证这个说法。

```c++
private:
    void _M_initialize_buckets(size_type __n)
    {
        const size_type __n_buckets = _M_next_size(__n);
        _M_buckets.reserve(__n_buckets);
        _M_buckets.insert(_M_buckets.end(), __n_buckets, (_Node*)0);
        _M_num_elements = 0;
    }
```

下面是_M_copy_from定义，用于从给定hashtable对象__ht拷贝所有元素到当前hashtable。

```c++
template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
void hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>
    ::_M_copy_from(const hashtable& __ht)
{
    _M_buckets.clear();
    _M_buckets.reserve(__ht._M_buckets.size());
    _M_buckets.insert(_M_buckets.end(), __ht._M_buckets.size(), (_Node*)0);
    __STL_TRY{
        for (size_type __i = 0; __i < __ht._M_buckets.size(); ++__i) {
        const _Node* __cur = __ht._M_buckets[__i];  //__cur指向__ht某一buckets的节点
        if (__cur) {
            _Node* __local_copy = _M_new_node(__cur->_M_val);  //__local_copy是拷贝节点
            _M_buckets[__i] = __local_copy;

            for (_Node* __next = __cur->_M_next;
                __next;
                __cur = __next, __next = __cur->_M_next) {  //遍历__ht当前buckets的所有节点
            __local_copy->_M_next = _M_new_node(__next->_M_val);  //拷贝
            __local_copy = __local_copy->_M_next;  //插入当前hashtable对应buckets
            }
        }
        }
        _M_num_elements = __ht._M_num_elements;
    }
    __STL_UNWIND(clear());
}
```


## size/max_size/empty



```c++
public:
    size_type size() const { return _M_num_elements; }
    size_type max_size() const { return size_type(-1); }
    bool empty() const { return size() == 0; }

public:
    size_type bucket_count() const { return _M_buckets.size(); }

    size_type max_bucket_count() const
    {
        return __stl_prime_list[(int)__stl_num_primes - 1];
    }

    size_type elems_in_bucket(size_type __bucket) const
    {
        size_type __result = 0;
        for (_Node* __cur = _M_buckets[__bucket]; __cur; __cur = __cur->_M_next)
            __result += 1;
        return __result;
    }
```

## begin/end

```c++
public:
    iterator begin()
    {
        for (size_type __n = 0; __n < _M_buckets.size(); ++__n)  //找到第一个非空buckets
            if (_M_buckets[__n])
                return iterator(_M_buckets[__n], this);
        return end();
    }

    iterator end() { return iterator(0, this); }

    const_iterator begin() const
    {
        for (size_type __n = 0; __n < _M_buckets.size(); ++__n)
            if (_M_buckets[__n])
                return const_iterator(_M_buckets[__n], this);
        return end();
    }

    const_iterator end() const { return const_iterator(0, this); }
```

## 结点的new/delete

下面两个辅助函数用于new或delete一个结点。

```c++
private:
    _Node* _M_new_node(const value_type& __obj)
    {
        _Node* __n = _M_get_node();
        __n->_M_next = 0;
        __STL_TRY{
            construct(&__n->_M_val, __obj);
            return __n;
        }
        __STL_UNWIND(_M_put_node(__n));
    }

    void _M_delete_node(_Node* __n)
    {
        destroy(&__n->_M_val);
        _M_put_node(__n);
    }
```


## 插入操作

hashtable中最复杂的便是插入操作。其中插入操作分为两种：
- insert_unique：不允许插入重复的key，如果插入key已经存在直接返回
- insert_equal：允许插入重复的key，如果插入的key已经存在就紧挨着相同的key存储。也就是相同的key存储在一起

下面我们先来看insert_unique。

### insert_unique

insert_unique有两个重载版本，一个是只插入一个元素，一个是传入一对迭代器，将迭代器中元素插入。

```c++
pair<iterator, bool> insert_unique(const value_type& __obj)
{
    resize(_M_num_elements + 1);
    return insert_unique_noresize(__obj);
}

template <class _InputIterator>
void insert_unique(_InputIterator __f, _InputIterator __l)
{
    insert_unique(__f, __l, __iterator_category(__f));
}

template <class _InputIterator>
void insert_unique(_InputIterator __f, _InputIterator __l,
    input_iterator_tag)
{
    for (; __f != __l; ++__f)
        insert_unique(*__f);
}

template <class _ForwardIterator>
void insert_unique(_ForwardIterator __f, _ForwardIterator __l, forward_iterator_tag)
{
    //forward_iterator和input_iterator唯一区别就是可以提前获取需要插入的元素数量，直接一步resize成需要大小
    size_type __n = 0;
    distance(__f, __l, __n);
    resize(_M_num_elements + __n);
    for (; __n > 0; --__n, ++__f)
        insert_unique_noresize(*__f);
}
```

其中传入迭代器的insert就像其他容器一样，区分了input iterator和forward iterator。对于forward iterator我们可以直接算出需要插入的元素数量并一次性将hashtable扩充到需要大小。而input iterator我们只能一个元素一个元素插入，这个过程有可能设计到多次扩容调整。

insert_unique操作中使用到了两个辅助函数，一个是用于hashtable扩容的resize，一个是insert_unique_noresize用于插入元素。

下面是resize函数定义：

```c++
template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
void hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>
::resize(size_type __num_elements_hint)
{
    const size_type __old_n = _M_buckets.size();  //__old_n：原来buckets数量
    if (__num_elements_hint > __old_n) {  //如果需要的元素数 > 原来buckets数。我们按一个buckets对应一个元素进行调整
        const size_type __n = _M_next_size(__num_elements_hint);  //__n：新buckets数量

        if (__n > __old_n) {
            vector<_Node*, _All> __tmp(__n, (_Node*)(0), _M_buckets.get_allocator());  //创建新buckets

            __STL_TRY{
              for (size_type __bucket = 0; __bucket < __old_n; ++__bucket) {  //遍历旧buckets

                _Node* __first = _M_buckets[__bucket];
                while (__first) {  //遍历旧buckets中每一个结点
                  size_type __new_bucket = _M_bkt_num(__first->_M_val, __n);  //获取结点新的bucket下标

                  _M_buckets[__bucket] = __first->_M_next;  //取出first结点
                  __first->_M_next = __tmp[__new_bucket];  //first插入新hashtable对应bucket链表头
                  __tmp[__new_bucket] = __first;  //更新hashtable对应bucket链表头
                  __first = _M_buckets[__bucket];  //first指向下一个旧hashtable结点
                }
              }
              _M_buckets.swap(__tmp);
            }

            /* 如果上面操作出现异常，下面释放tmp */
#         ifdef __STL_USE_EXCEPTIONS
                catch (...) {
                for (size_type __bucket = 0; __bucket < __tmp.size(); ++__bucket) {
                    while (__tmp[__bucket]) {
                        _Node* __next = __tmp[__bucket]->_M_next;
                        _M_delete_node(__tmp[__bucket]);
                        __tmp[__bucket] = __next;
                    }
                }
                throw;
            }
#         endif /* __STL_USE_EXCEPTIONS */
        }
    }
}
```


下面是insert_unique_noresize函数。注意insert_unique_noresize返回值类型是一个pair类型，因为insert_unique可能会插入失败，因此我们使用pair记录插入位置和是否插入成功。

```c++
template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
pair<typename hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::iterator, bool>
hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>
::insert_unique_noresize(const value_type& __obj)
{
    const size_type __n = _M_bkt_num(__obj);  //__n：新插入元素的bucket下标
    _Node* __first = _M_buckets[__n];

    for (_Node* __cur = __first; __cur; __cur = __cur->_M_next)
        if (_M_equals(_M_get_key(__cur->_M_val), _M_get_key(__obj)))  //遇到相同key的元素，返回插入失败
            return pair<iterator, bool>(iterator(__cur, this), false);

    _Node* __tmp = _M_new_node(__obj);
    __tmp->_M_next = __first;
    _M_buckets[__n] = __tmp;
    ++_M_num_elements;
    return pair<iterator, bool>(iterator(__tmp, this), true);
}
```


### insert_equal

insert_equal和insert_unique基本一致。都有两个重载版本，一个用于插入单个元素，一个将传入的迭代器区间元素全部插入。

insert_equal函数没有key的唯一性检查，因此永远能够插入成功。因此只返回插入元素的位置。其余不再赘述。

```c++
iterator insert_equal(const value_type& __obj)
{
    resize(_M_num_elements + 1);
    return insert_equal_noresize(__obj);
}

template <class _InputIterator>
void insert_equal(_InputIterator __f, _InputIterator __l)
{
    insert_equal(__f, __l, __iterator_category(__f));
}

template <class _InputIterator>
void insert_equal(_InputIterator __f, _InputIterator __l,
    input_iterator_tag)
{
    for (; __f != __l; ++__f)
        insert_equal(*__f);
}

template <class _ForwardIterator>
void insert_equal(_ForwardIterator __f, _ForwardIterator __l,
    forward_iterator_tag)
{
    size_type __n = 0;
    distance(__f, __l, __n);
    resize(_M_num_elements + __n);
    for (; __n > 0; --__n, ++__f)
        insert_equal_noresize(*__f);
}
```

下面是辅助函数insert_equal_noresize定义：

```c++
template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
typename hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::iterator
hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>
::insert_equal_noresize(const value_type& __obj)
{
    const size_type __n = _M_bkt_num(__obj);
    _Node* __first = _M_buckets[__n];

    for (_Node* __cur = __first; __cur; __cur = __cur->_M_next)
        //遇到__cur结点与插入元素key相同，那么新元素插入__cur结点之后
        if (_M_equals(_M_get_key(__cur->_M_val), _M_get_key(__obj))) {
            _Node* __tmp = _M_new_node(__obj);
            __tmp->_M_next = __cur->_M_next;
            __cur->_M_next = __tmp;
            ++_M_num_elements;
            return iterator(__tmp, this);
        }

    //没有遇到相同结点，直接插入对应bucket链表头
    _Node* __tmp = _M_new_node(__obj);
    __tmp->_M_next = __first;
    _M_buckets[__n] = __tmp;
    ++_M_num_elements;
    return iterator(__tmp, this);
}
```

## find

下面是用于搜索指定元素的find函数。其中find_or_insert同时兼具搜索和插入功能，如果待搜索元素的key已经存在，那么直接返回对应value。否则就插入。

```c++
public:

    template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
    typename hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::reference
    hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::find_or_insert(const value_type& __obj)
    {
        resize(_M_num_elements + 1);

        size_type __n = _M_bkt_num(__obj);
        _Node* __first = _M_buckets[__n];

        for (_Node* __cur = __first; __cur; __cur = __cur->_M_next)
            if (_M_equals(_M_get_key(__cur->_M_val), _M_get_key(__obj)))
                return __cur->_M_val;

        _Node* __tmp = _M_new_node(__obj);
        __tmp->_M_next = __first;
        _M_buckets[__n] = __tmp;
        ++_M_num_elements;
        return __tmp->_M_val;
    }

    iterator find(const key_type& __key)
    {
        size_type __n = _M_bkt_num_key(__key);
        _Node* __first;
        for (__first = _M_buckets[__n];
            __first && !_M_equals(_M_get_key(__first->_M_val), __key);
            __first = __first->_M_next)
        {
        }
        return iterator(__first, this);
    }

    const_iterator find(const key_type& __key) const
    {
        size_type __n = _M_bkt_num_key(__key);
        const _Node* __first;
        for (__first = _M_buckets[__n];
            __first && !_M_equals(_M_get_key(__first->_M_val), __key);
            __first = __first->_M_next)
        {
        }
        return const_iterator(__first, this);
    }
```


## erase/clear

下面是用于删除元素的erase和clear函数。erase也提供了三个重载版本，一个用于删除给定key的所有元素，一个用于删除迭代器指定元素，一个传入迭代器区间，用于删除区间内所有元素。

### erase

下面erase用于删除给定key的所有元素。删除过程很直观不再赘述。

```c++
template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
typename hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::size_type
hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::erase(const key_type& __key)
{
    const size_type __n = _M_bkt_num_key(__key);  //__n：给定key的bucket下标
    _Node* __first = _M_buckets[__n];  //__first：给定ket对应bucket链表结点指针
    size_type __erased = 0;  //记录删除数量

    if (__first) {
        _Node* __cur = __first;
        _Node* __next = __cur->_M_next;
        /* 下面遍历bucket，删除与给定key相同结点 */
        while (__next) {
            if (_M_equals(_M_get_key(__next->_M_val), __key)) {
                __cur->_M_next = __next->_M_next;
                _M_delete_node(__next);
                __next = __cur->_M_next;
                ++__erased;
                --_M_num_elements;
            }
            else {
                __cur = __next;
                __next = __cur->_M_next;
            }
        }

        /* 注意我们的头节点没有被判断，需要单独判断 */
        if (_M_equals(_M_get_key(__first->_M_val), __key)) {
            _M_buckets[__n] = __first->_M_next;
            _M_delete_node(__first);
            ++__erased;
            --_M_num_elements;
        }
    }
    return __erased;
}
```

下面erase删除迭代器指定元素：

```c++
template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
void hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::erase(const iterator& __it)
{
    _Node* __p = __it._M_cur;  //__p：迭代器指向的结点
    if (__p) {
        const size_type __n = _M_bkt_num(__p->_M_val);  //__n：__p的bucket下标
        _Node* __cur = _M_buckets[__n];  //__cur：__n下标对应的bucket链表指针

        if (__cur == __p) {  //如果__p就是bucket链表头节点，直接删除
            _M_buckets[__n] = __cur->_M_next;
            _M_delete_node(__cur);
            --_M_num_elements;
        }
        else {  //如果__p不是头节点，找到__p并删除
            _Node* __next = __cur->_M_next;
            while (__next) {
                if (__next == __p) {
                    __cur->_M_next = __next->_M_next;
                    _M_delete_node(__next);
                    --_M_num_elements;
                    break;
                }
                else {
                    __cur = __next;
                    __next = __cur->_M_next;
                }
            }
        }
    }
}
```

下面erase用于删除迭代器指定范围内元素。由于这些元素可能跨越多个bucket，因此我们使用辅助函数_M_erase_bucket实现对单个bucket中指定范围元素的删除。


```c++
template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
void hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::erase(iterator __first, iterator __last)
{
    /* 先分别获取first和last迭代器指向元素在哪个bucket */
    size_type __f_bucket = __first._M_cur ? _M_bkt_num(__first._M_cur->_M_val) : _M_buckets.size();
    size_type __l_bucket = __last._M_cur ? _M_bkt_num(__last._M_cur->_M_val) : _M_buckets.size();

    /* 根据__f_bucket和__l_bucket调用_M_erase_bucket删除各个bucket中在区间内元素 */
    if (__first._M_cur == __last._M_cur)
        return;
    else if (__f_bucket == __l_bucket)
        _M_erase_bucket(__f_bucket, __first._M_cur, __last._M_cur);
    else {
        _M_erase_bucket(__f_bucket, __first._M_cur, 0);
        for (size_type __n = __f_bucket + 1; __n < __l_bucket; ++__n)
            _M_erase_bucket(__n, 0);
        if (__l_bucket != _M_buckets.size())
            _M_erase_bucket(__l_bucket, __last._M_cur);
    }
}
```

_M_erase_bucket函数的定义如下。三个参数版本的_M_erase_bucket用于删除指定bucket和[__first, __last)区间内的元素。两个参数版本的_M_erase_bucket删除指定bucket从其实到__last位置（不包含__last）的元素。

```c++
template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
void hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>
::_M_erase_bucket(const size_type __n, _Node* __first, _Node* __last)
{
    _Node* __cur = _M_buckets[__n];
    if (__cur == __first)
        _M_erase_bucket(__n, __last);
    else {
        _Node* __next;
        /* 遍历找到__first的上一结点，此时__cur为__first前驱结点 */
        for (__next = __cur->_M_next; __next != __first; __cur = __next, __next = __cur->_M_next);
        
        /* 下面遍历删除 */
        while (__next != __last) {
            __cur->_M_next = __next->_M_next;
            _M_delete_node(__next);
            __next = __cur->_M_next;
            --_M_num_elements;
        }
    }
}

template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
void hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>
::_M_erase_bucket(const size_type __n, _Node* __last)
{
    _Node* __cur = _M_buckets[__n];
    while (__cur != __last) {
        _Node* __next = __cur->_M_next;
        _M_delete_node(__cur);
        __cur = __next;
        _M_buckets[__n] = __cur;
        --_M_num_elements;
    }
}
```


最后的最后，我们还为const_iterator提供了重载版本的erase。它是由非常量迭代器版本代理实现的。

```c++
template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
inline void
hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::erase(const_iterator __first, const_iterator __last)
{
    erase(iterator(const_cast<_Node*>(__first._M_cur),
        const_cast<hashtable*>(__first._M_ht)),
        iterator(const_cast<_Node*>(__last._M_cur),
            const_cast<hashtable*>(__last._M_ht)));
}


template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
inline void
hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::erase(const const_iterator& __it)
{
    erase(iterator(const_cast<_Node*>(__it._M_cur),
        const_cast<hashtable*>(__it._M_ht)));
}
```


### clear

clear函数将析构所有结点并释放对应内存。

```c++
template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
void hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::clear()
{
    for (size_type __i = 0; __i < _M_buckets.size(); ++__i) {
        _Node* __cur = _M_buckets[__i];
        while (__cur != 0) {
            _Node* __next = __cur->_M_next;
            _M_delete_node(__cur);
            __cur = __next;
        }
        _M_buckets[__i] = 0;
    }
    _M_num_elements = 0;
}
```



## count/equal_range


count函数用于统计等于给定key的元素数量。

```c++
size_type count(const key_type& __key) const
{
    const size_type __n = _M_bkt_num_key(__key);
    size_type __result = 0;

    for (const _Node* __cur = _M_buckets[__n]; __cur; __cur = __cur->_M_next)
        if (_M_equals(_M_get_key(__cur->_M_val), __key))
            ++__result;
    return __result;
}
```

equal_range用于获取等于给定key的元素范围。其包含常量和非常量两个版本。

```c++
template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
pair<typename hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::iterator,
    typename hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::iterator>
    hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::equal_range(const key_type& __key)
{
    typedef pair<iterator, iterator> _Pii;
    const size_type __n = _M_bkt_num_key(__key);  //__n为__key对应bucket下标

    for (_Node* __first = _M_buckets[__n]; __first; __first = __first->_M_next)
        if (_M_equals(_M_get_key(__first->_M_val), __key)) {  //找到第一个key与__key相等的结点
            /* 下面遍历所有key等于__key的结点 */
            for (_Node* __cur = __first->_M_next; __cur; __cur = __cur->_M_next)
                if (!_M_equals(_M_get_key(__cur->_M_val), __key))
                    return _Pii(iterator(__first, this), iterator(__cur, this));
            /* 注意如果bucket最后一个元素key也等于__key，我们返回的结束位置，也就是下一结点位于其他bucket */
            for (size_type __m = __n + 1; __m < _M_buckets.size(); ++__m)
                if (_M_buckets[__m])
                    return _Pii(iterator(__first, this), iterator(_M_buckets[__m], this));
            return _Pii(iterator(__first, this), end());
        }
    return _Pii(end(), end());
}

template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
pair<typename hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::const_iterator,
    typename hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::const_iterator>
    hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::equal_range(const key_type& __key) const
{
    typedef pair<const_iterator, const_iterator> _Pii;
    const size_type __n = _M_bkt_num_key(__key);

    for (const _Node* __first = _M_buckets[__n];
        __first;
        __first = __first->_M_next) {
        if (_M_equals(_M_get_key(__first->_M_val), __key)) {
            for (const _Node* __cur = __first->_M_next;
                __cur;
                __cur = __cur->_M_next)
                if (!_M_equals(_M_get_key(__cur->_M_val), __key))
                    return _Pii(const_iterator(__first, this),
                        const_iterator(__cur, this));
            for (size_type __m = __n + 1; __m < _M_buckets.size(); ++__m)
                if (_M_buckets[__m])
                    return _Pii(const_iterator(__first, this),
                        const_iterator(_M_buckets[__m], this));
            return _Pii(const_iterator(__first, this), end());
        }
    }
    return _Pii(end(), end());
}
```

## 获取key对应bucket下标

我们上面函数中最常用的一个辅助函数就是获取某一key或者value对应的bucket下标。下面是它们的实现。

```c++
private:
    size_type _M_bkt_num_key(const key_type& __key) const
    {
        return _M_bkt_num_key(__key, _M_buckets.size());
    }

    size_type _M_bkt_num(const value_type& __obj) const
    {
        return _M_bkt_num_key(_M_get_key(__obj));
    }

    size_type _M_bkt_num_key(const key_type& __key, size_t __n) const
    {
        return _M_hash(__key) % __n;
    }

    size_type _M_bkt_num(const value_type& __obj, size_t __n) const
    {
        return _M_bkt_num_key(_M_get_key(__obj), __n);
    }
```

## --

还有一些其他的函数罗列在这里。

_M_next_size是对__stl_next_prime函数的封装。

```c++
private:
    size_type _M_next_size(size_type __n) const
    {
        return __stl_next_prime(__n);
    }
```


```c++
template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
bool operator==(const hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>& __ht1,
    const hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>& __ht2)
{
    typedef typename hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>::_Node _Node;
    if (__ht1._M_buckets.size() != __ht2._M_buckets.size())
        return false;
    for (size_t __n = 0; __n < __ht1._M_buckets.size(); ++__n) {
        _Node* __cur1 = __ht1._M_buckets[__n];
        _Node* __cur2 = __ht2._M_buckets[__n];
        for (; __cur1 && __cur2 && __cur1->_M_val == __cur2->_M_val;
            __cur1 = __cur1->_M_next, __cur2 = __cur2->_M_next)
        {
        }
        if (__cur1 || __cur2)
            return false;
    }
    return true;
}

template <class _Val, class _Key, class _HF, class _Ex, class _Eq, class _All>
inline bool operator!=(const hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>& __ht1,
    const hashtable<_Val, _Key, _HF, _Ex, _Eq, _All>& __ht2) {
    return !(__ht1 == __ht2);
}

template <class _Val, class _Key, class _HF, class _Extract, class _EqKey,
    class _All>
inline void swap(hashtable<_Val, _Key, _HF, _Extract, _EqKey, _All>& __ht1,
    hashtable<_Val, _Key, _HF, _Extract, _EqKey, _All>& __ht2) {
    __ht1.swap(__ht2);
}
```




