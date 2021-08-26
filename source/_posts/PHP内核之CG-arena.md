---
title: PHP内核之CG(arena)
date: 2021-08-26 17:02:57
tags:
- PHP内核
---

在`PHP`内核里面，有多处通过`CG(arena)`来分配内存，比如`opcache`序列化`op_array`是从这个上面分配内存的。这块内存是连续的内存，会不断的通过`emalloc`来扩容。并且，我们不需要显示的去释放它，它会在`PHP`的`php_request_shutdown`函数里面调用`zend_arena_destroy(CG(arena))`来释放掉。
