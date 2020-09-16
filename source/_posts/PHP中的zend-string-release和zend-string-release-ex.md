---
title: PHP中的zend_string_release和zend_string_release_ex
date: 2020-07-29 12:37:39
tags:
- PHP
---

在`PHP`中，释放`zend_string`可以使用`zend_string_release`或者`zend_string_release_ex`。那什么时候应该用`zend_string_release`，什么时候应该用`zend_string_release_ex`呢？一句话总结就是，如果我们不确定这个`zend_string`是不是`persistent`方式分配的内存，那么就用`zend_string_release`，如果我们确定这个`zend_string`是不是以`persistent`方式分配的内存，那么就使用`zend_string_release_ex`，因为`zend_string_release_ex`可以稍微提升性能。我们来看一下这两个函数。

首先是`zend_string_release`：

```cpp
#define pefree(ptr, persistent)  ((persistent)?free(ptr):efree(ptr))

static zend_always_inline void zend_string_release(zend_string *s)
{
    if (!ZSTR_IS_INTERNED(s)) {
        if (GC_DELREF(s) == 0) {
            pefree(s, GC_FLAGS(s) & IS_STR_PERSISTENT);
        }
    }
}
```

这个函数做的事情比较简单，先对`zend_string`的引用计数减一，如果引用计数变为了`0`，那么就会真正的去调用`pefree`释放内存。除此之外，这里还需要判断`zend_string`的分配方式，如果是`persistent`方式分配的，那么调用`free`，否则调用`efree`。

我们再来看看`zend_string_release_ex`：

```cpp
static zend_always_inline void zend_string_release_ex(zend_string *s, int persistent)
{
    if (!ZSTR_IS_INTERNED(s)) {
        if (GC_DELREF(s) == 0) {
            if (persistent) {
                ZEND_ASSERT(GC_FLAGS(s) & IS_STR_PERSISTENT);
                free(s);
            } else {
                ZEND_ASSERT(!(GC_FLAGS(s) & IS_STR_PERSISTENT));
                efree(s);
            }
        }
    }
}
```

他做的事情也比较简单，先对`zend_string`的引用计数减一，如果引用计数变为了`0`，那么判断`persistent`再决定调用哪一个`free`。乍眼一看，似乎`zend_string_release_ex`做的事情比`zend_string_release`还多，多了一个断言，为啥性能就会更好呢？因为这个`ZEND_ASSERT`在`PHP`的非`debug`模式下，是不会执行的。并且，如果对`persistent`进行硬编码，编译器会对`zend_string_release_ex`进行优化：

```cpp
if (persistent) {
    ZEND_ASSERT(GC_FLAGS(s) & IS_STR_PERSISTENT);
    free(s);
} else {
    ZEND_ASSERT(!(GC_FLAGS(s) & IS_STR_PERSISTENT));
    efree(s);
}
```

也就是说，在编译的时候，就已经知道要走哪一个分支了。
