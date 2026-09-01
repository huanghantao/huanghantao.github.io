---
title: 如何准确的查看opline对应的handler名字
date: 2020-10-27 12:50:25
tags:
- PHP
- PHP内核
---

我们在分析`opcode`对应的`handler`的时候，往往会根据`opcode`的命名规则来推断具体的`handler`。然而，如果我们使用`PHP8`的话，我们可以利用`jit`的`debug`功能来快速的看到`opcode`对应的`handler`。我举个例子：

有如下代码：

```php
$a = [1, 2, 3];
$a[2];
```

像这个`$a[2]`对应的`handler`还是非常的长的，我们很难一口气推断出来。我们只需要配置一下`php.ini`就可以方便的拿到`handler`：

```ini
zend_extension=opcache.so
opcache.enable=1
opcache.enable_cli=1

opcache.jit=1201
opcache.jit_buffer_size=64M
opcache.jit_debug=0x01
```

执行结果如下：

```bash
JIT$/Users/hantaohuang/codeDir/cCode/php-src/test.php: ; (/Users/hantaohuang/codeDir/cCode/php-src/test.php)
    # 省略其他的汇编代码
    mov $ZEND_ASSIGN_SPEC_CV_CONST_RETVAL_UNUSED_HANDLER, %rax
    # 省略其他的汇编代码
    mov $ZEND_FETCH_DIM_R_INDEX_SPEC_CV_CONST_HANDLER, %rax
    # 省略其他的汇编代码
```

可以看到，`handler`是`ZEND_FETCH_DIM_R_INDEX_SPEC_CV_CONST_HANDLER`。
