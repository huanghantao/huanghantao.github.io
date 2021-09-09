---
title: PHP内核之zend_try_ct_eval_const
date: 2021-09-09 10:22:16
tags:
- PHP内核
---

我们来看看这么一段代码：

```php
<?php // other.php
namespace foo {
var_dump(true);
}
```

执行结果如下：

```bash
php other.php
bool(true)
```

还算是符合我们的预期对吧。

我们再来看看这段代码：

```php
<?php // test.php
define('foo\true', 'test');
require_once __DIR__ . '/other.php';
```

执行结果如下：

```bash
php test.php
string(4) "test"
```

因为在其他文件里面定义了`foo\true`常量，导致引入`other.php`文件的时候，`true`的值就被无情的修改了。

问题是出在了`zend_try_ct_eval_const`这个函数里面：

```cpp
static bool zend_try_ct_eval_const(zval *zv, zend_string *name, bool is_fully_qualified) /* {{{ */
{
    zend_constant *c = zend_hash_find_ptr(EG(zend_constants), name);
    if (c && can_ct_eval_const(c)) {
        ZVAL_COPY_OR_DUP(zv, &c->value);
        return 1;
    }

    {
        /* Substitute true, false and null (including unqualified usage in namespaces) */
        const char *lookup_name = ZSTR_VAL(name);
        size_t lookup_len = ZSTR_LEN(name);

        if (!is_fully_qualified) {
            zend_get_unqualified_name(name, &lookup_name, &lookup_len);
        }

        if ((c = zend_get_special_const(lookup_name, lookup_len))) {
            ZVAL_COPY_VALUE(zv, &c->value);
            return 1;
        }

        return 0;
    }
}
```

可以看到，前面先调用`can_ct_eval_const`来判断常量是否能够被替换。因为我们这里定义了`foo\true`常量，所以这里就判断能够被替换。所以这里拿到的值就是`test`了。

在`PHP8`中，这被当作了`BUG`来处理，解决方法也很简单，把`zend_get_special_const`放到`can_ct_eval_const`前面即可。`special_const`的值有三个，`true`、`false`、`null`：

```cpp
s`tatic bool zend_try_ct_eval_const(zval *zv, zend_string *name, bool is_fully_qualified) /* {{{ */
{
    /* Substitute true, false and null (including unqualified usage in namespaces)
    * before looking up the possibly namespaced name. */
    const char *lookup_name = ZSTR_VAL(name);
    size_t lookup_len = ZSTR_LEN(name);

    if (!is_fully_qualified) {
        zend_get_unqualified_name(name, &lookup_name, &lookup_len);
    }

    zend_constant *c;
    if ((c = zend_get_special_const(lookup_name, lookup_len))) {
        ZVAL_COPY_VALUE(zv, &c->value);
        return 1;
    }
    c = zend_hash_find_ptr(EG(zend_constants), name);
    if (c && can_ct_eval_const(c)) {
        ZVAL_COPY_OR_DUP(zv, &c->value);
        return 1;
    }
    return 0;
}`
```
