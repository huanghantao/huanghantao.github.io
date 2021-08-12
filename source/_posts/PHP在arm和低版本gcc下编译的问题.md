---
title: PHP在arm和低版本gcc下编译的问题
date: 2021-08-12 17:30:05
tags:
- PHP
---

如果在编译`php-src`的时候，遇到如下问题：

```bash
error: invalid 'asm': invalid operand prefix '%c'

__asm__ goto(

^
```

那么说明，编译器支持`__asm__ goto`但是不支持`%c`这个新特性。

那么，我们可以在执行完`php-src`的`./configure`脚本之后，在`main/php_config.h`文件的`#define HAVE_ASM_GOTO 1`后面加上`#undef HAVE_ASM_GOTO`。

或者找一个支持这种汇编写法的编译器。
