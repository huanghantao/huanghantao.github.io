---
title: PHP扩展如何获取私有属性的名字
date: 2021-01-07 11:39:39
tags:
- PHP
---

> 本篇文章基于PHP7.4.10

我们的测试脚本如下：

```php
<?php

class Foo
{
    public $a;

    private $b;

    public function __construct()
    {
        $this->a = 1;
        $this->b = 1;
    }
}

$foo = new Foo;

printAllAttributeKeys($foo);
```

其中`printAllAttributeKeys`是我们要编写的一个扩展函数，用来打印对象所有的属性名字。

因为`PHP`属性是存在一个哈希表里面的，所以我们可以进行如下操作：

```cpp
static PHP_FUNCTION(printAllAttributeKeys) {
    zend_array *properties;
    zval *zobj;
    zend_ulong num;
    zend_string *key;
    zval *val;

    ZEND_PARSE_PARAMETERS_START_EX(ZEND_PARSE_PARAMS_THROW, 1, 1)
        Z_PARAM_OBJECT(zobj)
    ZEND_PARSE_PARAMETERS_END_EX(RETURN_FALSE);

#if PHP_VERSION_ID >= 70400
    properties = zend_get_properties_for(zobj, ZEND_PROP_PURPOSE_VAR_EXPORT);
#else
    if (Z_OBJ_HANDLER_P(zobj, get_properties)) {
        properties = Z_OBJPROP_P(zobj);
    }
#endif

    ZEND_HASH_FOREACH_KEY_VAL_IND(properties, num, key, val) {
        printf("%s\n", ZSTR_VAL(key));
    } ZEND_HASH_FOREACH_END();
}
```

执行结果如下：

```bash
php test.php
a

```

可以发现，只打印出了`a`。实际上，对于`b`这个属性，它在`zend_string`里的存储内容为：

```bash
\0Foo\0b
```

如果我们要打印出私有属性，我们可以作如下操作：

```cpp
const char *get_property_name(zend_string *property_name) {
    const char *class_name, *_property_name;
    size_t _property_name_len;

    zend_unmangle_property_name_ex(property_name, &class_name, &_property_name, &_property_name_len);

    return _property_name;
}

static PHP_FUNCTION(printAllAttributeKeys) {
    zend_array *properties;
    zval *zobj;
    zend_ulong num;
    zend_string *key;
    zval *val;

    ZEND_PARSE_PARAMETERS_START_EX(ZEND_PARSE_PARAMS_THROW, 1, 1)
        Z_PARAM_OBJECT(zobj)
    ZEND_PARSE_PARAMETERS_END_EX(RETURN_FALSE);

#if PHP_VERSION_ID >= 70400
    properties = zend_get_properties_for(zobj, ZEND_PROP_PURPOSE_VAR_EXPORT);
#else
    if (Z_OBJ_HANDLER_P(zobj, get_properties)) {
        properties = Z_OBJPROP_P(zobj);
    }
#endif

    ZEND_HASH_FOREACH_KEY_VAL_IND(properties, num, key, val) {
        printf("%s\n", get_property_name(key));
    } ZEND_HASH_FOREACH_END();
}
```

对属性`key`这个`zend_string`调用`zend_unmangle_property_name_ex`即可获取到私有属性的名字。

执行结果如下：

```bash
php test.php
a
b
```
