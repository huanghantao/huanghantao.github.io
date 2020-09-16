---
title: PHP7源码分析--数组
date: 2019-04-28 15:09:03
tags:
- PHP7
---

这是一篇调试PHP数组的文章，希望对大家有所帮助。PHP的版本是**7.1.0**。这里，不会长篇大论的去讲解原因，只会告诉小伙伴们需要如何去做。至于为什么需要这么做，网上应该是有这种源码剖析的文章。

我们需要调试的脚本如下：

```php
<?php // 1

$a = array(); // 3

$a[1] = 'a'; // 5

$a[] = 'b'; // 7

$a['key1'] = 'value1'; // 9
$a['key2'] = 'value2'; // 10

echo $a['key1']; // 12

$a['key1'] = 'c'; // 14

unset($a['key2']); // 16
```

OK，我们开始进行调试，分析数组初始化的过程：

```shell
sh-4.2# cgdb php
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /home/codes/php/php-7.1.0/output/bin/php...done.
(gdb) 
```

然后，我们需要知道PHP7数组初始化的过程：

```
1、申请内存
/* fast cache for HashTables */
#define ALLOC_HASHTABLE(ht)	\
	(ht) = (HashTable *) emalloc(sizeof(HashTable))
2、调用_zend_hash_init方法
```

并且，对于array()这种方式，PHP7会在编译阶段就创建一个数组常量。因此，数组的初始化发生在编译阶段。

我们在`zend_compile`处打一个断点：

```shell
(gdb) b zend_compile
Breakpoint 1 at 0x7f85f8: file Zend/zend_language_scanner.l, line 578.
(gdb) 
```

然后运行脚本：

```shell
(gdb) r array.php 
Starting program: /home/codes/php/php-7.1.0/output/bin/php array.php
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".

Breakpoint 1, zend_compile (type=2) at Zend/zend_language_scanner.l:578
Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7_6.4.x86_64
(gdb) 
```

然后，我们在`_zend_hash_init`处打一个断点：

```shell
(gdb) b _zend_hash_init
Breakpoint 2 at 0x86167d: file /root/php-7.1.0/Zend/zend_hash.c, line 173.
(gdb) 
```

继续执行：

```shell
(gdb) c
Continuing.

Breakpoint 2, _zend_hash_init (ht=0x7ffff5e59360, nSize=0, pDestructor=0x84d18f <_zval_ptr_dtor_wrapper>, persistent=0 '\000', __zend_filename=0xd871c8 "/root/php-7.1.0/Zend/zend_compile.c", __zend_lineno
=6597) at /root/php-7.1.0/Zend/zend_hash.c:173
(gdb)
```

`_zend_hash_init`的代码如下：

```c
ZEND_API void ZEND_FASTCALL _zend_hash_init(HashTable *ht, uint32_t nSize, dtor_func_t pDestructor, zend_bool persistent ZEND_FILE_LINE_DC)
 172│ {
 173├───────> GC_REFCOUNT(ht) = 1;
 174│         GC_TYPE_INFO(ht) = IS_ARRAY;
 175│         ht->u.flags = (persistent ? HASH_FLAG_PERSISTENT : 0) | HASH_FLAG_APPLY_PROTECTION | HASH_FLAG_STATIC_KEYS;
 176│         ht->nTableSize = zend_hash_check_size(nSize);
 177│         ht->nTableMask = HT_MIN_MASK;
 178│         HT_SET_DATA_ADDR(ht, &uninitialized_bucket);
 179│         ht->nNumUsed = 0;
 180│         ht->nNumOfElements = 0;
 181│         ht->nInternalPointer = HT_INVALID_IDX;
 182│         ht->nNextFreeElement = 0;
 183│         ht->pDestructor = pDestructor;
 184│ }
```

可以看出，`HashTable`即`zend_array`的地址为`0x7ffff5e59360`。

我们继续往下走，即初始化`zend_array`的引用计数为1：

```shell
(gdb) n
(gdb) 
```

我们查看一下这个`zend_array`里面的内容：

```shell
(gdb) p	*ht
$1 = {gc = {refcount = 1, u = {v = {type = 255 '\377', flags = 127 '\177', gc_info = 0}, type_info = 32767}}, u = {v = {flags = 0 '\000', nApplyCount = 0 '\000', nIteratorsCount = 0 '\000', consistency =
0 '\000'}, flags = 0}, nTableMask = 0, arData = 0x0, nNumUsed = 0, nNumOfElements = 0, nTableSize = 0, nInternalPointer = 0, nNextFreeElement = 0, pDestructor = 0x0}
(gdb) 
```

可以看出`zend_array.gc.refcount`为1。即初始化一个数组的时候，它的引用计数是1。`arData`里面存放的地址为`0x0`，所以，还没有为`bucket`分配内存。

