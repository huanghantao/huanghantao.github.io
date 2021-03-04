---
title: PHP内核是如何处理常量的
date: 2021-03-04 11:29:36
tags:
- PHP
- PHP内核
---

在`PHP`内核中，有两种常量，一种是内核预定义常量，一种是魔术常量。

## 内核预定义常量

内核预定义常量它的值不会被改变。

内核预定义常量注册流程如下：

```bash
graph-easy <<< "
graph { flow: down; }
[php_module_startup] -> 
[zend_startup] -> 
[zend_register_standard_constants \n for example E_ERROR] -> 
[zend_register_constant]

[php_module_startup] -> 
[register constants \n for example PHP_VERSION] ->
[zend_register_constant]
"

+-------------------------+     +----------------------------------+
|   register constants    |     |        php_module_startup        |
| for example PHP_VERSION | <-- |                                  |
+-------------------------+     +----------------------------------+
  |                               |
  |                               |
  |                               v
  |                             +----------------------------------+
  |                             |           zend_startup           |
  |                             +----------------------------------+
  |                               |
  |                               |
  |                               v
  |                             +----------------------------------+
  |                             | zend_register_standard_constants |
  |                             |       for example E_ERROR        |
  |                             +----------------------------------+
  |                               |
  |                               |
  |                               v
  |                             +----------------------------------+
  +---------------------------> |      zend_register_constant      |
                                +----------------------------------+
```

核心函数是`zend_register_constant`：

```cpp
ZEND_API zend_result zend_register_constant(zend_constant *c)
{
  // 省略其他代码

  /* Check if the user is trying to define any special constant */
  if (zend_string_equals_literal(name, "__COMPILER_HALT_OFFSET__")
    || (!persistent && zend_get_special_const(ZSTR_VAL(name), ZSTR_LEN(name)))
    || zend_hash_add_constant(EG(zend_constants), name, c) == NULL
  ) {
    zend_error(E_WARNING, "Constant %s already defined", ZSTR_VAL(name));
    zend_string_release(c->name);
    if (!persistent) {
      zval_ptr_dtor_nogc(&c->value);
    }
    ret = FAILURE;
  }
  if (lowercase_name) {
    zend_string_release(lowercase_name);
  }
  return ret;
}
```

我们发现，`zend_hash_add_constant`把内核预定义常量存在了`EG(zend_constants)`这个哈希表里面。

## 魔术常量

魔术常量它的值会随着代码的位置而改变，例如`__FILE__`：

```bison
%token <ident> T_FILE            "'__FILE__'"

constant:
    name		{ $$ = zend_ast_create(ZEND_AST_CONST, $1); }
  |	T_LINE 		{ $$ = zend_ast_create_ex(ZEND_AST_MAGIC_CONST, T_LINE); }
  |	T_FILE 		{ $$ = zend_ast_create_ex(ZEND_AST_MAGIC_CONST, T_FILE); }
  |	T_DIR   	{ $$ = zend_ast_create_ex(ZEND_AST_MAGIC_CONST, T_DIR); }
  |	T_TRAIT_C	{ $$ = zend_ast_create_ex(ZEND_AST_MAGIC_CONST, T_TRAIT_C); }
  |	T_METHOD_C	{ $$ = zend_ast_create_ex(ZEND_AST_MAGIC_CONST, T_METHOD_C); }
  |	T_FUNC_C	{ $$ = zend_ast_create_ex(ZEND_AST_MAGIC_CONST, T_FUNC_C); }
  |	T_NS_C		{ $$ = zend_ast_create_ex(ZEND_AST_MAGIC_CONST, T_NS_C); }
  |	T_CLASS_C	{ $$ = zend_ast_create_ex(ZEND_AST_MAGIC_CONST, T_CLASS_C); }
;
```

我们发现，在词法分析的阶段，把`__FILE__`标注为了`T_FILE`这个`token`。

```cpp
static zend_bool zend_try_ct_eval_magic_const(zval *zv, zend_ast *ast) /* {{{ */
{
  zend_op_array *op_array = CG(active_op_array);
  zend_class_entry *ce = CG(active_class_entry);

  switch (ast->attr) {
    case T_LINE:
      ZVAL_LONG(zv, ast->lineno);
      break;
    case T_FILE:
      ZVAL_STR_COPY(zv, CG(compiled_filename));
      break;
  // 省略其他代码
}
```

然后，在语法分析阶段，直接把`__FILE__`替换成了当前正在编译的文件路径。

## 禁止常量替换

对于内核预定义常量，我们可以给`CG(compiler_options)`添加`ZEND_COMPILE_NO_CONSTANT_SUBSTITUTION`和`ZEND_COMPILE_NO_PERSISTENT_CONSTANT_SUBSTITUTION`来禁止常量替换：

```cpp
static zend_bool can_ct_eval_const(zend_constant *c) {
  if (ZEND_CONSTANT_FLAGS(c) & CONST_DEPRECATED) {
    return 0;
  }
  if ((ZEND_CONSTANT_FLAGS(c) & CONST_PERSISTENT)
      && !(CG(compiler_options) & ZEND_COMPILE_NO_PERSISTENT_CONSTANT_SUBSTITUTION)
      && !((ZEND_CONSTANT_FLAGS(c) & CONST_NO_FILE_CACHE)
        && (CG(compiler_options) & ZEND_COMPILE_WITH_FILE_CACHE))) {
    return 1;
  }
  if (Z_TYPE(c->value) < IS_OBJECT
      && !(CG(compiler_options) & ZEND_COMPILE_NO_CONSTANT_SUBSTITUTION)) {
    return 1;
  }
  return 0;
}
```

但是，对于魔术常量，我们是没有办法禁止的。

那么，什么场景下需要禁止编译期间的常量替换呢？比如我们在机器`1`上面，通过`PHP7.3`持久化了`op_array`，然后我们需要在机器`2`上面通过`PHP7.4`来跑，这时候就不能够在编译期间进行常量替换。否则当我们的代码依赖于`PHP`版本的时候，就会出现问题，例如：

```php
<?php

assert(PHP_VERSION == 7.3);
```

在机器`1`上通过`PHP7.3`持久化`op_array`，在机器`2`通过`PHP7.4`执行这个脚本，就会断言出错。
