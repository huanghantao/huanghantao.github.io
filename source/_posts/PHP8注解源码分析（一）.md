---
title: PHP8注解源码分析（一）
date: 2020-06-11 16:54:57
tags:
- PHP
---

> 本篇文章基于的PHP commit为：217f6e16d625abd9ce2ae1ae92421f77945649df

我们的测试脚本如下：

```php
<?php

<<Bean(1, 2)>>
class Foo
{
}
```

首先，我们需要关注的第一个函数是`zend_register_attribute_ce`：

```cpp
void zend_register_attribute_ce(void)
{
	zend_hash_init(&internal_validators, 8, NULL, NULL, 1);

	zend_class_entry ce;

	INIT_CLASS_ENTRY(ce, "PhpAttribute", NULL);
	zend_ce_php_attribute = zend_register_internal_class(&ce);
	zend_ce_php_attribute->ce_flags |= ZEND_ACC_FINAL;

	zend_compiler_attribute_register(zend_ce_php_attribute, zend_attribute_validate_phpattribute);
}
```

这个函数会在`PHP`模块初始化的阶段被调用，用来注册`PHP`内部类`PhpAttribute`。这个类非常的有用，类似于[民间版注解](https://github.com/doctrine/annotations)的`@Annotation`，可以用来定义一个注解类。

`OK`，我们来看看`zend_register_attribute_ce`这个函数，其中：

```cpp
zend_hash_init(&internal_validators, 8, NULL, NULL, 1);
```

用来初始化注解的验证器，比如说，限制这个注解只能够用在类上面。目前，验证器是空的。

```cpp
INIT_CLASS_ENTRY(ce, "PhpAttribute", NULL);
zend_ce_php_attribute = zend_register_internal_class(&ce);
zend_ce_php_attribute->ce_flags |= ZEND_ACC_FINAL;
```

用来定义一个`PhpAttribute`类，并且这个类是`final`的。

```cpp
zend_compiler_attribute_register(zend_ce_php_attribute, zend_attribute_validate_phpattribute);

ZEND_API void zend_compiler_attribute_register(zend_class_entry *ce, zend_attributes_internal_validator validator)
{
	if (ce->type != ZEND_INTERNAL_CLASS) {
		zend_error_noreturn(E_ERROR, "Only internal classes can be registered as compiler attribute");
	}

	zend_string *lcname = zend_string_tolower_ex(ce->name, 1);

	zend_hash_update_ptr(&internal_validators, lcname, validator);
	zend_string_release(lcname);

	zend_add_class_attribute(ce, zend_ce_php_attribute->name, 0);
}
```

可以看出，`zend_compiler_attribute_register`主要做两件事情，第一件事情是把`zend_attribute_validate_phpattribute`这个验证器添加到`internal_validators`这个哈希表里面。

第二件事情是把`PhpAttribute`注解的名字添加到`zend_ce_php_attribute->attributes`里面。

这样，`PhpAttribute`这个类算是创建完了。

接下来，就开始编译我们的这个`PHP`脚本了。在编译的过程中，一个很重要的函数是`zend_compile_attributes`：

```cpp
static void zend_compile_attributes(HashTable **attributes, zend_ast *ast, uint32_t offset, int target) /* {{{ */
{
	zend_ast_list *list = zend_ast_get_list(ast);
	uint32_t i, j;

	ZEND_ASSERT(ast->kind == ZEND_AST_ATTRIBUTE_LIST);

	for (i = 0; i < list->children; i++) {
		zend_ast *el = list->child[i];
		zend_string *name = zend_resolve_class_name_ast(el->child[0]);
		zend_ast_list *args = el->child[1] ? zend_ast_get_list(el->child[1]) : NULL;

		zend_attribute *attr = zend_add_attribute(attributes, 0, offset, name, args ? args->children : 0);
		zend_string_release(name);

		// Populate arguments
		if (args) {
			ZEND_ASSERT(args->kind == ZEND_AST_ARG_LIST);

			for (j = 0; j < args->children; j++) {
				zend_const_expr_to_zval(&attr->argv[j], args->child[j]);
			}
		}

		// Validate internal attribute
		zend_attributes_internal_validator validator = zend_attribute_get_validator(attr->lcname);

		if (validator != NULL) {
			validator(attr, target);
		}
	}
}
```

编译的这个`ast`节点它是`ZEND_AST_ATTRIBUTE_LIST`类型的`list`节点。可以看出，这实际上就开始编译我们的`Bean`注解了。

首先，这个`list`节点的第一个子节点`el->child[0]`是`ZEND_AST_ZVAL`类型的节点，里面保存了一个字符串，而这个字符串就是我们注解的名字`Bean`。并且，我们发现，这个字符串是通过函数`zend_resolve_class_name_ast`来解析的，说明这个注解的名字必须符合`PHP`类名的命名规范。

然后，这个`list`节点的第二个节点`el->child[1]`是`ZEND_AST_ARG_LIST`类型的`list`节点。我们可以很容易的知道，实际上就对应了`Bean(1, 2)`中的`1`和`2`，这两个都是`ZEND_AST_ZVAL`类型的节点。

在获取到`args`之后，调用了以下函数：

```cpp
zend_attribute *attr = zend_add_attribute(attributes, 0, offset, name, args ? args->children : 0);
```

(其中，`attributes`是我们定义的`Foo`类的`attributes`哈希表)

我们来看看这个`zend_add_attribute`函数会做些什么事情：

```cpp
ZEND_API zend_attribute *zend_add_attribute(HashTable **attributes, zend_bool persistent, uint32_t offset, zend_string *name, uint32_t argc)
{
	if (*attributes == NULL) {
		*attributes = pemalloc(sizeof(HashTable), persistent);
		zend_hash_init(*attributes, 8, NULL, persistent ? attr_pfree : attr_free, persistent);
	}

	zend_attribute *attr = pemalloc(ZEND_ATTRIBUTE_SIZE(argc), persistent);

	if (persistent == ((GC_FLAGS(name) & IS_STR_PERSISTENT) != 0)) {
		attr->name = zend_string_copy(name);
	} else {
		attr->name = zend_string_dup(name, persistent);
	}

	attr->lcname = zend_string_tolower_ex(attr->name, persistent);
	attr->offset = offset;
	attr->argc = argc;

	/* Initialize arguments to avoid partial initialization in case of fatal errors. */
	for (uint32_t i = 0; i < argc; i++) {
		ZVAL_UNDEF(&attr->argv[i]);
	}

	zend_hash_next_index_insert_ptr(*attributes, attr);

	return attr;
}
```

其中：

```cpp
if (*attributes == NULL) {
	*attributes = pemalloc(sizeof(HashTable), persistent);
	zend_hash_init(*attributes, 8, NULL, persistent ? attr_pfree : attr_free, persistent);
}
```

用来判断`Foo`类的`attributes`哈希表是否分配了内存，没有分配的话，就分配一下。

```cpp
zend_attribute *attr = pemalloc(ZEND_ATTRIBUTE_SIZE(argc), persistent);
```

用来分配一个`zend_attribute`的内存。我们看一下`ZEND_ATTRIBUTE_SIZE`这个宏：

```cpp
#define ZEND_ATTRIBUTE_SIZE(argc) (sizeof(zend_attribute) + sizeof(zval) * (argc) - sizeof(zval))
```

首先是求`zend_attribute`结构体的大小，然后再为分配`argc - 1`个`zval`的内存空间。为什么还要分配`argc - 1`个`zval`的内存空间呢？我们来看看`zend_attribute`这个结构体：

```cpp
typedef struct _zend_attribute {
	zend_string *name;
	zend_string *lcname;
	/* Parameter offsets start at 1, everything else uses 0. */
	uint32_t offset;
	uint32_t argc;
	zval argv[1];
} zend_attribute;
```

我们发现，这个结构体最后一个成员是`zval argv[1]`，所以，我们发现，这个实际上是一个柔性数组。所以，我们需要为`argv`额外分配内存。而`argc`的大小就是`2`。因为我们需要保存`1`和`2`两个值。

分配完了`zend_attribute`内存之后，就开始使用`zend_attribute`了。

```cpp
attr->name = zend_string_copy(name);
```

保存注解原始的名字，也就是`Bean`。

```cpp
attr->lcname = zend_string_tolower_ex(attr->name, persistent);
```

保存注解的小写名字，也就是`bean`。

```cpp
attr->argc = argc;
```

保存注解参数的个数，这里是`2`。

```cpp
zend_hash_next_index_insert_ptr(*attributes, attr);
```

最后，把`zend_attribute`插入`Foo`类的`attributes`哈希表。通过这个操作，我们可以知道，同一个类的注解可以有多个，因为底层使用数组保存的注解信息。

我们继续回到`zend_compile_attributes`函数里面：

```cpp
for (j = 0; j < args->children; j++) {
	zend_const_expr_to_zval(&attr->argv[j], args->child[j]);
}
```

计算注解的两个参数的值，然后保存到对应的`argv`里面。

到此位置，我们已经编译完成了注解的语法树。接着，就是验证这个注解是否合法了：

```cpp
// Validate internal attribute
zend_attributes_internal_validator validator = zend_attribute_get_validator(attr->lcname);

if (validator != NULL) {
	validator(attr, target);
}
```

所以，总结一下编译注解后的结果：

> 把注解名字和参数保存在一个zend_attribute的结构体里面，然后再把这个zend_attribute插入到对应的类结构体对象的attributes里面。这样，我们后续只要拿到了类的结构体指针，我们就可以拿到我们注解的内容，包括注解的名字和注解的参数。