继续执行代码到`zend_hash.c:177`之前：

```shell
(gdb) u	177
_zend_hash_init	(ht=0x10d6f40 <sapi_globals+448>, nSize=8, pDestructor=0x7cac14 <_type_dtor>, persistent=1 '\001', __zend_filename=0xd75821 "/root/php-7.1.0/main/SAPI.c", __zend_lineno=65) at /root/php-7.
1.0/Zend/zend_hash.c:177
(gdb) 
```

```c
 171│ ZEND_API void ZEND_FASTCALL _zend_hash_init(HashTable *ht, uint32_t nSize, dtor_func_t pDestructor, zend_bool persistent ZEND_FILE_LINE_DC)
 172│ {
 173│         GC_REFCOUNT(ht) = 1;
 174│         GC_TYPE_INFO(ht) = IS_ARRAY;
 175│         ht->u.flags = (persistent ? HASH_FLAG_PERSISTENT : 0) | HASH_FLAG_APPLY_PROTECTION | HASH_FLAG_STATIC_KEYS;
 176│         ht->nTableSize = zend_hash_check_size(nSize);
 177├───────> ht->nTableMask = HT_MIN_MASK;
 178│         HT_SET_DATA_ADDR(ht, &uninitialized_bucket);
 179│         ht->nNumUsed = 0;
 180│         ht->nNumOfElements = 0;
 181│         ht->nInternalPointer = HT_INVALID_IDX;
 182│         ht->nNextFreeElement = 0;
 183│         ht->pDestructor = pDestructor;
 184│ }
```

`_zend_hash_init`中174到176行的代码都是一些初始化的操作。

重点看执行完176行之后`nTableSize`的值：

```shell
(gdb) p ht->nTableSize 
$3 = 8
(gdb) 
```

`nTableSize`是`zend_array`的大小，而这个8就是`zend_array`的最小大小。

继续往下运行：

```shell
 177│         ht->nTableMask = HT_MIN_MASK;
 178├───────> HT_SET_DATA_ADDR(ht, &uninitialized_bucket);
```

执行完177行之后，我们来看一下`nTableMask`的值：

```shell
(gdb) p	ht->nTableMask
$4 = 4294967294
(gdb) 
```

这是以无符号整型打印出来的数字，我们需要以有符号的格式来打印：

```shell
(gdb) p (int32_t)ht->nTableMask
$5 = -2
(gdb) 
```

可以看出，在`zend_array`初始化的时候，`nTableMask`的值为-2。

继续执行178行：

```shell
 178│         HT_SET_DATA_ADDR(ht, &uninitialized_bucket);
 179├───────> ht->nNumUsed = 0;
```

`HT_SET_DATA_ADDR`是一个宏：

```c
#define HT_SET_DATA_ADDR(ht, ptr) do { \
		(ht)->arData = (Bucket*)(((char*)(ptr)) + HT_HASH_SIZE((ht)->nTableMask)); \
	} while (0)
```

它的作用初始化`arData`的地址，我们来打印一下：

```shell
(gdb) p ht->arData
$7 = (Bucket *) 0xd8eacc
(gdb) 
```

OK，`_zend_hash_init`函数剩下的部分都是简单的初始化了，我们直接走完：

```shell
(gdb) u	184
_zend_hash_init	(ht=0x10d6f40 <sapi_globals+448>, nSize=8, pDestructor=0x7cac14 <_type_dtor>, persistent=1 '\001', __zend_filename=0xd75821 "/root/php-7.1.0/main/SAPI.c", __zend_lineno=65) at /root/php-7.
1.0/Zend/zend_hash.c:184
(gdb) 
```

```c
 172│ {
 173│         GC_REFCOUNT(ht) = 1;
 174│         GC_TYPE_INFO(ht) = IS_ARRAY;
 175│         ht->u.flags = (persistent ? HASH_FLAG_PERSISTENT : 0) | HASH_FLAG_APPLY_PROTECTION | HASH_FLAG_STATIC_KEYS;
 176│         ht->nTableSize = zend_hash_check_size(nSize);
 177│         ht->nTableMask = HT_MIN_MASK;
 178│         HT_SET_DATA_ADDR(ht, &uninitialized_bucket);
 179│         ht->nNumUsed = 0;
 180│         ht->nNumOfElements = 0;
 181│         ht->nInternalPointer = HT_INVALID_IDX;
 182│         ht->nNextFreeElement = 0;
 183│         ht->pDestructor = pDestructor;
 184├>}
```

我们来看一下`zend_array` 的游标`nInternalPointer`的值：

```shell
(gdb) p (int32_t)ht->nInternalPointer
$9 = -1
(gdb) 
```

可以看出，在`zend_init`初始化的时候，游标的值为-1。

OK，数组现在初始化完毕。以上过程是发生在PHP脚本被编译的过程：

