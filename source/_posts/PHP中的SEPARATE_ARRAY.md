---
title: PHP中的SEPARATE_ARRAY
date: 2020-07-11 21:34:56
tags:
- PHP
---

> 本篇文章基于PHP的commit为：9fa1d1330138ac424f990ff03e62721120aaaec3

在`PHP`内核里面，有一个叫做`SEPARATE_ARRAY`的宏。长这样：

```cpp
#define SEPARATE_ARRAY(zv) do {                     \
    zval *_zv = (zv);                               \
    zend_array *_arr = Z_ARR_P(_zv);                \
    if (UNEXPECTED(GC_REFCOUNT(_arr) > 1)) {        \
        if (Z_REFCOUNTED_P(_zv)) {                  \
            GC_DELREF(_arr);                        \
        }                                           \
        ZVAL_ARR(_zv, zend_array_dup(_arr));        \
    }                                               \
} while (0)
```

一句话来说，这个宏做的事情就是分离`zend_array`。我们知道，`PHP`是通过引用计数来管理多个变量对数组的引用，如果其中一个变量需要去修改数组的内容，那么底层就会单独为这个变量分配一个新的`zend_array`，并且原来的`zend_array`的引用计数减一。然后，`SEPARATE_ARRAY`做的事情就是这个。

因为`PHP`底层实在是有太多需要修改数组的操作了，所以我们确实需要`SEPARATE_ARRAY`来帮助我们去分离数组。

除此之外，我们会在`zend_hash.c`文件的所有写数组的函数里面发现`HT_ASSERT_RC1`这个断言宏。这个宏对于写`C`扩展的我们来说，在`debug`上是非常的有帮助的。我们来看看`HT_ASSERT_RC1`这个宏：

```cpp
#define HT_ASSERT_RC1(ht) HT_ASSERT(ht, GC_REFCOUNT(ht) == 1)

#if ZEND_DEBUG
# define HT_ASSERT(ht, expr) \
    ZEND_ASSERT((expr) || (HT_FLAGS(ht) & HASH_FLAG_ALLOW_COW_VIOLATION))
#else
# define HT_ASSERT(ht, expr)
#endif
```

首先，这个宏只会在`PHP`开启`debug`的时候才会起作用。

然后，我们发现，这个断言宏能够成功的情况有两个。一个是设置了`zend_array`的`HASH_FLAG_ALLOW_COW_VIOLATION`标志；第二个是`zend_array`的引用计数是`1`。我们先来说一下第二点，因为第一点和第二点有关系。

为什么`zend_array`的引用计数要是`1`？

因为`PHP`扩展操作数组的函数没法对数组进行分离。我们知道，如果修改一个数组，是需要发生写时复制的（我们也可以叫做写时分离），如果不进行写时复制，那么就会导致其他引用了这个数组的变量出问题（可能这是一个你意想不到的修改）。所以，只要这个数组的引用计数是`1`，我们就可以确定，这个数组只有一个变量在引用，所以我们可以放心的去修改它了。

那么，如果我们非要在引用计数大于`1`的时候去修改这个数组呢？那么第一种情况就起作用了，我们可以设置`HASH_FLAG_ALLOW_COW_VIOLATION`这个标志（翻译过来就是**允许违反写时复制**），来强制不遵守写时复制的规则。
