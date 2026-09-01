---
title: PHP数组系列函数源码分析（一）--count
date: 2020-08-16 15:47:33
tags:
- PHP内核
---

> 本文基于的PHP commit为: d92229d8c78aac25925284e23aa7903dca9ed005

首先，我们来看一下`count`函数的`PHP`层：

```cpp
/* {{{ Count the number of elements in a variable (usually an array) */
PHP_FUNCTION(count)
{
    zval *array;
    zend_long mode = COUNT_NORMAL;
    zend_long cnt;

    ZEND_PARSE_PARAMETERS_START(1, 2)
        Z_PARAM_ZVAL(array)
        Z_PARAM_OPTIONAL
        Z_PARAM_LONG(mode)
    ZEND_PARSE_PARAMETERS_END();

    if (mode != COUNT_NORMAL && mode != COUNT_RECURSIVE) {
        zend_argument_value_error(2, "must be either COUNT_NORMAL or COUNT_RECURSIVE");
        RETURN_THROWS();
    }

    switch (Z_TYPE_P(array)) {
        case IS_NULL:
            /* Intentionally not converted to an exception */
            php_error_docref(NULL, E_WARNING, "Parameter must be an array or an object that implements Countable");
            RETURN_LONG(0);
            break;
        case IS_ARRAY:
            if (mode != COUNT_RECURSIVE) {
                cnt = zend_array_count(Z_ARRVAL_P(array));
            } else {
                cnt = php_count_recursive(Z_ARRVAL_P(array));
            }
            RETURN_LONG(cnt);
            break;
        case IS_OBJECT: {
            zval retval;
            /* first, we check if the handler is defined */
            if (Z_OBJ_HT_P(array)->count_elements) {
                RETVAL_LONG(1);
                if (SUCCESS == Z_OBJ_HT(*array)->count_elements(Z_OBJ_P(array), &Z_LVAL_P(return_value))) {
                    return;
                }
                if (EG(exception)) {
                    RETURN_THROWS();
                }
            }
            /* if not and the object implements Countable we call its count() method */
            if (instanceof_function(Z_OBJCE_P(array), zend_ce_countable)) {
                zend_call_method_with_0_params(Z_OBJ_P(array), NULL, NULL, "count", &retval);
                if (Z_TYPE(retval) != IS_UNDEF) {
                    RETVAL_LONG(zval_get_long(&retval));
                    zval_ptr_dtor(&retval);
                }
                return;
            }

            /* If There's no handler and it doesn't implement Countable then add a warning */
            /* Intentionally not converted to an exception */
            php_error_docref(NULL, E_WARNING, "Parameter must be an array or an object that implements Countable");
            RETURN_LONG(1);
            break;
        }
        default:
            /* Intentionally not converted to an exception */
            php_error_docref(NULL, E_WARNING, "Parameter must be an array or an object that implements Countable");
            RETURN_LONG(1);
            break;
    }
}
/* }}} */
```

可以看出，这个函数可以计算数组和对象。我们先来看一下是如何计算数组元素的个数的：

```cpp
if (mode != COUNT_RECURSIVE) {
    cnt = zend_array_count(Z_ARRVAL_P(array));
} else {
    cnt = php_count_recursive(Z_ARRVAL_P(array));
}
RETURN_LONG(cnt);
```

`COUNT_RECURSIVE`代表是否需要递归的去计算数组的元素个数（比如说，数组里面又套了一个数组）。如果不需要递归的去计算，那么调用`zend_array_count`；如果需要递归的去计算，那么调用`php_count_recursive`。

注意，`count`这个函数要被调用的话，我们得设置`count`的`mode`为`COUNT_RECURSIVE`。例如：

```php
<?php

declare(strict_types=1);

$arr = [
    'one' => 1,
    'two' => 2,
    'three' => 3,
];

$num = count($arr, COUNT_RECURSIVE);
var_dump($num);
```

否则，`count`会直接走`zend_count`对应的`opcode handler`，然后调用`zend_array_count`。

我们来看一看`zend_array_count`：

```cpp
ZEND_API uint32_t zend_array_count(HashTable *ht)
{
    uint32_t num;
    if (UNEXPECTED(HT_FLAGS(ht) & HASH_FLAG_HAS_EMPTY_IND)) {
        num = zend_array_recalc_elements(ht);
        if (UNEXPECTED(ht->nNumOfElements == num)) {
            HT_FLAGS(ht) &= ~HASH_FLAG_HAS_EMPTY_IND;
        }
    } else if (UNEXPECTED(ht == &EG(symbol_table))) {
        num = zend_array_recalc_elements(ht);
    } else {
        num = zend_hash_num_elements(ht);
    }
    return num;
}
```

可以看到，一个看似简单的`PHP`函数，有非常多的细节需要考虑。（我之前是觉得这个函数要实现起来非常简单啊，直接调用`zend_hash_num_elements`就好了）

我们先来看这部分代码：

```cpp
if (UNEXPECTED(HT_FLAGS(ht) & HASH_FLAG_HAS_EMPTY_IND)) {
    num = zend_array_recalc_elements(ht);
    if (UNEXPECTED(ht->nNumOfElements == num)) {
        HT_FLAGS(ht) &= ~HASH_FLAG_HAS_EMPTY_IND;
    }
}
```