```shell
(gdb) bt
#0  _zend_hash_init (ht=0x7ffff5e59360, nSize=0, pDestructor=0x84d18f <_zval_ptr_dtor_wrapper>, persistent=0 '\000', __zend_filename=0xd871c8 "/root/php-7.1.0/Zend/zend_compile.c", __zend_lineno=6597) at
/root/php-7.1.0/Zend/zend_hash.c:184
#1  0x0000000000855499 in _array_init (arg=0x7fffffffb018, size=0, __zend_filename=0xd871c8 "/root/php-7.1.0/Zend/zend_compile.c", __zend_lineno=6597) at /root/php-7.1.0/Zend/zend_API.c:1061
#2  0x000000000082f11a in zend_try_ct_eval_array (result=0x7fffffffb018, ast=0x7ffff5e72070) at /root/php-7.1.0/Zend/zend_compile.c:6597
#3  0x0000000000830bc3 in zend_compile_array (result=0x7fffffffb010, ast=0x7ffff5e72070) at /root/php-7.1.0/Zend/zend_compile.c:7203
#4  0x0000000000832f20 in zend_compile_expr (result=0x7fffffffb010, ast=0x7ffff5e72070) at /root/php-7.1.0/Zend/zend_compile.c:7951
#5  0x00000000008247d3 in zend_compile_assign (result=0x7fffffffb0d0, ast=0x7ffff5e720a0) at /root/php-7.1.0/Zend/zend_compile.c:2975
#6  0x0000000000832ce0 in zend_compile_expr (result=0x7fffffffb0d0, ast=0x7ffff5e720a0) at /root/php-7.1.0/Zend/zend_compile.c:7873
#7  0x00000000008329b4 in zend_compile_stmt (ast=0x7ffff5e720a0) at /root/php-7.1.0/Zend/zend_compile.c:7839
#8  0x0000000000832590 in zend_compile_top_stmt (ast=0x7ffff5e720a0) at /root/php-7.1.0/Zend/zend_compile.c:7725
#9  0x0000000000832572 in zend_compile_top_stmt (ast=0x7ffff5e722c0) at /root/php-7.1.0/Zend/zend_compile.c:7720
#10 0x00000000007f871f in zend_compile (type=2) at Zend/zend_language_scanner.l:601
#11 0x00000000007f886f in compile_file (file_handle=0x7fffffffe920, type=8) at Zend/zend_language_scanner.l:635
#12 0x0000000000692a03 in phar_compile_file (file_handle=0x7fffffffe920, type=8) at /root/php-7.1.0/ext/phar/phar.c:3305
#13 0x000000000085063c in zend_execute_scripts (type=8, retval=0x0, file_count=3) at /root/php-7.1.0/Zend/zend.c:1468
#14 0x00000000007c077f in php_execute_script (primary_file=0x7fffffffe920) at /root/php-7.1.0/main/main.c:2533
#15 0x000000000092f855 in do_cli (argc=2, argv=0x10dc280) at /root/php-7.1.0/sapi/cli/php_cli.c:990
#16 0x0000000000930814 in main (argc=2, argv=0x10dc280) at /root/php-7.1.0/sapi/cli/php_cli.c:1378
(gdb) 
```

OK，分析完数组初始化的过程，现在我们来分析一下脚本的第5行：

```php
$a[1] = 'a'; // 5
```

也就是把字符`'a'`插入到数组里面。

这个过程是发生在`zend`虚拟机执行opcode阶段。所以我们在`zend_execute`处打一个断点，并且执行到`zend_execute`位置：

```shell
(gdb) b	zend_execute
Breakpoint 3 at 0x8aea82: file /root/php-7.1.0/Zend/zend_vm_execute.h, line 461.
(gdb) c    
Continuing.

Breakpoint 3, zend_execute (op_array=0x7ffff5e7b000, return_value=0x0) at /root/php-7.1.0/Zend/zend_vm_execute.h:461
(gdb) 
```

```c
  457│ ZEND_API void zend_execute(zend_op_array *op_array, zval *return_value)
  458│ {
  459│         zend_execute_data *execute_data;
  460│
  461├───────> if (EG(exception) != NULL) {
  462│                 return;
  463│         }
```

我们继续执行，执行到474行：

```shell
(gdb) u	474
zend_execute (op_array=0x7ffff5e7b000, return_value=0x0) at /root/php-7.1.0/Zend/zend_vm_execute.h:474
(gdb) 
```

```c
  472│         EX(prev_execute_data) = EG(current_execute_data);
  473│         i_init_execute_data(execute_data, op_array, return_value);
  474├───────> zend_execute_ex(execute_data);
  475│         zend_vm_stack_free_call_frame(execute_data);
  476│ }
```

OK，我们进入`zend_execute_ex`函数里面：

