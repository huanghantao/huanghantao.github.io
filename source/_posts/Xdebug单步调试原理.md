---
title: Xdebug单步调试原理
date: 2020-08-12 22:22:07
tags:
- PHP
- Xdebug
---

这篇文章，我们来分析一下`Xdebug`单步调试的原理。

一句话总结起来就是，`Xdebug`利用`ZEND_EXT_STMT`这个`opcode`来实现了单步调试的功能。

那么，`ZEND_EXT_STMT`这个`opcode`是什么呢？大概可以这么理解，在执行一条语句之前，会执行`ZEND_EXT_STMT`这个`opcode`，这个`opcode`不会对代码的执行结果造成影响，但是它可以帮助我们来实现调试器的功能。

举个例子，有如下的`PHP`代码：

```php
<?php

$a = 1;

$b = 2;

$c = 3;
```

那么，它对应的`opcode`为：

```bash
[root@e2a14c00e7f6 test]# phpdbg test.php
[Welcome to phpdbg, the interactive PHP debugger, v0.5.0]
To get help using phpdbg type "help" and press enter
[Please report bugs to <http://bugs.php.net/report.php>]
[Successful compilation of /root/codeDir/phpCode/swoole/test/test.php]
prompt> b main
[Breakpoint #0 added at main]
prompt> r
[Breakpoint #0 in main() at /root/codeDir/phpCode/swoole/test/test.php:3, hits: 1]
>00003: $a = 1;
 00004:
 00005: $b = 2;
prompt> p
[Stack in /root/codeDir/phpCode/swoole/test/test.php (7 ops)]
L1-8 {main}() /root/codeDir/phpCode/swoole/test/test.php - 0x7f607d0693c0 + 7 ops
 L3    #0     EXT_STMT
 L3    #1     ASSIGN                  $a                   1
 L5    #2     EXT_STMT
 L5    #3     ASSIGN                  $b                   2
 L7    #4     EXT_STMT
 L7    #5     ASSIGN                  $c                   3
 L8    #6     RETURN<-1>              1
prompt>
```

其中，`L1-8`表示的是行数。我们发现，在执行每一条功能性的`opcode`的时候，都会先执行一条`ZEND_EXT_STMT`。

如果你没有开启`Xdebug`，大概率是看不到这个`EXT_STMT`的。也就是说，`Xdebug`做了某些手脚，使得生成的`opcode`里面包含了`EXT_STMT`。我们可以来看一看在哪个地方对生成的`opcode`进行了修改。

首先，我们得看一下`PHP`内核是如何为生成的`oparray`插入`EXT_STMT`的：

```cpp
void zend_do_extended_stmt(void) /* {{{ */
{
    zend_op *opline;

    if (!(CG(compiler_options) & ZEND_COMPILE_EXTENDED_STMT)) {
        return;
    }

    opline = get_next_op();

    opline->opcode = ZEND_EXT_STMT;
}
```

通过`zend_do_extended_stmt`这个函数来实现的。我们看到，只有当`CG(compiler_options)`开启了`ZEND_COMPILE_EXTENDED_STMT`标志，才会为`oparray`插入`EXT_STMT opcode`。

然后，我们发现，`Xdebug`里面，就有添加`ZEND_COMPILE_EXTENDED_STMT`标志的代码：

```cpp
PHP_RINIT_FUNCTION(xdebug)
{
    // 省略其他代码

    /* Only enabled extended info when it is not disabled */
    CG(compiler_options) = CG(compiler_options) | ZEND_COMPILE_EXTENDED_STMT;

    // 省略其他代码

    return SUCCESS;
}
```

如果你把这一行代码注释掉，那么生成的`opcode`就不会带有`EXT_STMT`了，并且`Xdebug`的单步调试功能会失效。

`OK`，介绍完了`ZEND_EXT_STMT`之后，我们来看一看他对应的`handler`：

```cpp
static ZEND_VM_COLD ZEND_OPCODE_HANDLER_RET ZEND_FASTCALL ZEND_EXT_STMT_SPEC_HANDLER(ZEND_OPCODE_HANDLER_ARGS)
{
    USE_OPLINE

    if (!EG(no_extensions)) {
        SAVE_OPLINE();
        zend_llist_apply_with_argument(&zend_extensions, (llist_apply_with_arg_func_t) zend_extension_statement_handler, execute_data);
        ZEND_VM_NEXT_OPCODE_CHECK_EXCEPTION();
    }
    ZEND_VM_NEXT_OPCODE();
}
```

我们发现，这个函数会去调用`zend_extensions`的`zend_extension_statement_handler`函数。而这个函数实际上就是`xdebug_statement_call`，它在`zend_extension_entry`里面被注册了。

`xdebug_statement_call`这个函数做的事情就是**阻塞**读取客户端发来的命令。

所以，在客户端发来命令之前，是不会执行`ZEND_EXT_STMT`后面的语句的。这就给了我们一种单步调试的感觉了。

明白了这个原理之后，我们完全可以自己写一个调试器了，有时间我写一个`demo`出来给大家分享下。
