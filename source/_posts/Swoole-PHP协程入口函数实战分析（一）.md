---
title: Swoole PHP协程入口函数实战分析（一）
date: 2020-03-21 13:31:57
tags:
- Swoole
- PHP
---

本篇文章，我们来实战分析一下`Swoole PHP`协程入口函数的实现原理以及细节。大家需要准备好一份`Swoole`源码好和我们一起动手操作，我们使用的`PHP`版本是`7.3.12`，`Swoole`的版本是`v4.4.16`。

首先，我们需要明白`PHP`协程的入口函数是`swoole::PHPCoroutine::main_func`，也就是说，我们创建的每一个`PHP`协程，都会从`main_func`这个函数开始。

我们现在来逐行分析`main_func`这个函数。首先，这个函数的参数是`void *arg`，是一个`void`类型的指针，从

```cpp
php_coro_args *php_arg = (php_coro_args *) arg;
```

可以看出`arg`原来的类型是`php_coro_args`类型的指针，`php_coro_args`这个结构体对应的成员如下：

```cpp
struct php_coro_args
{
    zend_fcall_info_cache *fci_cache;
    zval *argv;
    uint32_t argc;
};
```

可以发现，这三个成员是我们调用一个函数的基础，分别对应了函数本体、传递给函数的参数、传递给函数参数的个数。

在函数的开头，我们看到了`BAILOUT`这个东西，我们先来看看如果没有这个东西会有什么问题，我们把`commit`切一下：

```bash
git checkout ef1db99ecfa475ce34d4be744d1f811fadf566ac

git reset HEAD~
```

此时我们可以看到为支持`BAILOUT`功能而做出的文件改动。

（现在`main_func`这个函数名字是旧的名字`create_func`）

我们会发现，在`create_func`函数里面，增加了`zend_first_try`和`zend_catch`这个结构。我们很容易的想到这是用来捕获`PHP`异常的。

并且增加了`swoole::Coroutine::bailout`这个函数，这个函数会在`zend_catch`里面被调用：

```cpp
zend_catch {
    Coroutine::bailout([](){ sw_zend_bailout(); });
} zend_end_try();
```

我们来看看`bailout`这个函数做了什么事情：

```cpp
void Coroutine::bailout(sw_coro_bailout_t func)
{
    if (!func)
    {
        swError("bailout without bailout function");
    }
    if (!current)
    {
        func();
    }
    else
    {
        Coroutine *co = current;
        while (co->origin)
        {
            co = co->origin;
        }
        // it will jump to main context directly (it also breaks contexts)
        on_bailout = func;
        co->yield();
    }
    // expect that never here
    exit(1);
}
```

其中：

```cpp
if (!current)
{
    func();
}
```

表示如果不在协程环境，那么直接执行函数`func`。如果在协程环境，那么执行：

```cpp
Coroutine *co = current;
while (co->origin)
{
    co = co->origin;
}
// it will jump to main context directly (it also breaks contexts)
on_bailout = func;
co->yield();
```

表示查找到主协程，然后再把函数`on_bailout = func`赋值给`swoole::Coroutine::on_bailout`这个函数指针。

然后调用`co->yield()`切换协程上下文。这个时候，就回到非协程环境了。

现在，我们来看两个问题，第一个问题是这里的函数`func`是什么，第二个问题是`on_bailout`在哪里被调用了。

首先来看第一个问题。函数`func`是：

```cpp
sw_zend_bailout();
```

对应着：

```cpp
#define sw_zend_bailout() zend_bailout()

BEGIN_EXTERN_C()
ZEND_API ZEND_COLD void _zend_bailout(const char *filename, uint32_t lineno) /* {{{ */
{

    if (!EG(bailout)) {
        zend_output_debug_string(1, "%s(%d) : Bailed out without a bailout address!", filename, lineno);
        exit(-1);
    }
    gc_protect(1);
    CG(unclean_shutdown) = 1;
    CG(active_class_entry) = NULL;
    CG(in_compilation) = 0;
    EG(current_execute_data) = NULL;
    LONGJMP(*EG(bailout), FAILURE);
}
/* }}} */
END_EXTERN_C()
```

`zend_bailout`函数是用来结束程序运行的。我们重点关注`LONGJMP(*EG(bailout), FAILURE)`做了什么事情。我们知道在`C`语言里面，有一对函数`setjmp`和`longjmp`，分别用来保存当前程序运行的环境和恢复被保存的环境。

我们来看看宏`LONGJMP`展开来是什么东西：

```cpp
# define LONGJMP(a,b) longjmp(a, b)
```

就是我们的`longjmp`函数。

我们可以看一下`longjmp`这个函数的描述：

```shell
Jump to the environment saved in ENV, making the setjmp call there return VAL, or 1 if VAL is 0.

翻译：跳转到在ENV中保存的环境，使setjmp调用在那里返回VAL，如果VAL为0则返回1。
```

