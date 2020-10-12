---
title: PHP编译优化--常量折叠
date: 2020-10-12 00:14:02
tags:
- 编译原理
- PHP
---

本文基于的`PHP8 commit`为：`14806e0824ecd598df74cac855868422e44aea53`

有如下代码：

```php
<?php

echo 1 + 2 + 3;
```

对应的`opcode`为：

```bash
L3    #0     ECHO                    6
L4    #1     RETURN<-1>              1
```

我们可以分析出，这个`echo`表达式对应的`AST`大概如下：

```bash
            ZEND_ECHO
        +
    +       3
1       2
```

所以，可以看到，`PHP`代码生成的时候，很轻松的进行优化了：

```bash
        ZEND_ECHO
    +
3       3
```

最后就会优化为：

```bash
    ZEND_ECHO
6
```

所以生成的`opcode`只有一条。

那么，我们再来看一个`PHP`目前没有优化的例子：

```php
<?php

$x = 1;

echo $x + 2 + 3;
```

对应的`opcode`如下：

```bash
L3    #0     ASSIGN                  $x                   1
L5    #1     ADD                     $x                   2                    ~1
L5    #2     ADD                     ~1                   3                    ~2
L5    #3     ECHO                    ~2
L6    #4     RETURN<-1>              1
```

可以看到，这里没有进行优化，理论上来说，对常量进行折叠的话，可以减少一条`opcode`。那么为什么`PHP`内核它没有对这种情况进行优化呢？我们先来看一看这条语句对应的`AST`：

```bash
            ZEND_ECHO
        +
    +       3
x       2
```

可以发现，如果对`AST`进行深度遍历的话，是先进行`x + 2`，而`x`是一个变量，折叠不了，所以就没有优化到了，具体的代码是这样的（在函数`zend_compile_binary_op`里面）：

```cpp
if (left_node.op_type == IS_CONST && right_node.op_type == IS_CONST) {
    if (zend_try_ct_eval_binary_op(&result->u.constant, opcode,
            &left_node.u.constant, &right_node.u.constant)
    ) {
        result->op_type = IS_CONST;
        zval_ptr_dtor(&left_node.u.constant);
        zval_ptr_dtor(&right_node.u.constant);
        return;
    }
}
```

我们发现，折叠的情况只有是当左右节点都为`IS_CONST`类型的时候，才会生效。

那么，面对这种情况，理论上我们可以怎么解决呢？我们可以对这个`AST`进行旋转，得到：

```bash
        ZEND_ECHO
    +               +
x               2       3
```

然后，我们就可以优化为：

```bash
        ZEND_ECHO
    +               5
x
```

既然，`PHP`没有做这方面的优化工作，那么，我们写代码的时候，就可以稍微注意一下了。常量尽可能的往左边靠拢，例如`1 + 2 + x`这样。

后续我们的`yaphp`会使用`LLVM`来对这方面进行优化。
