---
title: PHP8 RETURN_THROWS宏用法
date: 2020-07-07 17:38:07
tags:
- PHP
---

> 本文基于php的commit为：b18b2c8fe587321384c9423470cf97d8040b32e2

在`PHP8`之前，扩展函数解析参数的时候，如果解析失败了，那么会`return`，如下所示：

```cpp
if (zend_parse_parameters_none() == FAILURE) {
    return;
}
```

在`PHP8`后，扩展函数解析参数失败的时候，会使用`RETURN_THROWS`这个宏，例如：

```cpp
if (c() == FAILURE) {
    RETURN_THROWS();
}
```

我们来看看`RETURN_THROWS`这个宏：

```cpp
#define RETURN_THROWS()             do { ZEND_ASSERT(EG(exception)); (void) return_value; return; } while (0)

#if ZEND_DEBUG
# define ZEND_ASSERT(c) assert(c)
#else
# define ZEND_ASSERT(c) ZEND_ASSUME(c)
#endif
```

可以看出，这个宏只会去断言此时`EG(exception)`不为`NULL`。因为在`PHP8`中，大部分不被期待的错误都应该抛异常。

但是，真正会去打印异常消息的地方是在函数`zend_execute_scripts`里面：

```cpp
if (op_array) {
    zend_execute(op_array, retval);
    zend_exception_restore();
    if (UNEXPECTED(EG(exception))) {
        if (Z_TYPE(EG(user_exception_handler)) != IS_UNDEF) {
            zend_user_exception_handler();
        }
        if (EG(exception)) {
            ret = zend_exception_error(EG(exception), E_ERROR);
        }
    }
    destroy_op_array(op_array);
    efree_size(op_array, sizeof(zend_op_array));
} else if (type==ZEND_REQUIRE) {
    ret = FAILURE;
}
```

可以看到，在执行完了我们的`opcode`之后，会去判断`EG(exception)`是否为`NULL`，如果不为`NULL`，就会调用`zend_exception_error`函数，这个函数的核心是：

```cpp
zend_string *message = zval_get_string(GET_PROPERTY(&exception, ZEND_STR_MESSAGE));
zend_string *file = zval_get_string(GET_PROPERTY_SILENT(&exception, ZEND_STR_FILE));
zend_long line = zval_get_long(GET_PROPERTY_SILENT(&exception, ZEND_STR_LINE));
```

（其中，`exception`指向的是`EG(exception)`）

可以看到，这三行会去获取`EG(exception)`异常对象`zend_object`的三个属性，分别是`message`，`file`、`line`。然后组成我们的报错信息，最后通过`zend_error_va`函数（实际上调用的是`zend_error_cb`函数）打印出来。
