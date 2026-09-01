---
title: PHP函数编译之后的内存存储结构
date: 2021-01-30 15:03:34
tags:
- PHP
- PHP内核
---

> 本文基于 PHP8.0.1

测试脚本如下：

```php
<?php

namespace Bar;

function Foo1(string $arg1) {
}

function foo2(string $arg2) {
}

Foo1('aa');
foo2('bb');
```

流程如下：

```bash
   +----------------------------------------+   
   |                                        |   
   |         zend_compile_top_stmt          |   
   |                                        |   
   +----------------------------------------+   
                        |                       
                        |                       
                        |                       
         ast->kind == ZEND_AST_FUNC_DECL        
                        |                       
                        |                       
                        v                       
   +----------------------------------------+   
   |         zend_compile_func_decl         |   
   |                                        |   
   |         start compile function         |   
   +----------------------------------------+   
                        |                       
                        |                       
                        |                       
                        v                       
   +----------------------------------------+   
   |                                        |   
   |             init_op_array              |   
   |                                        |   
   +----------------------------------------+   
                        |                       
                        |                       
                        v                       
+----------------------------------------------+
|                                              |
|                                              |
|             zend_begin_func_decl             |
|                                              |
| 1. convert function from unqualified_name to |
|  namespace_name (it means Foo1 and function  |
|            op_array to Bar\Foo1)             |
|   2. insert bar\foo1 to CG(function_table)   |
|                                              |
|                                              |
+----------------------------------------------+
                        |                       
                        |                       
                        |                       
                        v                       
   +----------------------------------------+   
   |                                        |   
   |          zend_compile_params           |   
   |                                        |   
   +----------------------------------------+   
                        |                       
                        |                       
                        |                       
                        |                       
                        v                       
   +----------------------------------------+   
   |                                        |   
   |       zend_compile_stmt stmt_ast       |   
   |                                        |   
   +----------------------------------------+   
                        |                       
                        |                       
                        |                       
                        v                       
   +----------------------------------------+   
   |                                        |   
   |       zend_compile_stmt stmt_ast       |   
   |                                        |   
   +----------------------------------------+   
                        |                       
                        |                       
                        |                       
                        v                       
   +----------------------------------------+   
   |       zend_emit_final_return(0)        |   
   |                                        |   
   |            add return null             |   
   +----------------------------------------+   
                        |                       
                        |                       
                        |                       
                        v                       
   +----------------------------------------+   
   |                                        |   
   |                pass_two                |   
   |                                        |   
   +----------------------------------------+   
```

对应的主函数常量存储的内容如下：

```bash
0: Bar\Foo1
1: bar\foo1
2: foo1
3: aa
4: Bar\foo2
5: bar\foo2
6: foo2
7: bb
8: 1
```

说明，在常量表里面，既存了函数原来的名字，也存了函数的全小写名字。

`opline`如果要用到函数的名字，那么偏移量存的是函数原来的名字。但是，在查找函数的时候，需要用小写的名字，因为`CG(function_table)`里面存的是全小写的名字（目的是为了让PHP脚本的函数不区分大小写）。所以，如果`opline`需要通过函数名字来查找`zend_function`，那么要对`opline`引用的`zval *literal`偏移量`+1`。

那如果使用了命名参数，那么常量区还会存储参数的名字，例如：

```php
<?php

namespace Bar;

function Foo1(string $arg1) {
}

function foo2(string $arg2) {
}

Foo1(arg1: 'aa');
foo2(arg2: 'bb');
```

对应的主函数常量存储的内容如下：

```bash
0: Bar\Foo1
1: bar\foo1
2: foo1
3: aa
4: arg1
5: Bar\foo2
6: bar\foo2
7: foo2
8: bb
9: arg2
10: 1
```
