---
title: PHP的emalloc
date: 2019-09-23 20:49:12
tags:
- PHP
- PHP扩展开发
- PHP内核
---

这篇文章，我们来实战操作一下扩展的内存管理，感受一下内存泄漏，加深对`PHP`内存管理的理解。

首先，创建扩展目录：

```shell
~/codeDir/cCode/php-7.1.0/ext # ./ext_skel --extname=memory
```

然后进入目录：

```shell
~/codeDir/cCode/php-7.1.0/ext # cd memory/
```

替换文件`config.m4`为如下内容：

```shell
PHP_ARG_ENABLE(memory, whether to enable memory support,
Make sure that the comment is aligned:
[  --enable-memory           Enable memory support])

if test "$PHP_MEMORY" != "no"; then
  PHP_NEW_EXTENSION(memory, memory.c, $ext_shared,, -DZEND_ENABLE_STATIC_TSRMLS_CACHE=1)
fi
```

然后编辑文件`memory.c`里面的`PHP_FUNCTION(confirm_memory_compiled)`方法：

```c
PHP_FUNCTION(confirm_memory_compiled)
{
  void *foo = malloc(2 * 1024 * 1024);
}
```

这里，我们定义了一个`PHP`函数`confirm_memory_compiled`。它做的事情很简单，就是从堆中申请一块`2M`的内存，并且没有主动释放。

接着，编译、安装扩展：

```shell
~/codeDir/cCode/php-7.1.0/ext/memory # phpize ; ./configure
~/codeDir/cCode/php-7.1.0/ext/memory # make ; make install
```

然后把扩展加入配置文件里面：

```ini
extension=memory.so
```

然后确认是否安装扩展成功：

```shell
~/codeDir/cCode/php-7.1.0/ext/memory # php --ri memory

memory

memory support => enabled
~/codeDir/cCode/php-7.1.0/ext/memory #
```

然后编写测试脚本：

```php
<?php
confirm_memory_compiled();

```

执行脚本：

```shell
~/codeDir/cCode/php-7.1.0/ext/memory # php memory.php
~/codeDir/cCode/php-7.1.0/ext/memory #
```

没有报错，说明我们的脚本正常执行了。

接着，我们启动一个`PHP`自带的服务器：

```shell
~/codeDir/cCode/php-7.1.0/ext/memory # php -S 127.0.0.1:80 -t ./
PHP 7.3.5 Development Server started at Mon Sep 23 13:08:13 2019
Listening on http://127.0.0.1:80
Document root is /root/codeDir/cCode/php-7.1.0/ext/memory
Press Ctrl-C to quit.

```

然后，另起一个终端，执行`top`命令，用来观察`PHP`进程的内存使用情况：

```shell
35247 root      20   0   27.8m  14.7m   0.0   0.7   0:00.02 S  `- php -S 127.0.0.1:80 -t ./
```

可以看到，在启动服务器的时候，`PHP`占用了`27.8M`的内存。

我们请求一次我们的服务器：

```shell
~/codeDir/cppCode/study # curl 127.0.0.1/memory.php
```

然后查看`PHP`占的内存：

```shell
13713 root      20   0   29.8m  14.9m   0.0   0.7   0:00.01 S  `- php -S 127.0.0.1:80 -t ./
```

我们发现`PHP`多占了`2M`的内存。

我们再请求一次：

```shell
~/codeDir/cppCode/study # curl 127.0.0.1/memory.php
```

然后再次查看`PHP`占的内存：

```shell
13713 root      20   0   31.8m  15.0m   0.0   0.8   0:00.03 S  `- php -S 127.0.0.1:80 -t ./
```

我们发现`PHP`又多占了`2M`的内存。

所以说，如果我们在一次请求的生命周期通过`malloc`分配了内存，但是没有释放，那么就会造成`PHP`**整个**生命周期的内存泄漏。

我们修改扩展函数：

```c
PHP_FUNCTION(confirm_memory_compiled)
{
  void *foo = emalloc(2 * 1024 * 1024);
}
```

然后，重新编译、安装扩展：

```shell
~/codeDir/cCode/php-7.1.0/ext/memory # make clean ; make ; make install
```

重新启动服务器：

```shell
~/codeDir/cCode/php-7.1.0/ext/memory # php -S 127.0.0.1:80 -t ./
PHP 7.3.5 Development Server started at Mon Sep 23 14:27:22 2019
Listening on http://127.0.0.1:80
Document root is /root/codeDir/cCode/php-7.1.0/ext/memory
Press Ctrl-C to quit.

```

此时，`PHP`进程占用的内存：

```shell
14470 root      20   0   27.8m  14.6m   0.0   0.7   0:00.01 S  `- php -S 127.0.0.1:80 -t ./
```

然后，我们请求一次服务器：

```shell
~/codeDir/cppCode/study # curl 127.0.0.1/memory.php
```

查看`PHP`内存占用情况：

```shell
14470 root      20   0   27.8m  15.2m   0.0   0.8   0:00.02 S  `- php -S 127.0.0.1:80 -t ./
```

发现，没有增长。

再次请求服务器：

```shell
~/codeDir/cppCode/study # curl 127.0.0.1/memory.php
```

再次查看`PHP`内存占用情况：

```shell
14470 root      20   0   27.8m  15.2m   0.7   0.8   0:00.03 S  `- php -S 127.0.0.1:80 -t ./
```

发现还是没有增长。

所以说，如果我们在一次请求的生命周期中通过`emalloc`分配了内存，但是没有释放，那么在`PHP`**整个**生命周期是不会造成内存泄漏的。因为在请求结束的时候，`PHP`会自动帮我们释放掉这些内存。但是，在一次请求中，如果一直不自己释放内存，那么这次请求很可能会内存不够，导致`PHP`进程挂掉。

以上对`malloc`和`emalloc`的分析适用于`FPM`模式。但是对于`Swoole`这类扩展，接管了`PHP`的请求生命周期，所有对`Swoole`的请求都是在同一个请求生命周期里面，并且，这个请求生命周期一直不会结束。所以，就算我们使用了`emalloc`这类内存管理器，如果没有主动释放，也是会造成内存泄漏的，因为此时`PHP`的请求生命周期不会结束，因此`PHP`不会自己帮我们去释放这些内存。