```shell
(gdb) s
execute_ex (ex=0x7ffff5e14030) at /root/php-7.1.0/Zend/zend_vm_execute.h:411
(gdb) 
```

```c
  406│ ZEND_API void execute_ex(zend_execute_data *ex)
  407│ {
  408│         DCL_OPLINE
  409│
  410│ #ifdef ZEND_VM_IP_GLOBAL_REG
  411├───────> const zend_op *orig_opline = opline;
  412│ #endif
  413│ #ifdef ZEND_VM_FP_GLOBAL_REG
  414│         zend_execute_data *orig_execute_data = execute_data;
  415│         execute_data = ex;
  416│ #else
  417│         zend_execute_data *execute_data = ex;
  418│ #endif
```

我们继续让代码执行到429行之前：

```shell
(gdb) u	429
execute_ex (ex=0x7ffff5e14030) at /root/php-7.1.0/Zend/zend_vm_execute.h:429
(gdb) 
```

```c
  425│ #if !defined(ZEND_VM_FP_GLOBAL_REG) || !defined(ZEND_VM_IP_GLOBAL_REG)
  426│                         int ret;
  427│ #endif
  428│ #if defined(ZEND_VM_FP_GLOBAL_REG) && defined(ZEND_VM_IP_GLOBAL_REG)
  429├───────────────> ((opcode_handler_t)OPLINE->handler)(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU);
  430│                 if (UNEXPECTED(!OPLINE)) {
  431│ #else
```

然后进入这个handler里面：

```shell
(gdb) s
ZEND_ASSIGN_SPEC_CV_CONST_RETVAL_UNUSED_HANDLER () at /root/php-7.1.0/Zend/zend_vm_execute.h:39440
(gdb) 
```

```c
39433│ static ZEND_OPCODE_HANDLER_RET ZEND_FASTCALL ZEND_ASSIGN_SPEC_CV_CONST_RETVAL_UNUSED_HANDLER(ZEND_OPCODE_HANDLER_ARGS)
39434│ {
39435│         USE_OPLINE
39436│
39437│         zval *value;
39438│         zval *variable_ptr;
39439│
39440├───────> SAVE_OPLINE();
39441│         value = EX_CONSTANT(opline->op2);
39442│         variable_ptr = _get_zval_ptr_cv_undef_BP_VAR_W(execute_data, opline->op1.var);
```

我们继续执行完39441处的代码：

```shell
(gdb) u	39442
ZEND_ASSIGN_SPEC_CV_CONST_RETVAL_UNUSED_HANDLER () at /root/php-7.1.0/Zend/zend_vm_execute.h:39442
(gdb) 
```

```c
39440│         SAVE_OPLINE();
39441│         value = EX_CONSTANT(opline->op2);
39442├───────> variable_ptr = _get_zval_ptr_cv_undef_BP_VAR_W(execute_data, opline->op1.var);
```

此时，我们得到了一个`value`。我们打印一下这个`value`的内容：

```shell
(gdb) p	value
$1 = (zval *) 0x7ffff5e7b100
(gdb) 
```

可以看到，这是一个`zval`结构，我们看看这个`zval`里面的内容：

```shell
(gdb) p	*value
$2 = {value = {lval = 140737318851424, dval = 6.9533474332294241e-310, counted = 0x7ffff5e59360, str = 0x7ffff5e59360, arr = 0x7ffff5e59360, obj = 0x7ffff5e59360, res = 0x7ffff5e59360, ref = 0x7ffff5e5936
0, ast = 0x7ffff5e59360, zv = 0x7ffff5e59360, ptr = 0x7ffff5e59360, ce = 0x7ffff5e59360, func = 0x7ffff5e59360, ww = {w1 = 4125463392, w2 = 32767}}, u1 = {v = {type = 7 '\a', type_flags = 28 '\034', const
_flags = 0 '\000', reserved = 0 '\000'}, type_info = 7175}, u2 = {next = 4294967295, cache_slot = 4294967295, lineno = 4294967295, num_args = 4294967295, fe_pos = 4294967295, fe_iter_idx = 4294967295, acc
ess_flags = 4294967295, property_guard = 4294967295}}
(gdb) 
```

因为`zval.u1.v.type`为7，所以这个`zval.value`是`zend_array`类型。所以我们应该取`zval`里面的`arr`，我们发现`arr`里面存放的地址是`0x7ffff5e59360`，而这个地址正是我们之前讲解数组初始化时候的那个`zend_array`的地址。

我们继续往下执行：

```shell
(gdb) n
(gdb) 
```

```c
39450├───────────────> value = zend_assign_to_variable(variable_ptr, value, IS_CONST);
39451│                 if (UNEXPECTED(0)) {
39452│                         ZVAL_COPY(EX_VAR(opline->result.var), value);
39453│                 }
39454│
39455│                 /* zend_assign_to_variable() always takes care of op2, never free it! */
39456│         }
39457│
39458│         ZEND_VM_NEXT_OPCODE_CHECK_EXCEPTION();
39459│ }
```

