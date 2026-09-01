---
title: zend_read_property的rv参数作用
date: 2020-08-09 21:54:46
tags:
- PHP内核
---

> 本文基于的PHP版本为7.3.12

我们在开发`PHP`扩展的时候，经常会要去读取对象的属性，一般来说就是使用`zend_read_property`这个函数来完成。这个函数的原型如下：

```cpp
ZEND_API zval *zend_read_property(zend_class_entry *scope, zval *object, const char *name, size_t name_length, zend_bool silent, zval *rv);
```

这个函数的返回值很容易理解，就是这个属性对应的`value`值。那么，最后一个参数`zval *rv`有什么用呢？

这个参数是给动态属性来用的。我们知道，当访问一个`PHP`对象的动态属性的时候，是会去调用这个对象的`__get`魔术方法。而动态属性它的内存是不在`zend_object`上面的，它是通过`__get`魔术方法来得到的。而普通的属性它的内存是在`zend_object`上面的，所以，当我们去访问普通的属性的时候，可以直接返回一个`zval`。

既然访问动态属性是通过调用`__get`魔术方法来实现的，那么，类似于`zend_call_method`一样，我们需要去设置`zend_fcall_info::retval`。当函数调用结束的时候，返回值就会存放在这个`zval`中。这样，我们就可以取到动态属性的值了。

核心的代码如下：

```cpp
static void zend_std_call_getter(zend_object *zobj, zend_string *prop_name, zval *retval) /* {{{ */
{
    zend_class_entry *ce = zobj->ce;
    zend_class_entry *orig_fake_scope = EG(fake_scope);
    zend_fcall_info fci;
    zend_fcall_info_cache fcic;
    zval member;

    EG(fake_scope) = NULL;

    /* __get handler is called with one argument:
        property name

    it should return whether the call was successful or not
    */

    ZVAL_STR(&member, prop_name);

    fci.size = sizeof(fci);
    fci.object = zobj;
    fci.retval = retval;
    fci.param_count = 1;
    fci.params = &member;
    fci.no_separation = 1;
    ZVAL_UNDEF(&fci.function_name); /* Unused */

    fcic.function_handler = ce->__get;
    fcic.called_scope = ce;
    fcic.object = zobj;

    zend_call_function(&fci, &fcic);

    EG(fake_scope) = orig_fake_scope;
}
```

当访问`PHP`对象的动态属性的时候，就会去调用这个函数。而这个函数的`retval`就是我们`zend_read_property`的`rv`参数。
