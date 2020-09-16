---
title: PHP内存分配超过限制的退出流程
date: 2020-06-28 12:24:44
tags:
- PHP
---

我们知道，在`PHP`的世界里，如果我们要申请一块内存 ，但是没有申请到，那么就会导致`fatal`级别的错误。我们来测试下：

```php
<?php

ini_set('memory_limit','1M');

$str = str_repeat('a', 2 * 1024 * 1024);
```

执行结果如下：

```bash
[root@a896c4eb1fc4 library]# php oom.php
PHP Fatal error:  Allowed memory size of 2097152 bytes exhausted at /root/php-src/Zend/zend_string.h:144 (tried to allocate 2097184 bytes) in /root/codeDir/phpCode/library/oom.php on line 5

Fatal error: Allowed memory size of 2097152 bytes exhausted at /root/php-src/Zend/zend_string.h:144 (tried to allocate 2097184 bytes) in /root/codeDir/phpCode/library/oom.php on line 5
[root@a896c4eb1fc4 library]#
```

可以看到，这里抛出了`fatal`的错误。

可能有小伙伴会觉得很正常，既然内存用完了，就应该报错，然后终止程序的执行才对。况且，大部分的`PHP`程序都是`FPM`的模型，就算这个`PHP`进程挂了，也不会影响后续的请求。但是，这对于基于`CLI`的常驻内存的`PHP`程序就是致命的了，一旦超过了内存限制，就会导致整个服务挂了，哪怕这次内存申请是很不重要的，也会导致整个`VM`的崩溃。比如说，我想要分配一个内存，但是不确定要分配多少，所以我只能够去尝试着分配。比如说第一次尝试分配`2M`，第二次尝试分配`1M`。然而，第一次申请的内存太多了，达到了限制，直接就是`fatal`了，就没有后续尝试分配`1M`的事情了。所以，这就会导致，我们不敢百分之百的去使用内存资源，因为一旦我们不小心申请的内存超过了限制，程序就会直接奔溃，没有任何拯救的余地。

我们来打个类似的比方，我们写一个`Web`服务器，我们要去`accept`连接，但是，这个时候返回了一个`Too many open files`的错误码。这个时候，我们是直接让程序`exit`吗？还是`sleep`一会儿，然后再去`accept`连接？显然是后者。所以，我们写长生命周期的脚本，需要把内存限制往大了开。

我们现在来看一下`PHP`内核是如何处理内存达到限制的情况的。重点在函数`zend_mm_safe_error`里面：

```cpp
static ZEND_COLD ZEND_NORETURN void zend_mm_safe_error(zend_mm_heap *heap,
	const char *format,
	size_t limit,
#if ZEND_DEBUG
	const char *filename,
	uint32_t lineno,
#endif
	size_t size)
{

	heap->overflow = 1;
	zend_try {
		zend_error_noreturn(E_ERROR,
			format,
			limit,
#if ZEND_DEBUG
			filename,
			lineno,
#endif
			size);
	} zend_catch {
	}  zend_end_try();
	heap->overflow = 0;
	zend_bailout();
	exit(1);
}
```

在调用`str_repeat`时候，如果内存不够，`zend_mm_safe_error`就会被调用。我们发现，在这个函数里面，调用了`zend_bailout()`，这就会导致`PHP`的执行流回到`php_execute_script`这个函数的`zend_try`里面，然后，`PHP`脚本退出执行。

所以，我们发现，只要有一次申请的`PHP`内存累积到了我们设置的限制，就没有任何拯救的余地了，进程直接退出了。