`zend_assign_to_variable`把`value`这个`zval`assign到`variable_ptr`这里。我们打印一下`variable_ptr`：

```shell
(gdb) p	variable_ptr
$4 = (zval *) 0x7ffff5e14080
(gdb) 
```

此时打印出来的是zend虚拟机的栈的起始地址。在执行`zend_assign_to_variable`之前，我们打印一下这个`zend_array`里面的内容：

```shell
(gdb) p	*value.value.arr
$9 = {gc = {refcount = 1, u = {v = {type = 7 '\a', flags = 0 '\000', gc_info = 0}, type_info = 7}}, u = {v = {flags = 18 '\022', nApplyCount = 0 '\000', nIteratorsCount = 0 '\000', consistency = 0 '\000'}
, flags = 18}, nTableMask = 4294967294, arData = 0xd8eacc, nNumUsed = 0, nNumOfElements = 0, nTableSize = 8, nInternalPointer = 4294967295, nNextFreeElement = 0, pDestructor = 0x84d18f <_zval_ptr_dtor_wra
pper>}
(gdb) 
```

可以看到，这里的`refcount`仍然是1，也就是此时只有`value`指向这个`zend_array`。

我们继续往下执行：

```shell
(gdb) n
(gdb) 
```

然后重新打印一下这个value的内容：

```shell
(gdb) p *value.value.arr
$10 = {gc = {refcount = 2, u = {v = {type = 7 '\a', flags = 0 '\000', gc_info = 0}, type_info = 7}}, u = {v = {flags = 18 '\022', nApplyCount = 0 '\000', nIteratorsCount = 0 '\000', consistency = 0 '\000'
}, flags = 18}, nTableMask = 4294967294, arData = 0xd8eacc, nNumUsed = 0, nNumOfElements = 0, nTableSize = 8, nInternalPointer = 4294967295, nNextFreeElement = 0, pDestructor = 0x84d18f <_zval_ptr_dtor_wr
apper>}
(gdb) 
```

我们发现，此时的`zend_array.gc.refcount`的值为2了。说明`zend_assign_to_variable`操作使得多了一个`zval`使用了这个`zend_array`。

我们继续执行：

```shell
(gdb) n
(gdb) n
execute_ex (ex=0x7ffff5e14030) at /root/php-7.1.0/Zend/zend_vm_execute.h:430
(gdb) n
(gdb) n
(gdb) 
```

```shell
  428│ #if defined(ZEND_VM_FP_GLOBAL_REG) && defined(ZEND_VM_IP_GLOBAL_REG)
  429├───────────────> ((opcode_handler_t)OPLINE->handler)(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU);
  430│                 if (UNEXPECTED(!OPLINE)) {
  431│ #else
  432│                 if (UNEXPECTED((ret = ((opcode_handler_t)OPLINE->handler)(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU)) != 0)) {
  433│ #endif
```

我们进入这个handler：

```shell
(gdb) s
ZEND_ASSIGN_DIM_SPEC_CV_CONST_OP_DATA_CONST_HANDLER () at /root/php-7.1.0/Zend/zend_vm_execute.h:39077
(gdb) 
```

```c
39077├───────> SAVE_OPLINE();
39078│         object_ptr = _get_zval_ptr_cv_undef_BP_VAR_W(execute_data, opline->op1.var);
```

我们继续执行完39078行：

```shell
(gdb) u	39079
ZEND_ASSIGN_DIM_SPEC_CV_CONST_OP_DATA_CONST_HANDLER () at /root/php-7.1.0/Zend/zend_vm_execute.h:39080
(gdb) 
```

然后打印这个`object_ptr`：

```shell
(gdb) p	object_ptr
$12 = (zval *) 0x7ffff5e14080
(gdb) 
```

这个和之前打印的那个zend虚拟机的栈的起始地址一致。我们查看一下里面的内容：

```shell
(gdb) p	*object_ptr    
$13 = {value = {lval = 140737318851424, dval = 6.9533474332294241e-310, counted = 0x7ffff5e59360, str = 0x7ffff5e59360, arr = 0x7ffff5e59360, obj = 0x7ffff5e59360, res = 0x7ffff5e59360, ref = 0x7ffff5e593
60, ast = 0x7ffff5e59360, zv = 0x7ffff5e59360, ptr = 0x7ffff5e59360, ce = 0x7ffff5e59360, func = 0x7ffff5e59360, ww = {w1 = 4125463392, w2 = 32767}}, u1 = {v = {type = 7 '\a', type_flags = 28 '\034', cons
t_flags = 0 '\000', reserved = 0 '\000'}, type_info = 7175}, u2 = {next = 0, cache_slot = 0, lineno = 0, num_args = 0, fe_pos = 0, fe_iter_idx = 0, access_flags = 0, property_guard = 0}}
(gdb) 
```

