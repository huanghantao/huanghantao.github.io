---
title: 《PHP扩展开发》--return_value的使用
date: 2019-05-25 15:31:40
tags:
- PHP
- PHP扩展
---

在PHP扩展中编写函数的时候，如果我们要从PHP扩展返回值给PHP脚本，可以使用`return_value`来做到。例如：

```c
PHP_FUNCTION(hello_world) {
    ZVAL_STRINGL(return_value, "hello world!", strlen("hello world!"));
};
```

```c
#define ZVAL_STRINGL(z, s, l) do {				\
		ZVAL_NEW_STR(z, zend_string_init(s, l, 0));		\
	} while (0)

static zend_always_inline zend_string *zend_string_init(const char *str, size_t len, int persistent)
{
	zend_string *ret = zend_string_alloc(len, persistent);

	memcpy(ZSTR_VAL(ret), str, len);
	ZSTR_VAL(ret)[len] = '\0';
	return ret;
}

#define ZSTR_VAL(zstr)  (zstr)->val

#define ZVAL_NEW_STR(z, s) do {					\
		zval *__z = (z);						\
		zend_string *__s = (s);					\
		Z_STR_P(__z) = __s;						\
		Z_TYPE_INFO_P(__z) = IS_STRING_EX;		\
	} while (0)
```

其中，`zend_string_init`的作用就是从堆中分配一块内存给`ret`。然后把字符串`hello world!`以及字符串的长度保存在`ret`里面，然后，返回包含`hello world!`字符串的地址。

`ZVAL_NEW_STR(z, s)`的作用是让`z`指向`s`，也就是说，此时的`return_value`里面的`value.str`指向了`zend_string_init`返回的包含了`hello world!`字符串的`zend_string`。

所以说，通过`return_value`可以找到字符串`hello world!`。

一个问题是这个`return_value`是哪里来的？

我们展开`PHP_FUNCTION`可以看到：

```c
#define PHP_FUNCTION			ZEND_FUNCTION
#define ZEND_FUNCTION(name)				ZEND_NAMED_FUNCTION(ZEND_FN(name))
#define ZEND_FN(name) zif_##name
#define ZEND_NAMED_FUNCTION(name)		void name(INTERNAL_FUNCTION_PARAMETERS)
#define INTERNAL_FUNCTION_PARAMETERS zend_execute_data *execute_data, zval *return_value
```

因此，我们对`PHP_FUNCTION(hello_world)`进行展开的话，可以得到：

```c
zif_hello_world(zend_execute_data *execute_data, zval *return_value)
```

所以，我们可以在使用了`PHP_FUNCTION`宏里面使用`return_value`。我们往`return_value`里面填充值的话，`PHP`脚本就可以得到返回值了。

因此，我们执行如下脚本：

```php
<?php

$ret = hello_world();
var_dump($ret);
```

就可以打印出：

```shell
~/codeDir/phpCode/test # php study.php 
string(12) "hello world!"
```









