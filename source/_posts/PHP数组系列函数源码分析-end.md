---
title: PHP数组系列函数源码分析--end
date: 2020-08-16 20:15:42
tags:
- PHP内核
---

> 本文基于PHP的commit: d92229d8c78aac25925284e23aa7903dca9ed005

如果我们要获取数组的最后一个元素，我们很可能会这么写：

```php
<?php

declare(strict_types=1);

$arr = [
    'one' => 1,
    'two' => 2,
    'three' => 3,
];

var_dump(end($arr));
```

输出结果如下：

```bash
int(3)
```

我们来看一下`end`对应的`PHP`层代码：

```cpp
PHP_FUNCTION(end)
{
    HashTable *array;
    zval *entry;

    ZEND_PARSE_PARAMETERS_START(1, 1)
        Z_PARAM_ARRAY_OR_OBJECT_HT_EX(array, 0, 1)
    ZEND_PARSE_PARAMETERS_END();

    zend_hash_internal_pointer_end(array);

    if (USED_RET()) {
        if ((entry = zend_hash_get_current_data(array)) == NULL) {
            RETURN_FALSE;
        }

        if (Z_TYPE_P(entry) == IS_INDIRECT) {
            entry = Z_INDIRECT_P(entry);
        }

        ZVAL_COPY_DEREF(return_value, entry);
    }
}
```

可以看到，核心代码就是`zend_hash_internal_pointer_end`，它负责找到数组的最后一个元素：

```cpp
#define zend_hash_internal_pointer_end(ht) \
    zend_hash_internal_pointer_end_ex(ht, &(ht)->nInternalPointer)

/* This function will be extremely optimized by remembering
* the end of the list
*/
ZEND_API void ZEND_FASTCALL zend_hash_internal_pointer_end_ex(HashTable *ht, HashPosition *pos)
{
    uint32_t idx;

    IS_CONSISTENT(ht);
    HT_ASSERT(ht, &ht->nInternalPointer != pos || GC_REFCOUNT(ht) == 1);

    idx = ht->nNumUsed;
    while (idx > 0) {
        idx--;
        if (Z_TYPE(ht->arData[idx].val) != IS_UNDEF) {
            *pos = idx;
            return;
        }
    }
    *pos = ht->nNumUsed;
}
```

通过这个函数的注释，我们可以明白。如果我们能够大概记住数组的末尾的元素，那么，这个函数的性能是非常高的。

这个代码也是很简单的，首先，通过：

```cpp
idx = ht->nNumUsed;
idx--;
```

来找到最后一个`bucket`的位置。然后，判断`bucket`里面的变量是否是`IS_UNDEF`。如果不是，那么就找到了数组的最后一个元素；否则，一直往前找。

所以，如果这个数组的末尾都是`IS_UNDEF`，那么这个函数的性能会非常的差劲。极端情况下，只有数组的第一个元素不是`IS_UNDEF`，其他的都是`IS_UNDEF`，那么这个函数的时间复杂度就是`O(n)`了。

这里，我们有一个需要非常注意的点，这个`end`函数会去设置`nInternalPointer`指针。如果我们调用`end`函数后，紧接着调用`current`函数，那么我们就会得到数组的最后一个元素：

```php
<?php

declare(strict_types=1);

$arr = [
    'one' => 1,
    'two' => 2,
    'three' => 3,
];

var_dump(current($arr));
var_dump(end($arr));
var_dump(current($arr));
```

输出结果如下：

```bash
int(1)
int(3)
int(3)
```

但是，并不是说`nInternalPointer`就代表最后一个元素的位置。`nInternalPointer`表示数组里面有这么一个指针，它指向了`PHP`数组里面的一个元素，仅此而已。
