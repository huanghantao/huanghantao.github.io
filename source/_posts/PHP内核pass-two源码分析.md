---
title: PHP内核pass_two源码分析
date: 2020-10-21 12:34:42
tags:
- PHP内核
---

> 本文基于的PHP8 commit为：14806e0824ecd598df74cac855868422e44aea53

我们先来看一下`PHP`脚本到`opcode`的生成流程，在函数`zend_compile`里面：

```cpp
// 删除了部分代码
static zend_op_array *zend_compile(int type)
{
    if (!zendparse()) {
        init_op_array(op_array, type, INITIAL_OP_ARRAY_SIZE);

        zend_compile_top_stmt(CG(ast));
        pass_two(op_array);
    }

    return op_array;
}
```

总结起来如下：

```cpp
1. 调用zendparse完成词法分析、语法分析从而生成AST。
2. 调用init_op_array, zend_compile_top_stmt来完成AST到opcode的转化，此时还没有设置opcode对应的handler，以及有一部分东西是编译时的。
3. 调用pass_two完成编译时到运行时信息的转化、设置opcode对应的handler。
```

我们来看看具体的代码：

```cpp
ZEND_API void pass_two(zend_op_array *op_array)
{
    zend_op *opline, *end;

    if (!ZEND_USER_CODE(op_array->type)) {
        return;
    }

#if ZEND_USE_ABS_CONST_ADDR
    if (CG(context).opcodes_size != op_array->last) {
        op_array->opcodes = (zend_op *) erealloc(op_array->opcodes, sizeof(zend_op)*op_array->last);
        CG(context).opcodes_size = op_array->last;
    }
    if (CG(context).literals_size != op_array->last_literal) {
        op_array->literals = (zval*)erealloc(op_array->literals, sizeof(zval) * op_array->last_literal);
        CG(context).literals_size = op_array->last_literal;
    }
#else
    op_array->opcodes = (zend_op *) erealloc(op_array->opcodes,
        ZEND_MM_ALIGNED_SIZE_EX(sizeof(zend_op) * op_array->last, 16) +
        sizeof(zval) * op_array->last_literal);
    if (op_array->literals) {
        memcpy(((char*)op_array->opcodes) + ZEND_MM_ALIGNED_SIZE_EX(sizeof(zend_op) * op_array->last, 16),
            op_array->literals, sizeof(zval) * op_array->last_literal);
        efree(op_array->literals);
        op_array->literals = (zval*)(((char*)op_array->opcodes) + ZEND_MM_ALIGNED_SIZE_EX(sizeof(zend_op) * op_array->last, 16));
    }
    CG(context).opcodes_size = op_array->last;
    CG(context).literals_size = op_array->last_literal;
#endif

    /* Needs to be set directly after the opcode/literal reallocation, to ensure destruction
    * happens correctly if any of the following fixups generate a fatal error. */
    op_array->fn_flags |= ZEND_ACC_DONE_PASS_TWO;

    opline = op_array->opcodes;
    end = opline + op_array->last;
    while (opline < end) {
        if (opline->op1_type == IS_CONST) {
            ZEND_PASS_TWO_UPDATE_CONSTANT(op_array, opline, opline->op1);
        } else if (opline->op1_type & (IS_VAR|IS_TMP_VAR)) {
            opline->op1.var = EX_NUM_TO_VAR(op_array->last_var + opline->op1.var);
        }
        if (opline->op2_type == IS_CONST) {
            ZEND_PASS_TWO_UPDATE_CONSTANT(op_array, opline, opline->op2);
        } else if (opline->op2_type & (IS_VAR|IS_TMP_VAR)) {
            opline->op2.var = EX_NUM_TO_VAR(op_array->last_var + opline->op2.var);
        }
        if (opline->result_type & (IS_VAR|IS_TMP_VAR)) {
            opline->result.var = EX_NUM_TO_VAR(op_array->last_var + opline->result.var);
        }
        ZEND_VM_SET_OPCODE_HANDLER(opline);
        opline++;
    }

    return;
}
```

其中：

```cpp
#if ZEND_USE_ABS_CONST_ADDR
    if (CG(context).opcodes_size != op_array->last) {
        op_array->opcodes = (zend_op *) erealloc(op_array->opcodes, sizeof(zend_op)*op_array->last);
        CG(context).opcodes_size = op_array->last;
    }
    if (CG(context).literals_size != op_array->last_literal) {
        op_array->literals = (zval*)erealloc(op_array->literals, sizeof(zval) * op_array->last_literal);
        CG(context).literals_size = op_array->last_literal;
    }
#else
```

是在`32`位的机器上面进行设置的，此时，会重新分配`opcodes`和`literals`，可以避免内存的浪费。

```cpp
    op_array->opcodes = (zend_op *) erealloc(op_array->opcodes,
        ZEND_MM_ALIGNED_SIZE_EX(sizeof(zend_op) * op_array->last, 16) +
        sizeof(zval) * op_array->last_literal);
    if (op_array->literals) {
        memcpy(((char*)op_array->opcodes) + ZEND_MM_ALIGNED_SIZE_EX(sizeof(zend_op) * op_array->last, 16),
            op_array->literals, sizeof(zval) * op_array->last_literal);
        efree(op_array->literals);
        op_array->literals = (zval*)(((char*)op_array->opcodes) + ZEND_MM_ALIGNED_SIZE_EX(sizeof(zend_op) * op_array->last, 16));
    }
    CG(context).opcodes_size = op_array->last;
    CG(context).literals_size = op_array->last_literal;
#endif
```

