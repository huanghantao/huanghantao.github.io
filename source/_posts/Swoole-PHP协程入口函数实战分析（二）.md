---
title: Swoole PHP协程入口函数实战分析（二）
date: 2020-03-22 13:30:16
tags:
- Swoole
- PHP
---

上一篇文章，我们分析了`Swoole PHP`协程入口函数`swoole::PHPCoroutine::main_func`的`zend_first_try`和`zend_catch`，明白了这对结构解决了什么问题。这篇文章，我们继续分析协程入口函数。

我们使用的`PHP`版本是`7.3.12`，`Swoole`的版本是`v4.4.16`。我们把`commit`切到`v4.4.16`：

```bash
git checkout v4.4.16
```

我们继续读代码：

```cpp
int i;
php_coro_args *php_arg = (php_coro_args *) arg;
zend_fcall_info_cache fci_cache = *php_arg->fci_cache;
zend_function *func = fci_cache.function_handler;
zval *argv = php_arg->argv;
int argc = php_arg->argc;
php_coro_task *task;
zend_execute_data *call;
zval _retval, *retval = &_retval;
```

其中：

```cpp
php_coro_args *php_arg = (php_coro_args *) arg;
zend_fcall_info_cache fci_cache = *php_arg->fci_cache;
zend_function *func = fci_cache.function_handler;
zval *argv = php_arg->argv;
int argc = php_arg->argc;
```

其中，

`fci_cache`这个结构是由我们在函数`Coroutine::create`中传递进去的函数生成的。例如：

```php
Coroutine::create(function () {
    echo 'hello swoole' . PHP_EOL;
});
```

就是由这个匿名函数生成的。

`func`则是对应这个匿名函数本体。

`argv`则对应着我们传递给函数的参数，`argc`则是参数的个数。例如：

```php
<?php

use Swoole\Coroutine;

Coroutine::create(function ($arg1, $arg2) {
    echo 'hello swoole' . PHP_EOL;
}, 1, 'arg2');
```

那么`argv[0]`存储的就是这里的整形参数`1`，`argv[1]`存储的就是这里的字符串参数`arg2`。对应的，`argc`就等于`2`。

`_retval`则是用来保存函数的返回值。注意，这里保存的不是`Coroutine::create`这个函数的返回值，而是传递给`Coroutine::create`的函数的返回值（我们把传递进去的函数叫做协程任务函数吧）。

举个例子：

```php
<?php

use Swoole\Coroutine;

Coroutine::create(function ($arg1, $arg2) {
    echo 'hello swoole' . PHP_EOL;
    return 'ret';
}, 1, 'arg2');
```

此时，`_retval`存储的就是字符串`ret`。

我们继续读代码：

```cpp
if (fci_cache.object)
{
    GC_ADDREF(fci_cache.object);
}
```

这段代码解决了什么问题呢？

> 协程任务函数是属于某个对象的话，那么需要给这个对象加引用计数，不然协程发生切换时，`PHP`会默认释放掉这个对象，导致下次协程切换回来发生错误。

我们编写一下测试脚本：

```php
<?php

use Swoole\Coroutine;

class Test
{
    public function func1()
    {
        echo 'hello swoole' . PHP_EOL;
    }
}

Coroutine::create([new Test, 'func1']);
```

此时就会进入`if (fci_cache.object)`的逻辑了。

我们可以注释掉`main_func`的以下代码：

```cpp
if (fci_cache.object)
{
    GC_ADDREF(fci_cache.object);
}
```

```cpp
if (fci_cache.object)
{
    OBJ_RELEASE(fci_cache.object);
}
```

然后重新编译、安装扩展。接着执行这个测试脚本：

```bash
[root@592b0366acbf coroutine]# php test.php
*RECURSION*
[root@592b0366acbf coroutine]# php test.php
Segmentation fault
```

不出意外，会得到这两个错误。我们来跟踪代码的执行流程。

首先，程序会进入`main_func`函数里面，并且调用`zend_execute_ex(EG(current_execute_data));`来执行我们的协程任务函数。`zend_execute_ex`对应的就是`PHP`内核的`execute_ex`这个函数（在文件`zend_vm_execute.h`里面）。接着，执行`ZEND_DO_ICALL_SPEC_RETVAL_UNUSED_HANDLER`这个`PHP`的`handler`。然后，在这个`handler`里面调用`fbc->internal_function.handler(call, ret);`方法，而这个`handler`实际上就是我们的`var_dump`函数了。函数如下：

```c
PHP_FUNCTION(var_dump)
{
    zval *args;
    int argc;
    int i;

    ZEND_PARSE_PARAMETERS_START(1, -1)
        Z_PARAM_VARIADIC('+', args, argc)
    ZEND_PARSE_PARAMETERS_END();

    for (i = 0; i < argc; i++) {
        php_var_dump(&args[i], 1);
    }
}
```

`args`就是我们传递给`var_dump`函数的参数，`argc`则是我们传递给`var_dump`函数的参数个数。如果我们去调试的话，会发现`args[0]`的类型是`object`，也就是我们的`Test`类对象。

然后，调用`php_var_dump`来打印变量信息。此时会进入这个`case`分支：

```c
case IS_OBJECT:
    if (Z_IS_RECURSIVE_P(struc)) {
        PUTS("*RECURSION*\n");
        return;
    }
```
