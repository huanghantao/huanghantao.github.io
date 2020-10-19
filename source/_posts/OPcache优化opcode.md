---
title: OPcache优化opcode
date: 2020-10-20 01:29:38
tags:
- PHP
- 编译原理
- OPcache
---

这篇文章，我们会通过一些例子来介绍一下`Opcache`对于`opcode`的一些优化。

## 简单的本地优化（pass1）

这个`pass`会开启一些简单的优化，例如优化常数条件`JMP`。

对应的`opcache`配置如下：

```bash
zend_extension=opcache.so
opcache.enable=1
opcache.enable_cli=1
opcache.opt_debug_level=0x30000
opcache.optimization_level=0x1
```

我们有如下代码：

```php
<?php

function foo()
{
    if (1) {
        echo 1;
    } else {
        echo 2;
    }
}
```

执行结果如下：

```bash
foo:
     ; (lines=5, args=0, vars=0, tmps=0)
     ; (before optimizer)
     ; /Users/hantaohuang/codeDir/cCode/php-src/test.php:3-10
0000 JMPZ int(1) 0003
0001 ECHO int(1)
0002 JMP 0004
0003 ECHO int(2)
0004 RETURN null

foo:
     ; (lines=5, args=0, vars=0, tmps=0)
     ; (after optimizer)
     ; /Users/hantaohuang/codeDir/cCode/php-src/test.php:3-10
0000 NOP
0001 ECHO int(1)
0002 JMP 0004
0003 ECHO int(2)
0004 RETURN null
```

我们发现，优化前，我们需要执行`JMPZ`，并且按照条件来执行`0003`或者`0004`。但是优化后的代码，我们只需要执行`0001`和`0004`即可。我们发现，优化后的代码`0002`和`0003`实际上不会被执行，但是却生成了`opcode`，实际上是因为我们没有开启对应的优化，我们后面会有例子来讲解。

## 常数传播优化（pass1）

对应的`opcache`配置如下：

```bash
zend_extension=opcache.so
opcache.enable=1
opcache.enable_cli=1
opcache.opt_debug_level=0x30000
opcache.optimization_level=0xe0
```

我们有如下代码：

```php
<?php

function foo()
{
    $a = 1;
    echo $a + 2 + 3;
}
```

执行结果如下：

```bash
foo:
     ; (lines=5, args=0, vars=1, tmps=3)
     ; (before optimizer)
     ; /Users/hantaohuang/codeDir/cCode/php-src/test.php:3-7
     ; return  [] RANGE[0..0]
0000 ASSIGN CV0($a) int(1)
0001 T2 = ADD CV0($a) int(2)
0002 T3 = ADD T2 int(3)
0003 ECHO T3
0004 RETURN null

foo:
     ; (lines=3, args=0, vars=1, tmps=3)
     ; (after optimizer)
     ; /Users/hantaohuang/codeDir/cCode/php-src/test.php:3-7
0000 CV0($a) = QM_ASSIGN int(1)
0001 ECHO string("6")
0002 RETURN null
```

我们发现，因为`a`是常量`1`，所以在优化`opcode`的时候，会直接用`1`替换掉`a`。

## 死代码消除

对应的`opcache`配置如下：

```bash
zend_extension=opcache.so
opcache.enable=1
opcache.enable_cli=1
opcache.opt_debug_level=0x30000
opcache.optimization_level=0x2061
```

我们有如下代码：

```php
<?php

function foo()
{
    if (1) {
        echo 1;
    } else {
        echo 2;
    }
}
```

执行结果如下：

```bash
foo:
     ; (lines=5, args=0, vars=0, tmps=0)
     ; (before optimizer)
     ; /Users/hantaohuang/codeDir/cCode/php-src/test.php:3-10
     ; return  [] RANGE[0..0]
0000 JMPZ int(1) 0003
0001 ECHO int(1)
0002 JMP 0004
0003 ECHO int(2)
0004 RETURN null

foo:
     ; (lines=2, args=0, vars=0, tmps=0)
     ; (after optimizer)
     ; /Users/hantaohuang/codeDir/cCode/php-src/test.php:3-10
0000 ECHO int(1)
0001 RETURN null
```

我们发现，优化前的`0000`, `0002`, `0003`都被删除了，因为它们不会被执行。
