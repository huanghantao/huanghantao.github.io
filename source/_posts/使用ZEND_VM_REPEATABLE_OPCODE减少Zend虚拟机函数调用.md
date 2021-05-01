---
title: 使用ZEND_VM_REPEATABLE_OPCODE减少Zend虚拟机函数调用
date: 2021-04-30 15:45:54
tags:
- PHP
- PHP内核
---

> 本文基于PHP8.0.1

我们先来看看对应的宏：

```cpp
#define ZEND_VM_REPEATABLE_OPCODE \
    do {
#define ZEND_VM_REPEAT_OPCODE(_opcode) \
    } while (UNEXPECTED((++opline)->opcode == _opcode)); \
    OPLINE = opline; \
    ZEND_VM_CONTINUE()
```

可以看到，在`ZEND_VM_REPEATABLE_OPCODE`和`ZEND_VM_REPEAT_OPCODE`两个宏之间，会判断下一个`opcode`是否和当前的`opcode`一样，如果一样，那么再次进入循环。这可以运用在`opline`的`handler`里面，比如说如下脚本：

```php
<?php

function foo($a = 1, $b = 2)
{
    var_dump($a, $b);
}

foo();
```

函数`foo`生成的`opcodes`如下：

```bash
L3-6 foo() /Users/codinghuang/.phpbrew/build/php-8.0.1/test5.php - 0x10e85f3c0 + 7 ops
 L3    #0     RECV_INIT               1                    1                    $a
 L3    #1     RECV_INIT               2                    2                    $b
 L5    #2     INIT_FCALL<2>           112                  "var_dump"
 L5    #3     SEND_VAR                $a                   1
 L5    #4     SEND_VAR                $b                   2
 L5    #5     DO_ICALL
 L6    #6     RETURN<-1>              null
```

可以看到，当函数有默认参数的时候，会通过这个`ZEND_RECV_INIT`来接收参数的值。

```cpp
static ZEND_VM_HOT ZEND_OPCODE_HANDLER_RET ZEND_FASTCALL ZEND_RECV_INIT_SPEC_CONST_HANDLER(ZEND_OPCODE_HANDLER_ARGS)
{
    USE_OPLINE
    uint32_t arg_num;
    zval *param;

    ZEND_VM_REPEATABLE_OPCODE

    arg_num = opline->op1.num;
    param = EX_VAR(opline->result.var);
    if (arg_num > EX_NUM_ARGS()) {
        zval *default_value = RT_CONSTANT(opline, opline->op2);

        if (Z_OPT_TYPE_P(default_value) == IS_CONSTANT_AST) {
            zval *cache_val = (zval*)CACHE_ADDR(Z_CACHE_SLOT_P(default_value));

            /* we keep in cache only not refcounted values */
            if (Z_TYPE_P(cache_val) != IS_UNDEF) {
                ZVAL_COPY_VALUE(param, cache_val);
            } else {
                SAVE_OPLINE();
                ZVAL_COPY(param, default_value);
                if (UNEXPECTED(zval_update_constant_ex(param, EX(func)->op_array.scope) != SUCCESS)) {
                    zval_ptr_dtor_nogc(param);
                    ZVAL_UNDEF(param);
                    HANDLE_EXCEPTION();
                }
                if (!Z_REFCOUNTED_P(param)) {
                    ZVAL_COPY_VALUE(cache_val, param);
                }
            }
            goto recv_init_check_type;
        } else {
            ZVAL_COPY(param, default_value);
        }
    } else {
recv_init_check_type:
        if (UNEXPECTED((EX(func)->op_array.fn_flags & ZEND_ACC_HAS_TYPE_HINTS) != 0)) {
            SAVE_OPLINE();
            if (UNEXPECTED(!zend_verify_recv_arg_type(EX(func), arg_num, param, CACHE_ADDR(opline->extended_value)))) {
                HANDLE_EXCEPTION();
            }
        }
    }

    ZEND_VM_REPEAT_OPCODE(ZEND_RECV_INIT);
    ZEND_VM_NEXT_OPCODE();
}
```

这里就用到了这个优化，连续的两个`opcode`是一样的，所以在第一条`ZEND_RECV_INIT`执行完之后，不会退出这个函数，而是回到了`ZEND_VM_REPEATABLE_OPCODE`处，继续执行下一条`opline`。
