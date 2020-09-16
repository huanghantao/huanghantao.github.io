---
title: PHP8通过ZPP宏获取zend_class_entry
date: 2020-07-01 21:12:03
tags:
- PHP
---

> 本文基于的`commit`为：e93d20ad7ebc1075ef1248a663935ee5ea69f1cd

昨天（2020-6-30），有一个宏被加入到了`PHP8`里面，用来快速获取一个类名字或者对象的`zend_class_entry`结构，在`PHP7`里面，如果我们要去获取类，我们必须得这么做：

```cpp
zval *arg;
zend_class_entry *ce = NULL;

zend_parse_parameters(ZEND_NUM_ARGS(), "z", &arg)

if (Z_TYPE_P(arg) == IS_OBJECT) {
    ce = Z_OBJ_P(arg)->ce;
} else if (Z_TYPE_P(arg) == IS_STRING) {
    ce = zend_lookup_class(Z_STR_P(arg));
}
RETURN_STR_COPY(ce->name);
```

现在，有了`Z_PARAM_CLASS_NAME_OR_OBJ`宏，我们可以非常方便的去实现这个操作：

```cpp
ZEND_PARSE_PARAMETERS_START(1, 1)
    Z_PARAM_CLASS_NAME_OR_OBJ(ce)
ZEND_PARSE_PARAMETERS_END();

RETURN_STR_COPY(ce->name);
```
