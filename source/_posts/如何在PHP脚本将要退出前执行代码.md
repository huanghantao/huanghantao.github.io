---
title: 如何在PHP脚本将要退出前执行代码
date: 2020-02-22 10:59:21
tags:
- PHP
- PHP扩展开发
---

这篇文章，我们来介绍下如何通过`PHP`扩展在`PHP`脚本将要退出前执行代码。我们可以看一段`Swoole`的协程代码：

```php
<?php

use Swoole\Coroutine;

go(function () {
    go(function () {
        Coroutine::sleep(2);
        var_dump("1");
    });
    go(function () {
        Coroutine::sleep(2);
        var_dump("2");
    });
    var_dump("3");
});
var_dump("4");
```

执行结果如下：

```shell
string(1) "3"
string(1) "4"
string(1) "1"
string(1) "2"
```

我们发现，在打印`4`之后，`PHP`脚本算是执行完了，这个时候，`PHP`进程也快要退出了。那为什么还会打印出`1`和`2`呢？

因为`PHP`是一门解释性语言，虽然`PHP`脚本的代码跑完了，但是`PHP`命令解释器还没有跑完，自然可以让`PHP`代码继续跑。所以，实际上，这个代码会这样：

```php
<?php

use Swoole\Coroutine;
use Swoole\Event;

go(function () {
    go(function () {
        Coroutine::sleep(2);
        var_dump("1");
    });
    go(function () {
        Coroutine::sleep(2);
        var_dump("2");
    });
    var_dump("3");
});
var_dump("4");

Event::wait();
```

在脚本的最后会调用`Event::wait()`来等待事件的结束。（而这里的事件就是定时器的事件）

好的，我们现在通过一个小`demo`来实现这个功能。

首先，我们需要创建`PHP`扩展的基础骨架。通过`ext_skel`工具生成：

```shell
[root@64fa874bf7d4 ext]# php ext_skel.php --ext register
```

然后进入`register`目录：

```shell
[root@64fa874bf7d4 ext]# cd register/
```

替换`config.m4`为如下内容：

```shell
PHP_ARG_ENABLE(register, whether to enable register support,
Make sure that the comment is aligned:
[  --enable-register           Enable register support])
​
if test "$PHP_REGISTER" != "no"; then
  PHP_NEW_EXTENSION(register, register.c, $ext_shared,, -DZEND_ENABLE_STATIC_TSRMLS_CACHE=1)
fi
```

然后，我们创建和编写测试脚本：

```shell
[root@64fa874bf7d4 register]# touch test.php
```

```php
<?php

function test() {
    var_dump("codinghuang");
}
```

这段脚本只定义了一个`test`函数，并没有调用。现在我们的任务就是去调用它。

我们开始编写`PHP`扩展。在文件`register.c`的`PHP_RINIT_FUNCTION`里面注册`test`函数：

```c
#include "ext/standard/basic_functions.h"
#include "zend_API.h"

PHP_RINIT_FUNCTION(register)
{
#if defined(ZTS) && defined(COMPILE_DL_REGISTER)
	ZEND_TSRMLS_CACHE_UPDATE();
#endif

	php_shutdown_function_entry shutdown_function_entry;
    shutdown_function_entry.arg_count = 1;
    shutdown_function_entry.arguments = (zval *) safe_emalloc(sizeof(zval), 1, 0);
    ZVAL_STRING(&shutdown_function_entry.arguments[0], "test");
    register_user_shutdown_function("test", ZSTR_LEN(Z_STR(shutdown_function_entry.arguments[0])), &shutdown_function_entry);

	return SUCCESS;
}
```

这里，我们首先对`php_shutdown_function_entry`结构进行初始化，`php_shutdown_function_entry.arguments`的第一个位置填函数的名字。`shutdown_function_entry.arg_count`填写1，因为函数名字也算做是`arguments`。初始化完`php_shutdown_function_entry`之后，我们调用`register_user_shutdown_function`函数即可注册`test`函数了。这样，就会在`php`请求`shutdown`阶段调用我们注册的函数了。

然后编译、安装扩展：

```shell
[root@64fa874bf7d4 register]# phpize && ./configure && make && make install
```

然后把扩展在`php.ini`文件里面进行开启：

```ini
; Enable zlib extension module
extension=register.so
```

然后执行脚本：

```shell
[root@64fa874bf7d4 register]# php test.php
string(11) "codinghuang"
[root@64fa874bf7d4 register]#
```

我们发现，成功的调用了`test`函数。

这让我想起了我之前面试腾讯的时候，有一道题目，说是如何在每一个`PHP`函数调用之前，都执行一段代码。这个问题以后补上。
