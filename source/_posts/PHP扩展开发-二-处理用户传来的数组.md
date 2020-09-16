---
title: PHP扩展开发(二)--处理用户传来的数组
date: 2018-12-18 17:07:30
tags:
- PHP
- PHP 扩展开发
---

很多时候，我们需要处理用户传来的数组，例如我们的tinyswoole server的set函数，它是接收一个数组的：

```php
$serv->set([
    'reactor_num' => 2,
]);
```

然后，tinyswoole server需要保存这些数组信息。

这个时候，我们就需要去获取这个数组里面的值。这里，我们可以这样写：

```c
PHP_METHOD(tinyswoole_server, set)
{
	zval *zset = NULL;
	tswServer *serv;
	HashTable *vht;
	zval *v;

	serv = TSwooleG.serv;

	ZEND_PARSE_PARAMETERS_START(1, 1)
        Z_PARAM_ARRAY(zset)
    ZEND_PARSE_PARAMETERS_END_EX(RETURN_FALSE);

    vht = Z_ARRVAL_P(zset);

	if (php_tinyswoole_array_get_value(vht, "reactor_num", v)) {
        convert_to_long(v);
        serv->reactor_num = (uint16_t)Z_LVAL_P(v);
    } else {
		serv->reactor_num = 2;
	}
}
```

其中`Z_ARRVAL_P(zset)`可以获取zval结构里面的`HashTable *`。

其中`php_tinyswoole_array_get_value`这个宏可以用来获取数组key所对应的值：

```c
#define php_tinyswoole_array_get_value(ht, str, v)     ((v = zend_hash_str_find(ht, str, sizeof(str)-1)) && !ZVAL_IS_NULL(v))
```

`convert_to_long`函数的作用是进行zval类型的转换。我们来看一看zval的数据结构：

```c
struct _zval_struct {
	zend_value        value;			/* value */
	union {
		struct {
			ZEND_ENDIAN_LOHI_4(
				zend_uchar    type,			/* active type */
				zend_uchar    type_flags,
				zend_uchar    const_flags,
				zend_uchar    reserved)	    /* call info for EX(This) */
		} v;
		uint32_t type_info;
	} u1;
	union {
		uint32_t     next;                 /* hash collision chain */
		uint32_t     cache_slot;           /* literal cache slot */
		uint32_t     lineno;               /* line number (for ast nodes) */
		uint32_t     num_args;             /* arguments number for EX(This) */
		uint32_t     fe_pos;               /* foreach position */
		uint32_t     fe_iter_idx;          /* foreach iterator index */
		uint32_t     access_flags;         /* class constant access flags */
		uint32_t     property_guard;       /* single property guard */
		uint32_t     extra;                /* not further specified */
	} u2;
};
```

这里的`zend_uchar type`会记录变量的类型。