因为`zval.u1.v.typle`为7，所以它是一个数组。我们查看一下这个数组的内容：

```shell
(gdb) p	$13.value.arr 
$14 = (zend_array *) 0x7ffff5e59360
(gdb) p *$13.value.arr
$15 = {gc = {refcount = 2, u = {v = {type = 7 '\a', flags = 0 '\000', gc_info = 0}, type_info = 7}}, u = {v = {flags = 18 '\022', nApplyCount = 0 '\000', nIteratorsCount = 0 '\000', consistency = 0 '\000'
}, flags = 18}, nTableMask = 4294967294, arData = 0xd8eacc, nNumUsed = 0, nNumOfElements = 0, nTableSize = 8, nInternalPointer = 4294967295, nNextFreeElement = 0, pDestructor = 0x84d18f <_zval_ptr_dtor_wr
apper>}
(gdb) 
```

继续执行：

```shell
(gdb) n
(gdb) 
```

```c
39081│ try_assign_dim_array:
39082├───────────────> SEPARATE_ARRAY(object_ptr);
39083│                 if (IS_CONST == IS_UNUSED) {
```

`SEPARATE_ARRAY`会进行写时分离，我们执行一下：

```shell
(gdb) n
(gdb) 
```

```shell
39089│                 } else {
39090├───────────────────────> dim = EX_CONSTANT(opline->op2);
39091│                         if (IS_CONST == IS_CONST) {
```

我们来看一下`object_ptr`里面的内容：

```shell
(gdb) p *object_ptr 
$16 = {value = {lval = 140737318851520, dval = 6.9533474332341671e-310, counted = 0x7ffff5e593c0, str =	0x7ffff5e593c0, arr = 0x7ffff5e593c0, obj = 0x7ffff5e593c0, res = 0x7ffff5e593c0, ref = 0x7ffff5e593
c0, ast = 0x7ffff5e593c0, zv = 0x7ffff5e593c0, ptr = 0x7ffff5e593c0, ce = 0x7ffff5e593c0, func = 0x7ffff5e593c0, ww = {w1 = 4125463488, w2 = 32767}}, u1 = {v = {type = 7 '\a', type_flags = 28 '\034', cons
t_flags = 0 '\000', reserved = 0 '\000'}, type_info = 7175}, u2 = {next = 0, cache_slot = 0, lineno = 0, num_args = 0, fe_pos = 0, fe_iter_idx = 0, access_flags = 0, property_guard = 0}}
(gdb) 
```

此时，`object_ptr.u1.v.type`为7，所以依然是一个数组。但是，此时的`object_ptr.value.arr`存放的地址为`0x7ffff5e593c0`（`$a`中`zend_array`里面存放的地址），不再是之前的`0x7ffff5e59360`了，即不再是最初我们初始化的那个`zend_array`了。

我们来看看这个`object_ptr`中`zend_array`的内容：

```shell
(gdb) p *object_ptr.value.arr
$17 = {gc = {refcount = 1, u = {v = {type = 7 '\a', flags = 0 '\000', gc_info = 0}, type_info = 7}}, u = {v = {flags = 18 '\022', nApplyCount = 0 '\000', nIteratorsCount = 0 '\000', consistency = 0 '\000'
}, flags = 18}, nTableMask = 4294967294, arData = 0xd8eacc, nNumUsed = 0, nNumOfElements = 0, nTableSize = 8, nInternalPointer = 4294967295, nNextFreeElement = 0, pDestructor = 0x84d18f <_zval_ptr_dtor_wr
apper>}
(gdb) 
```

可以看到，这个`refcount`不再是2了，因为进行了写时分离，所以变成了1。

同时，我们再来看看之前初始化的那个`zend_array`的内容：

```shell
(gdb) p	*(zend_array *)0x7ffff5e59360
$18 = {gc = {refcount = 1, u = {v = {type = 7 '\a', flags = 0 '\000', gc_info = 0}, type_info = 7}}, u = {v = {flags = 18 '\022', nApplyCount = 0 '\000', nIteratorsCount = 0 '\000', consistency = 0 '\000'
}, flags = 18}, nTableMask = 4294967294, arData = 0xd8eacc, nNumUsed = 0, nNumOfElements = 0, nTableSize = 8, nInternalPointer = 4294967295, nNextFreeElement = 0, pDestructor = 0x84d18f <_zval_ptr_dtor_wr
apper>}
(gdb) 
```

可以看到，它的`refcount`也变了1。我们继续往下执行：

```shell
(gdb) n
(gdb)
```

