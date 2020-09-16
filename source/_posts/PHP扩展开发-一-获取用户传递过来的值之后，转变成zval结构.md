---
title: PHP扩展开发(一)--把基本类型的值转变成zval结构
date: 2018-12-18 12:10:25
tags:
- PHP
- PHP 扩展开发
---

我们在开发PHP扩展的时候，经常会有需要把基本类型的值转化为zval结构的需求，这里，我给出两段代码，来完成这个需求。

```c
int fd;
zval *zfd;

fd = 1;
TSW_MAKE_STD_ZVAL(zfd); // Let zfd point to a piece of memory in the stack
ZVAL_LONG(zfd, fd); // Before using ZVAL_LONG, you need to allocate memory first.
```

```c
#define TSW_MAKE_STD_ZVAL(p)		zval _stack_zval_##p; p = &(_stack_zval_##p)
```

`TSW_MAKE_STD_ZVAL`宏的作用就是在栈上创建一块临时的内存，然后让p指针指向这块内存。然后，我们再使用`ZVAL_*`系列的函数来进行转换。