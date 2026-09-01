---
title: 在PHP的run-tests测试里面保持closing tag
date: 2020-08-10 10:01:12
tags:
- PHP
---

这里的测试指的是`PHP`底层的测试，例如一些扩展呀啥的测试，而不是`PHPUnit`之类的测试。

我们尽可能的保留`PHP`的`closing tag`，即`?>`。例如我们应该这么写：

```php
--FILE--
 <?php
 var_dump(new class{});
 ?>
 --EXPECTF--
 object(class@%s)#%d (0) {
 }
```

而不是：

```php
--FILE--
 <?php
 var_dump(new class{});
 --EXPECTF--
 object(class@%s)#%d (0) {
 }
```

详细的可以看`php-src`的[这个pr](https://github.com/php/php-src/pull/5958)。