```c
39090│                         dim = EX_CONSTANT(opline->op2);
39091│                         if (IS_CONST == IS_CONST) {
39092├───────────────────────────────> variable_ptr = zend_fetch_dimension_address_inner_W_CONST(Z_ARRVAL_P(object_ptr), dim);
```

我们打印一下这个`dim`的内容：

```shell
(gdb) p	dim
$19 = (zval *) 0x7ffff5e7b110
(gdb) 
```

发现这是一个zval，我们看看这个zval的内容：

```shell
(gdb) p	*dim
$20 = {value = {lval = 1, dval = 4.9406564584124654e-324, counted = 0x1, str = 0x1, arr = 0x1, obj = 0x1, res = 0x1, ref = 0x1, ast = 0x1, zv = 0x1, ptr = 0x1, ce = 0x1, func = 0x1, ww = {w1 = 1, w2 = 0}}
, u1 = {v = {type = 4 '\004', type_flags = 0 '\000', const_flags = 0 '\000', reserved = 0 '\000'}, type_info = 4}, u2 = {next = 4294967295, cache_slot = 4294967295, lineno = 4294967295, num_args = 4294967
295, fe_pos = 4294967295, fe_iter_idx = 4294967295, access_flags = 4294967295, property_guard = 4294967295}}
(gdb) 
```

因为`zval.u1.v.type`为4，所以它是一个整形，我们打印一下这个内容：

```shell
(gdb) p	dim.value.lval 
$21 = 1
(gdb) 
```

发现是1。而这个1就是PHP脚本中第5行：

```php
$a[1] = 'a'; // 5
```

中数组的索引1。

我们进入`zend_fetch_dimension_address_inner_W_CONST`这个函数里面：

```shell
(gdb) s
zend_fetch_dimension_address_inner_W_CONST (ht=0x7ffff5e593c0, dim=0x7ffff5e7b110) at /root/php-7.1.0/Zend/zend_execute.c:1651
(gdb) 
```

```c
1649│ static zend_never_inline zval* ZEND_FASTCALL zend_fetch_dimension_address_inner_W_CONST(HashTable *ht, const zval *dim)
1650│ {
1651├───────> return zend_fetch_dimension_address_inner(ht, dim, IS_CONST, BP_VAR_W);
1652│ }
```

继续进入：

```shell
(gdb) s
zend_fetch_dimension_address_inner (ht=0x7ffff5e593c0, dim=0x7ffff5e7b110, dim_type=1, type=1) at /root/php-7.1.0/Zend/zend_execute.c:1540
(gdb) 
```

```c
1539│ try_again:
1540├───────> if (EXPECTED(Z_TYPE_P(dim) == IS_LONG)) {
1541│                 hval = Z_LVAL_P(dim);
1542│ num_index:
1543│                 ZEND_HASH_INDEX_FIND(ht, hval, retval, num_undef);
1544│                 return retval;
```

因为`dim`是一个long类型，所以，直接取出dim里面的内容。我们继续执行，取出dim里面的lval：

```shell
(gdb) n
(gdb) n
(gdb) 
```

```shell
1539│ try_again:
1540│         if (EXPECTED(Z_TYPE_P(dim) == IS_LONG)) {
1541│                 hval = Z_LVAL_P(dim);
1542│ num_index:
1543├───────────────> ZEND_HASH_INDEX_FIND(ht, hval, retval, num_undef);
1544│                 return retval;
```

我们确认一下`hval`的是不是1：

```shell
(gdb) p	hval
$22 = 1
(gdb) 
```

没问题。

我们继续执行：

```shell
(gdb) n
(gdb) 
```

```shell
1542│ num_index:
1543│                 ZEND_HASH_INDEX_FIND(ht, hval, retval, num_undef);
1544│                 return retval;
1545│ num_undef:
1546├───────────────> switch (type) {
```

显然，数组`$a`索引为1的元素是不存在的：

```shell
(gdb) p	type
$23 = 1
(gdb) 
```

这个type为1，说明zval的类型是`IS_NULL`：

```c
#define IS_NULL						1
```

我们继续执行：

```shell
(gdb) n
(gdb) n
(gdb) n
(gdb) n
(gdb) n
zend_fetch_dimension_address_inner_W_CONST (ht=0x7ffff5e593c0, dim=0x7ffff5e7b110) at /root/php-7.1.0/Zend/zend_execute.c:1652
(gdb) n
ZEND_ASSIGN_DIM_SPEC_CV_CONST_OP_DATA_CONST_HANDLER () at /root/php-7.1.0/Zend/zend_vm_execute.h:39096
(gdb) n
(gdb) n
(gdb) n
```

```c
39100│                 value = EX_CONSTANT((opline+1)->op1);
39101├───────────────> value = zend_assign_to_variable(variable_ptr, value, IS_CONST);
```

