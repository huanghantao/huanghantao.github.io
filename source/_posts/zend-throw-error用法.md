---
title: zend_throw_error用法
date: 2020-07-08 09:34:53
tags:
- PHP
---

> 本文基于PHP的commit为：b18b2c8fe587321384c9423470cf97d8040b32e2

在执行`PHP`扩展层面的代码的时候，如果遇到了错误，我们可以通过`zend_throw_error`这个函数来设置`error`异常对象，然后使用宏`RETURN_THROWS`来退出扩展函数。例如：

```cpp
digest = algo->hash(password, options);
if (!digest) {
    if (!EG(exception)) {
        zend_throw_error(NULL, "Password hashing failed for unknown reason");
    }
    RETURN_THROWS();
}
```

我们来看看`zend_throw_error`会做些什么：

```cpp
ZEND_API ZEND_COLD void zend_throw_error(zend_class_entry *exception_ce, const char *format, ...) /* {{{ */
{
    va_list va;
    char *message = NULL;

    if (!exception_ce) {
        exception_ce = zend_ce_error;
    }

    /* Marker used to disable exception generation during preloading. */
    if (EG(exception) == (void*)(uintptr_t)-1) {
        return;
    }

    va_start(va, format);
    zend_vspprintf(&message, 0, format, va);

    //TODO: we can't convert compile-time errors to exceptions yet???
    if (EG(current_execute_data) && !CG(in_compilation)) {
        zend_throw_exception(exception_ce, message, 0);
    } else {
        zend_error(E_ERROR, "%s", message);
    }

    efree(message);
    va_end(va);
}
```

首先是：

```cpp
if (!exception_ce) {
    exception_ce = zend_ce_error;
}
```

会先判断是否传递了异常类`exception_ce`，如果没有传递，那么使用`PHP`默认的`zend_ce_error`异常类。

然后，这里核心的地方是：

```cpp
if (EG(current_execute_data) && !CG(in_compilation)) {
    zend_throw_exception(exception_ce, message, 0);
}

ZEND_API ZEND_COLD zend_object *zend_throw_exception(zend_class_entry *exception_ce, const char *message, zend_long code) /* {{{ */
{
    zend_string *msg_str = message ? zend_string_init(message, strlen(message), 0) : NULL;
    zend_object *ex = zend_throw_exception_zstr(exception_ce, msg_str, code);
    if (msg_str) {
        zend_string_release(msg_str);
    }
    return ex;
}

static zend_object *zend_throw_exception_zstr(zend_class_entry *exception_ce, zend_string *message, zend_long code) /* {{{ */
{
    // 省略其他代码
    object_init_ex(&ex, exception_ce);

    if (message) {
        ZVAL_STR(&tmp, message);
        zend_update_property_ex(exception_ce, &ex, ZSTR_KNOWN(ZEND_STR_MESSAGE), &tmp);
    }
    if (code) {
        ZVAL_LONG(&tmp, code);
        zend_update_property_ex(exception_ce, &ex, ZSTR_KNOWN(ZEND_STR_CODE), &tmp);
    }

    zend_throw_exception_internal(&ex);
    return Z_OBJ(ex);
}

ZEND_API ZEND_COLD void zend_throw_exception_internal(zval *exception) /* {{{ */
{
    // 省略其他代码
    if (exception != NULL) {
        zend_object *previous = EG(exception);
        zend_exception_set_previous(Z_OBJ_P(exception), EG(exception));
        EG(exception) = Z_OBJ_P(exception);
        if (previous) {
            return;
        }
    }
    // 省略其他代码
}
```

这段代码做了一件事情，把`zend_ce_error`异常类实例化，然后设置它的`message`等属性，最后设置`EG(exception)`为这个实例化的对象。（所以，我们的`RETURN_THROWS`断言会成功）。
