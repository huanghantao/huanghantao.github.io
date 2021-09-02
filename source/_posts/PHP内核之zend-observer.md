---
title: PHP内核之zend_observer
date: 2021-09-02 10:41:37
tags:
- PHP内核
---

如果我们要在调用`PHP`脚本里面的每一个函数前打印一句`hello world`，大致有以下几种做法。（假设我们的扩展叫做`observer`）

## Hook zend_execute_ex

例如，我们可以在`PHP`的模块初始化之前，进行如下的操作：

```cpp
void (*old_zend_execute_ex)(zend_execute_data *execute_data);

void observer_execute_ex(zend_execute_data *execute_data)
{
    // printf("hello world\n");

    old_zend_execute_ex(execute_data);
}

static PHP_MINIT_FUNCTION(observer)
{
    old_zend_execute_ex = zend_execute_ex;
    zend_execute_ex = observer_execute_ex;
    return SUCCESS;
}
```

但是，这种方式会有堆栈溢出的风险。

我们来看看，假设我们有如下的`PHP`代码：

```php
<?php

function foo() {
    static $i = 0;
    $i++;

    var_dump($i);
    foo();
}

foo();
```

在没有`hook zend_execute_ex`的情况下，执行结果如下：

```bash
# 省略其他的打印
int(813536)
int(813537)
int(813538)

Fatal error: Allowed memory size of 134217728 bytes exhausted at Zend/zend_execute.h:208 (tried to allocate 262176 bytes) in /Users/codinghuang/codeDir/cCode/php-src/test.php on line 7
```

可以看到，这里调用了`813538`次`foo`函数，然后进程达到`PHP`内存最大的限制而停下来了。

当我们把`PHP`的内存限制设置到无限的时候，我们会发现，这个`foo`函数可以无限的递归下去。为什么可以无限的递归下去呢？因为这种情况下，PHP有一个无限制的`PHP`调用堆栈。

在`hook`了`zend_execute_ex`的情况下，执行结果如下：

```bash
# 省略其他的打印
int(47597)
int(47598)
int(47599)
[1]    10240 segmentation fault  php test.php
```

为什么这种情况下，就不能无限制的递归下去呢？因为当我们`hook`了`zend_execute_ex`之后，`PHP`的函数调用都放在`C`堆栈上面，也就意味着，受限于`ulimit -s`的值。我机器默认的堆栈大小如下：

```bash
ulimit -s
8192
```

这种情况下，我们只有调大系统的堆栈限制，才能解决堆栈被破坏的问题，例如我们扩大一倍的堆栈大小：

```bash
ulimit -s 16384
```

再次执行脚本：

```bash
# 省略其他的打印
int(95259)
int(95260)
int(95261)
[1]    18466 segmentation fault  php test.php
```

可以看到，`foo`函数调用次数基本上是成倍的增加了。

## Hook opcode的handler

我们可以在`ZendVM`执行函数调用的`opcode`之前，执行一段我们的`handler`：

```cpp
static int do_ucall_handler(zend_execute_data *execute_data) {
    // printf("hello world\n");

    return ZEND_USER_OPCODE_DISPATCH;
}

static PHP_MINIT_FUNCTION(observer)
{
    zend_set_user_opcode_handler(ZEND_DO_UCALL, do_ucall_handler);
    zend_set_user_opcode_handler(ZEND_DO_FCALL_BY_NAME, do_ucall_handler);
    return SUCCESS;
}
```

这个实现方式也是可以无限制的递归调用`foo`函数的，因为它仅仅用到了`PHP`栈去实现`foo`函数的调用。但是这种实现方式有一个问题，就是如何去处理`opcode`的转发，我们可以会把其他扩展`hook`的这个`opcode`对应的`handler`抹掉，从而导致一些出乎意料的问题。并且，这里还有一个很大的问题就是，和`JIT`不兼容。使用`Swoole`的小伙伴们应该知道，`JIT`刚出来的时候，`Swoole`下是无法开启`JIT`的，就是因为`Swoole`去`Hook`了某些`opcode`导致的。

## observer api

这个是新一代的方式，也是目前`PHP8`推荐的一种方式，这种即没有堆栈问题，也不会影响`JIT`。

```cpp
static void observer_begin(zend_execute_data *execute_data) {
    printf("hello world\n");
}

static zend_observer_fcall_handlers observer_handler(zend_execute_data *execute_data) {
    zend_observer_fcall_handlers handlers = {NULL, NULL};

    handlers.begin = observer_begin;
    return handlers;
}

static PHP_MINIT_FUNCTION(observer)
{
    REGISTER_INI_ENTRIES();
    zend_observer_fcall_register(observer_handler);
    return SUCCESS;
}
```