我们打印一下`value`：

```shell
(gdb) p	value
$24 = (zval *) 0x7ffff5e7b120
(gdb) 
```

这是一个zval类型，我们看看里面的内容：

```shell
(gdb) p	*value
$25 = {value = {lval = 140737318491520, dval = 6.9533474154478039e-310, counted = 0x7ffff5e01580, str =	0x7ffff5e01580, arr = 0x7ffff5e01580, obj = 0x7ffff5e01580, res = 0x7ffff5e01580, ref =	0x7ffff5e015
80, ast = 0x7ffff5e01580, zv = 0x7ffff5e01580, ptr = 0x7ffff5e01580, ce = 0x7ffff5e01580, func = 0x7ffff5e01580, ww = {w1 = 4125103488, w2 = 32767}}, u1 = {v = {type = 6 '\006', type_flags = 0 '\000', con
st_flags = 0 '\000', reserved = 0 '\000'}, type_info = 6}, u2 = {next = 4294967295, cache_slot = 4294967295, lineno = 4294967295, num_args = 4294967295, fe_pos = 4294967295, fe_iter_idx = 4294967295, acce
ss_flags = 4294967295, property_guard = 4294967295}}
(gdb) 
```

因为type是一个6，所以是字符串类型。我们打印一下`zend_string`里面的内容：

```shell
(gdb) p *value.value.str
$27 = {gc = {refcount = 0, u = {v = {type = 6 '\006', flags = 2 '\002', gc_info = 0}, type_info = 518}}, h = 9223372036854953478, len = 1, val = "a"}
(gdb) 
```

发现字符串的len是1，所以我们可以查看一下字符串的内容：

```shell
(gdb) p *value.value.str.val@1
$28 = "a"
(gdb) 
```

发现它是字符`'a'`。与我们脚本第5行的一致。

可想而知，接下来39101行`zend_assign_to_variable`的操作字符串的分配操作了。

我们继续执行：

```shell
(gdb) n
(gdb) 
```

```c
39102├───────────────> if (UNEXPECTED(RETURN_VALUE_USED(opline))) {
39103│                         ZVAL_COPY(EX_VAR(opline->result.var), value);
39104│                 }
```

我们打印一下数组`$a`对应的`zend_array`：

```shell
(gdb) p *(zend_array *)0x7ffff5e593c0
$31 = {gc = {refcount = 1, u = {v = {type = 7 '\a', flags = 0 '\000', gc_info = 0}, type_info = 7}}, u = {v = {flags = 30 '\036', nApplyCount = 0 '\000', nIteratorsCount = 0 '\000', consistency = 0 '\000'
}, flags = 30}, nTableMask = 4294967294, arData = 0x7ffff5e5fa08, nNumUsed = 2, nNumOfElements = 1, nTableSize = 8, nInternalPointer = 1, nNextFreeElement = 2, pDestructor = 0x84d18f <_zval_ptr_dtor_wrapp
er>}
(gdb) 
```

此时，`nNumUsed`的值为2了。

此时，`arData`里面是有内容的了，我们打印一下：

```shell
(gdb) p	$31.arData[1]
$32 = {val = {value = {lval = 140737318491520, dval = 6.9533474154478039e-310, counted = 0x7ffff5e01580, str = 0x7ffff5e01580, arr = 0x7ffff5e01580, obj = 0x7ffff5e01580, res = 0x7ffff5e01580, ref = 0x7ff
ff5e01580, ast = 0x7ffff5e01580, zv = 0x7ffff5e01580, ptr = 0x7ffff5e01580, ce = 0x7ffff5e01580, func = 0x7ffff5e01580, ww = {w1 = 4125103488, w2 = 32767}}, u1 = {v = {type = 6 '\006', type_flags = 0 '\00
0', const_flags = 0 '\000', reserved = 0 '\000'}, type_info = 6}, u2 = {next = 32767, cache_slot = 32767, lineno = 32767, num_args = 32767, fe_pos = 32767, fe_iter_idx = 32767, access_flags = 32767, prope
rty_guard = 32767}}, h = 1, key = 0x0}
(gdb) 
```

这是一个`Bucket`类型，里面直接嵌入了一个zval，这个zval的type为6，是一个字符串，所以，我们打印一下这个`zend_string`：

```shell
(gdb) p *$31.arData[1].val.value.str
$36 = {gc = {refcount = 0, u = {v = {type = 6 '\006', flags = 2 '\002', gc_info = 0}, type_info = 518}}, h = 9223372036854953478, len = 1, val = "a"}
(gdb) 
```

字符串的长度是1，我们继续打印字符串的内容：

```shell
(gdb) p	*$31.arData[1].val.value.str.val@1
$37 = "a"
(gdb) 
```

至此，调试完了PHP脚本的第5行。

（未完）



