首先是判断是否是`HASH_FLAG_HAS_EMPTY_IND`标志（`IND`应该是`indirect`的意思，而不是`index`）。这个标志表示是否存在空的间接`zval`。搜索整个`PHP`源码，我们发现，这个标志在两个地方会被设置：

```cpp
ZEND_API int ZEND_FASTCALL zend_hash_del_ind(HashTable *ht, zend_string *key);
ZEND_API int ZEND_FASTCALL zend_hash_str_del_ind(HashTable *ht, const char *str, size_t len);
```

并且，我们会发现，这两个函数似乎只使用在了符号表`EG(symbol_table)`上面。而`EG(symbol_table)`对应的`PHP`变量是`$GLOBALS`。

所以，我们可以很轻易的写一个测试例子：

```php
<?php

declare(strict_types=1);

var_dump(count($GLOBALS));

$one = 1;
$two = 2;
$three = 3;

var_dump(count($GLOBALS));

unset($GLOBALS['two']);

var_dump(count($GLOBALS));

var_dump($two);
```

执行结果如下：

```bash
int(8)
int(11)
int(10)

Warning: Undefined variable $two in /Users/hantaohuang/codeDir/cCode/php-src/test.php on line 17
NULL
```

可以看到，最初`$GLOBALS`数组里面的元素个数是`8`个。

注意，这里指的是`$GLOBALS`数组里面非`UNDEF`的元素个数是`8`个，实际上，因为最初的`$GLOBALS`有一些元素它是`UNDEF`，所以，`nNumOfElements`它的值会大于`8`，就这个脚本而言，初始的`nNumOfElements`的值是`11`，因为`PHP`在编译阶段，就会往`$GLOBALS`数组里面插入我们在全局作用域使用到的变量（即`a`、`b`、`c`），但是因为这些变量是在后面使用的，所以，最开始的时候，这`3`个数组元素是`UNDEF`的。

当我们在全局作用域里面为这`3`个数组元素赋值之后，`$GLOBALS`数组里面的元素个数变成了`11`。并且，当我们`unset`掉`$GLOBALS`数组里面的一个元素之后，数组里面的元素少了一个。

这里，我们需要注意的一个点是，我们得`unset($GLOBALS['two'])`，而不能`unset($two)`。否则是不会设置`HASH_FLAG_HAS_EMPTY_IND`标志的。（因为这个标志是在`UNSET_DIM`这个`opcode`里面设置的）

然后就是`zend_array_recalc_elements`这个函数了：

```cpp
static uint32_t zend_array_recalc_elements(HashTable *ht)
{
    zval *val;
    uint32_t num = ht->nNumOfElements;

    ZEND_HASH_FOREACH_VAL(ht, val) {
        if (Z_TYPE_P(val) == IS_INDIRECT) {
            if (UNEXPECTED(Z_TYPE_P(Z_INDIRECT_P(val)) == IS_UNDEF)) {
                num--;
            }
        }
    } ZEND_HASH_FOREACH_END();
    return num;
}
```

顾名思义，这个函数就是用来重新计算数组里面元素的个数的。那上面的那个例子来说，`unset($GLOBALS['two'])`是不会减少数组的`nNumOfElements`的值的。所以，我们需要这么一个函数来计算真正的元素个数。

我们接着来看后面的代码：

```cpp
else if (UNEXPECTED(ht == &EG(symbol_table))) {
    num = zend_array_recalc_elements(ht);
}
```

我们也可以很轻易的写出对应的测试代码：

```php
<?php

declare(strict_types=1);

$one = 1;
$two = 2;
$three = 3;

unset($two);

var_dump(count($GLOBALS));

var_dump($two);
```

输出如下：

```bash
int(10)

Warning: Undefined variable $two in /Users/hantaohuang/codeDir/cCode/php-src/test.php on line 13
NULL
```

这个代码和上面的代码的区别是，这里我们是直接`unset($two)`。那么，此时就不会执行`UNSET_DIM handler`了，因此也不会设置数组的`HASH_FLAG_HAS_EMPTY_IND`标志。但是，`$GLOBALS['two']`它依然是`UNDEF`的，因为`$GLOBALS['two']`它是变量`$two`的一个间接`zval`。所以，在`unset`之后，`$GLOBALS`的元素个数也是`10`。

我们接着来看后面的代码：

```cpp
else {
    num = zend_hash_num_elements(ht);
}
```

这段代码就简单了，直接是取数组的`nNumOfElements`值。我们可以非常轻易的写出测试代码：

```php
<?php

declare(strict_types=1);

$arr = [
    'one' => 1,
    'two' => 2,
    'three' => 3,
];

var_dump(count($arr));

unset($arr['two']);

var_dump(count($arr));
```

我们稍微解释一下。

当最开始定义数组的时候，数组的`nNumUsed`和`nNumOfElements`都是`3`。`unset($arr['two'])`之后，`nNumUsed`和`nNumOfElements`分别是`3`和`2`。所以，`count($arr)`得到的元素个数是`2`。

> 可以看出，一个简单的`count`函数，实际上还是有非常多的细节需要考虑的。而这一切的一切，都来自于`$GLOBALS`这个变量。顺便一提的是，最近`PHP`内核的诸多`bug`都是由`$GLOBALS`这个变量引起的。