所以，`LONGJMP(*EG(bailout), FAILURE)`的意图就比较明显了，程序会跳转到`*EG(bailout)`上下文中，然后使`SETJMP`调用在那里返回`FAILURE`，也就是`-1`。

我们再来看第二个问题。`on_bailout`是在函数`swoole::Coroutine::check_end`中被调用的：

```cpp
inline void check_end()
{
    if (ctx.is_end())
    {
        close();
    }
    if (unlikely(on_bailout))
    {
        on_bailout();
    }
}
```

也就是说，当`on_bailout`这个函数指针不为空的时候，会去调用这个`on_bailout`，也就是函数`sw_zend_bailout`。并且，只有当程序逻辑进入了：

```cpp
zend_catch {
    Coroutine::bailout([](){ sw_zend_bailout(); });
} zend_end_try();
```

的时候（即程序抛出了异常），这个`on_bailout`指针才不为空。

现在，我们执行如下命令：

```bash
git reset --hard a0384ea2981125fc9a7e1a68e489ffb5b40ad426
```

此时，我们的`Swoole`就处于没有`bailout`的样子了。我们重新编译、安装扩展后，编写如下测试脚本：

```php
<?php

register_shutdown_function(function () {
    echo 'shutdown' . PHP_EOL;
});

echo 'execute script' . PHP_EOL;

throw new Exception();
```

我们执行脚本：

```bash
[root@592b0366acbf bailout]# php error.php
execute script
PHP Fatal error:  Uncaught Exception in /root/codeDir/phpCode/swoole/coroutine/bailout/error.php:9
Stack trace:
#0 {main}
  thrown in /root/codeDir/phpCode/swoole/coroutine/bailout/error.php on line 9

Fatal error: Uncaught Exception in /root/codeDir/phpCode/swoole/coroutine/bailout/error.php:9
Stack trace:
#0 {main}
  thrown in /root/codeDir/phpCode/swoole/coroutine/bailout/error.php on line 9
shutdown
[root@592b0366acbf bailout]#
```

可以发现，在脚本结束执行后，会调用我们通过`register_shutdown_function`函数注册的匿名函数，打印出字符串`shutdown`。

我们现在修改脚本：

```php
<?php

register_shutdown_function(function () {
    echo 'shutdown' . PHP_EOL;
});

echo 'execute script' . PHP_EOL;

go(function () {
    throw new Exception;
});
```

然后执行脚本：

```bash
[root@592b0366acbf bailout]# php error.php
execute script
PHP Fatal error:  Uncaught Exception in /root/codeDir/phpCode/swoole/coroutine/bailout/error.php:10
Stack trace:
#0 {main}
  thrown in /root/codeDir/phpCode/swoole/coroutine/bailout/error.php on line 10

Fatal error: Uncaught Exception in /root/codeDir/phpCode/swoole/coroutine/bailout/error.php:10
Stack trace:
#0 {main}
  thrown in /root/codeDir/phpCode/swoole/coroutine/bailout/error.php on line 10
/root/php-7.3.12/main/main.c(1414) : Bailed out without a bailout address!
[root@592b0366acbf bailout]#
```

可以发现，此时我们通过`register_shutdown_function`函数注册的匿名函数无法被执行。第二个脚本和第一个脚本的区别就是第二个脚本在`PHP`协程里面抛出了异常。

为什么我们`PHP`协程里面跑出异常会出现这个问题呢？我们需要跟踪一下程序的执行流程。首先，当我们的程序跑出异常的时候，会使得协程入口函数`create_func`进入如下代码：

```cpp
if (UNEXPECTED(EG(exception)))
{
    zend_exception_error(EG(exception), E_ERROR);
}
```

之后，程序就会调用`php_error_cb`，在这个函数里面打印出异常信息。

之后，程序就会执行如下代码：

```c
if (type != E_PARSE) {
    /* restore memory limit */
    zend_set_memory_limit(PG(memory_limit));
    efree(buffer);
    zend_objects_store_mark_destructed(&EG(objects_store));
    zend_bailout();
    return;
}
```

我们发现，这里调用了`zend_bailout`函数，我们上面分析了这个函数，它会调用`LONGJMP`，然后把上下文切换到`EG(bailout)`。因为这里的`EG(bailout)`是空指针，所以`zend_bailout`这个函数自然就会报错了。

所以，`Swoole`的解决方案就是，在`create_func`这个协程入口函数体里面包了一层`try ... catch`。

我们切换一下`commit`：

```cpp
git checkout ef1db99ecfa475ce34d4be744d1f811fadf566ac

git reset HEAD~
```

然后重新编译、安装扩展。

编写如下测试脚本：

```php
<?php

register_shutdown_function(function () {
    echo 'shutdown' . PHP_EOL;
});

echo 'execute script' . PHP_EOL;

go(function () {
    throw new Exception;
});
```

执行脚本：

