---
title: PHP内核对符号的处理
date: 2020-11-01 15:14:03
tags:
- PHP
- PHP内核
---

> `PHP`内核在编译`PHP`脚本的过程中，会把符号的名字转化为符号对应的存储空间的地址（注意，不是符号的地址，而是符号对应的存储空间的地址）。

我们知道，`opline`的结构如下：

```cpp
typedef union _znode_op {
    uint32_t      constant;
    uint32_t      var;
    uint32_t      num;
} znode_op;

struct _zend_op {
    const void *handler;
    znode_op op1;
    znode_op op2;
    znode_op result;
    uint32_t extended_value;
    uint32_t lineno;
    zend_uchar opcode;
    zend_uchar op1_type;
    zend_uchar op2_type;
    zend_uchar result_type;
};
```

`znode_op`这个结构，它是一个`uint32_t`类型的数字，可以用来存放和操作数地址有关的东西。这也就意味着，编译完`PHP`脚本之后，可以丢弃这些符号的名字，都转换成地址即可。

而名字到地址的转换，核心函数是`lookup_cv`：

```cpp
static int lookup_cv(zend_string *name) /* {{{ */{
    zend_op_array *op_array = CG(active_op_array);
    int i = 0;
    zend_ulong hash_value = zend_string_hash_val(name);

    while (i < op_array->last_var) {
        if (ZSTR_H(op_array->vars[i]) == hash_value
        && zend_string_equals(op_array->vars[i], name)) {
            return EX_NUM_TO_VAR(i);
        }
        i++;
    }
    i = op_array->last_var;
    op_array->last_var++;
    if (op_array->last_var > CG(context).vars_size) {
        CG(context).vars_size += 16; /* FIXME */
        op_array->vars = erealloc(op_array->vars, CG(context).vars_size * sizeof(zend_string*));
    }

    op_array->vars[i] = zend_string_copy(name);
    return EX_NUM_TO_VAR(i);
}
```

这段代码，就是用来确定一个个`CV`变量在栈中的存储地址。也就意味着，栈的大小，在编译期间就确定好了。
