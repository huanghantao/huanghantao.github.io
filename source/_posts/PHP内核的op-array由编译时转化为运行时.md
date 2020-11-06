---
title: PHP内核的op_array由编译时转化为运行时
date: 2020-10-31 23:00:46
tags:
- PHP
- PHP内核
---

`PHP`内核在`pass_two`这个函数里面，会对`op_array`进行一个编译时到运行时的转化。

主要体现在以下几个地方：

## 重新分配literals

让`literals`和`opcodes`由原来分散存储的内存合并为连续的一块内存。这么做除了内存连续带来的性能提升之外，另一个好处是，在执行`opline`的时候，直接通过偏移量就可以拿到对应的字面量了，不需要传递`op_array`，相当于少传递了一个参数（之前需要通过`op_array->literals`的方式来获取）。

## 重新设置临时变量的var值

> znode_op::var最终是要存储这个变量相对`execute_data`的偏移量

我们知道，`IS_CV`变量它相对`execute_data`的偏移量在编译这个变量的时候就已经通过`EX_NUM_TO_VAR`确定了。但是，`IS_TMP`类型的变量，它的`znode_op::var`里面只存了这个临时变量是第几个，还没有确定这个临时变量相对`execute_data`的偏移量。所以，在编译时转化为运行时的阶段，需要确定好。

那么为什么只有`IS_TMP`需要做转化呢？而`IS_CV`不需要呢？这是和`PHP`栈帧的设计有关的，`PHP`的栈帧结构如下：

```bash
/*
 * Stack Frame Layout (the whole stack frame is allocated at once)
 * ==================
 *
 *                             +========================================+
 * EG(current_execute_data) -> | zend_execute_data                      |
 *                             +----------------------------------------+
 *     EX_VAR_NUM(0) --------> | VAR[0] = ARG[1]                        |
 *                             | ...                                    |
 *                             | VAR[op_array->num_args-1] = ARG[N]     |
 *                             | ...                                    |
 *                             | VAR[op_array->last_var-1]              |
 *                             | VAR[op_array->last_var] = TMP[0]       |
 *                             | ...                                    |
 *                             | VAR[op_array->last_var+op_array->T-1]  |
 *                             | ARG[N+1] (extra_args)                  |
 *                             | ...                                    |
 *                             +----------------------------------------+
 */
```

可以发现，前面是`IS_CV`类型的变量，`IS_TMP`类型的变量在`IS_CV`变量的后面。所以，我们在编译出`IS_TMP`的时候，还无法确定`IS_CV`变量的个数，所以，也就无法确定`IS_TMP`相对于`execute_data`的偏移量。所以，得把`IS_TMP`的转化放在后面进行。