```bash
[root@592b0366acbf bailout]# php error.php
execute script
PHP Fatal error:  Uncaught Exception in /root/codeDir/phpCode/swoole/coroutine/bailout/error.php:10
Stack trace:
#0 {main}
  thrown in /root/codeDir/phpCode/swoole/coroutine/bailout/error.php on line 10

Fatal error: Uncaught Exception in /root/codeDir/phpCode/swoole/coroutine/bailout/error.php:10
Stack trace:
#0 {main}
  thrown in /root/codeDir/phpCode/swoole/coroutine/bailout/error.php on line 10
shutdown
[root@592b0366acbf bailout]#
```

我们发现，此时`register_shutdown_function`注册的匿名函数被执行了。也就是说，程序没有因为我们抛出异常使得`php cli`提早终止执行了。

我们现在来跟踪一下程序的执行流程。

首先，程序会执行`zend_first_try`，而这个宏会去设置`EG(bailout)`的地址，这个地址就在`create_func`里面，并且会保存原来的`EG(bailout)`地址。

然后，我们的`PHP`脚本退出时候，程序会执行到以下代码：

```cpp
if (UNEXPECTED(EG(exception)))
{
    zend_exception_error(EG(exception), E_ERROR);
}
```

然后程序依然会执行函数`php_error_cb`来打印出异常信息。

然后程序会执行函数`zend_bailout`：

```cpp
ZEND_API ZEND_COLD void _zend_bailout(const char *filename, uint32_t lineno) /* {{{ */
{
    if (!EG(bailout)) {
        zend_output_debug_string(1, "%s(%d) : Bailed out without a bailout address!", filename, lineno);
        exit(-1);
    }
    gc_protect(1);
    CG(unclean_shutdown) = 1;
    CG(active_class_entry) = NULL;
    CG(in_compilation) = 0;
    EG(current_execute_data) = NULL;
    LONGJMP(*EG(bailout), FAILURE);
    }
```

因为`EG(bailout)`不在为空了，所以程序会执行到代码：

```cpp
LONGJMP(*EG(bailout), FAILURE)
```

此时，程序会回到`Swoole`内核`create_func`的`zend_first_try`宏里面。然后进入`zend_catch`：

```cpp
zend_catch {
    Coroutine::bailout([](){ sw_zend_bailout(); });
} zend_end_try();
```

然后执行函数`swoole::Coroutine::bailout`的如下代码：

```cpp
else
{
    Coroutine *co = current;
    while (co->origin)
    {
        co = co->origin;
    }
    // it will jump to main context directly (it also breaks contexts)
    on_bailout = func;
    co->yield();
}
```

因为，我们现在就处于第一个协程里面，所以`co->origin`为`nullptr`。接着，程序执行代码：

```cpp
on_bailout = func;
```

实际上就是：

```cpp
on_bailout = sw_zend_bailout;
```

接着，程序执行`co->yield()`，切换上下文，此时，就会到非协程环境。然后程序就执行到了函数`swoole::Coroutine::run`里面的`check_end()`这个位置：

```cpp
inline long run()
{
    long cid = this->cid;
    origin = current;
    current = this;
    ctx.swap_in();
    check_end();
    return cid;
}
```

我们前面分析过，`check_end`这个函数会去调用`on_bailout`回调函数，也就是`sw_zend_bailout`。而`sw_zend_bailout`对应着`zend_bailout`：

```c
ZEND_API ZEND_COLD void _zend_bailout(const char *filename, uint32_t lineno) /* {{{ */
{

    if (!EG(bailout)) {
        zend_output_debug_string(1, "%s(%d) : Bailed out without a bailout address!", filename, lineno);
        exit(-1);
    }
    gc_protect(1);
    CG(unclean_shutdown) = 1;
    CG(active_class_entry) = NULL;
    CG(in_compilation) = 0;
    EG(current_execute_data) = NULL;
    LONGJMP(*EG(bailout), FAILURE);
}
```

此时的`EG(bailout)`地址已经不再是`create_func`函数的`zend_first_try`里面了，而是`zend_first_try`中的`__orig_bailout`。

所以，当程序执行完`LONGJMP(*EG(bailout), FAILURE)`之后，就会回到`php cli`的`php_execute_script`函数的`zend_try`里面：

```c
zend_try {
    char realfile[MAXPATHLEN];

#ifdef PHP_WIN32
    if(primary_file->filename) {
        UpdateIniFromRegistry((char*)primary_file->filename);
    }
#endif
```

接着，`php cli`程序可以顺利的回到`do_cli`函数的：

```c
out:
if (request_started) {
    php_request_shutdown((void *) 0);
}
```

这个位置。

执行完函数`php_request_shutdown`之后，我们在`PHP`脚本里面通过`register_shutdown_function`函数注册的匿名函数就会被执行。最终，`php cli`程序正常退出。

## 总结

`Swoole`内核通过`zend_first_try`保存当前地址到`EG(bailout)`，使得程序在抛出异常的时候，程序先回到`Swoole`内核的`zend_first_try`里面。然后，恢复`EG(bailout)`为`__orig_bailout`。最终，使得`php cli`程序正常退出。