是在`64`位的机器上面进行设置的，此时，会重新分配`opcodes`，大小是`opline`的条数加上字面量的个数，然后把`literals`拷贝到`opcodes`的最后面。这样，使得`opcodes`和`literals`是在一块连续的内存上面。

```cpp
while (opline < end) {
        if (opline->op1_type == IS_CONST) {
            c(op_array, opline, opline->op1);
        } else if (opline->op1_type & (IS_VAR|IS_TMP_VAR)) {
            opline->op1.var = EX_NUM_TO_VAR(op_array->last_var + opline->op1.var);
        }
        if (opline->op2_type == IS_CONST) {
            ZEND_PASS_TWO_UPDATE_CONSTANT(op_array, opline, opline->op2);
        } else if (opline->op2_type & (IS_VAR|IS_TMP_VAR)) {
            opline->op2.var = EX_NUM_TO_VAR(op_array->last_var + opline->op2.var);
        }
        if (opline->result_type & (IS_VAR|IS_TMP_VAR)) {
            opline->result.var = EX_NUM_TO_VAR(op_array->last_var + opline->result.var);
        }
        ZEND_VM_SET_OPCODE_HANDLER(opline);
        opline++;
    }
```

调用`ZEND_PASS_TWO_UPDATE_CONSTANT`来完成常量编译时到运行时的转换。我们来看看这个宏：

```cpp
/* constant-time constant */
# define CT_CONSTANT_EX(op_array, num) \
    ((op_array)->literals + (num))

# define CT_CONSTANT(node) \
    CT_CONSTANT_EX(CG(active_op_array), (node).constant)

#if ZEND_USE_ABS_CONST_ADDR

/* run-time constant */
# define RT_CONSTANT(opline, node) \
    (node).zv

/* convert constant from compile-time to run-time */
# define ZEND_PASS_TWO_UPDATE_CONSTANT(op_array, opline, node) do { \
        (node).zv = CT_CONSTANT_EX(op_array, (node).constant); \
    } while (0)

#else

/* At run-time, constants are allocated together with op_array->opcodes
* and addressed relatively to current opline.
*/

/* run-time constant */
# define RT_CONSTANT(opline, node) \
    ((zval*)(((char*)(opline)) + (int32_t)(node).constant))

/* convert constant from compile-time to run-time */
# define ZEND_PASS_TWO_UPDATE_CONSTANT(op_array, opline, node) do { \
        (node).constant = \
            (((char*)CT_CONSTANT_EX(op_array, (node).constant)) - \
            ((char*)opline)); \
    } while (0)

#endif
```

在`32`位的机器上，走的逻辑是：

```cpp
# define CT_CONSTANT_EX(op_array, num) \
    ((op_array)->literals + (num))

/* run-time constant */
# define RT_CONSTANT(opline, node) \
    (node).zv

/* convert constant from compile-time to run-time */
# define ZEND_PASS_TWO_UPDATE_CONSTANT(op_array, opline, node) do { \
        (node).zv = CT_CONSTANT_EX(op_array, (node).constant); \
    } while (0)
```

我们知道，在编译的时候，`(node).constant`存的是字面量在`(op_array)->literals`的索引，也就是`1`，`2`，`3`等等。

而进行编译时到运行时的转换后，`(node).constant`存的就是字面量在`(op_array)->literals`的绝对地址了。

我们再来看看`64`位的机器上，走的逻辑是：

```cpp
# define CT_CONSTANT_EX(op_array, num) \
    ((op_array)->literals + (num))

/* run-time constant */
# define RT_CONSTANT(opline, node) \
    ((zval*)(((char*)(opline)) + (int32_t)(node).constant))

/* convert constant from compile-time to run-time */
# define ZEND_PASS_TWO_UPDATE_CONSTANT(op_array, opline, node) do { \
        (node).constant = \
            (((char*)CT_CONSTANT_EX(op_array, (node).constant)) - \
            ((char*)opline)); \
    } while (0)

#endif
```

我们发现，进行编译时到运行时的转换后，`(node).constant`存的就是字面量相对当前`opline`的相对地址了。因为在`64`位的机器上，`opcodes`和`literals`是在一块连续的内存上面，所以可以存一个相对地址。如下图：

```bash
+----------------------------+   
|          opcodes           |   
+----------------------------+   
                                 
+----------------------------+   
|          opline1           |--+
+----------------------------+  |
|          opline2           |  |
+----------------------------+  |
|          opline3           |  |
+----------------------------+  |
|                            |  |
|           ......           |  |
|     Continuous memory      |  |
|                            |  |
|                            |  |
+----------------------------+  |
|          literal1          |<-+
+----------------------------+   
|          literal2          |   
+----------------------------+   
|          literal3          |   
+----------------------------+   
|                            |   
|           ......           |   
|                            |   
+----------------------------+   
```

```cpp
ZEND_VM_SET_OPCODE_HANDLER(opline);
```

这一步就是设置我们`opcode`对应的`handler`了。
