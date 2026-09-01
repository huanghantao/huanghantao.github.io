---
title: PHP内核中与异常处理有关的结构体
date: 2021-03-03 11:59:07
tags:
- PHP
- PHP内核
---

> 本文基于PHP8.0.1

测试脚本如下：

```php
<?php

function test(array $arr)
{
}

try {
    test(10000);
} catch (\TypeError $e) {
    echo "handle error\n";
}
```

对应的`opcodes`如下：

```bash
[Stack in /root/codeDir/phpCode/test/test.php (7 ops)]
L1-12 {main}() /root/codeDir/phpCode/test/test.php - 0x7f61ca65e3c0 + 7 ops
 L8    #0     INIT_FCALL<1>           96                   "test"
 L8    #1     SEND_VAL                10000                1
 L8    #2     DO_FCALL
 L8    #3     JMP                     J6
 L9    #4     CATCH<9>                "TypeError"                               $e
 L10   #5     ECHO                    "handle error\n"
 L12   #6     RETURN<-1>              1

[User Function test (2 ops)]
L3-5 test() /root/codeDir/phpCode/test/test.php - 0x7f06cee66000 + 2 ops
 L3    #0     RECV                    1                                         $arr
 L5    #1     RETURN<-1>              null
```

执行结果如下：

```bash
[root@97043d024896 test]# php test.php
handle error
```

我们来梳理一下流程：

```bash
graph-easy <<< "
graph { flow: down; }
[zend_startup] -> 
[zend_init_exception_op] -> 
[ZEND_RECV_SPEC_UNUSED_HANDLER] -> 
[zend_verify_recv_arg_type_helper_SPEC] -> 
[zend_throw_error] -> 
[zend_throw_exception] -> 
[zend_throw_exception_internal] -> 
[ZEND_HANDLE_EXCEPTION_SPEC_HANDLER]
"

+---------------------------------------+
|             zend_startup              |
+---------------------------------------+
                    |
                    |
                    v
+---------------------------------------+
|        zend_init_exception_op         |
+---------------------------------------+
                    |
                    |
                    v
+---------------------------------------+
|     ZEND_RECV_SPEC_UNUSED_HANDLER     |
+---------------------------------------+
                    |
                    |
                    v
+---------------------------------------+
| zend_verify_recv_arg_type_helper_SPEC |
+---------------------------------------+
                    |
                    |
                    v
+---------------------------------------+
|           zend_throw_error            |
+---------------------------------------+
                    |
                    |
                    v
+---------------------------------------+
|         zend_throw_exception          |
+---------------------------------------+
                    |
                    |
                    v
+---------------------------------------+
|     zend_throw_exception_internal     |
+---------------------------------------+
                    |
                    |
                    v
+---------------------------------------+
|  ZEND_HANDLE_EXCEPTION_SPEC_HANDLER   |
+---------------------------------------+
```

首先，在`zend_startup`阶段，初始化`PHP`的异常处理`opline`：

```cpp
static void zend_init_exception_op(void)
{
    memset(EG(exception_op), 0, sizeof(EG(exception_op)));
    EG(exception_op)[0].opcode = ZEND_HANDLE_EXCEPTION;
    ZEND_VM_SET_OPCODE_HANDLER(EG(exception_op));
    EG(exception_op)[1].opcode = ZEND_HANDLE_EXCEPTION;
    ZEND_VM_SET_OPCODE_HANDLER(EG(exception_op)+1);
    EG(exception_op)[2].opcode = ZEND_HANDLE_EXCEPTION;
    ZEND_VM_SET_OPCODE_HANDLER(EG(exception_op)+2);
}
```

这里，我们可以看到，`EG(exception_op)`是一个包含`3`条`opline`的数组。并且，把`3`条`opline`的`opcode`都设置为了`ZEND_HANDLE_EXCEPTION`。

接着，`php`解释器开始执行我们的测试脚本。当`test`函数接收参数的时候，调用`zend_verify_recv_arg_type_helper_SPEC`发现参数不对：

```cpp
static zend_never_inline ZEND_COLD ZEND_OPCODE_HANDLER_RET ZEND_FASTCALL zend_verify_recv_arg_type_helper_SPEC(zval *op_1 ZEND_OPCODE_HANDLER_ARGS_DC)
{
    USE_OPLINE

    SAVE_OPLINE();
    if (UNEXPECTED(!zend_verify_recv_arg_type(EX(func), opline->op1.num, op_1, CACHE_ADDR(opline->extended_value)))) {
        HANDLE_EXCEPTION();
    }

    ZEND_VM_NEXT_OPCODE();
}
```

就会调用`zend_throw_error`来抛出一个`error`级别的异常。

最终，调用`zend_throw_exception_internal`函数：

```cpp
ZEND_API ZEND_COLD void zend_throw_exception_internal(zend_object *exception) /* {{{ */
{
    // 省略其他代码

    if (zend_throw_exception_hook) {
        zend_throw_exception_hook(exception);
    }

    if (is_handle_exception_set()) {
        /* no need to rethrow the exception */
        return;
    }
    EG(opline_before_exception) = EG(current_execute_data)->opline;
    EG(current_execute_data)->opline = EG(exception_op);
}
```

我们发现，这里设置了`EG(current_execute_data)->opline`为`EG(exception_op)`。

接着，就会执行`zend_verify_recv_arg_type_helper_SPEC`里面的`HANDLE_EXCEPTION()`：

```cpp
#define HANDLE_EXCEPTION() ZEND_ASSERT(EG(exception)); LOAD_OPLINE(); ZEND_VM_CONTINUE()
```

所以，下一条`opline`就变成了去执行`ZEND_HANDLE_EXCEPTION`对应的`ZEND_HANDLE_EXCEPTION_SPEC_HANDLER`了。而这个`handler`就是用来处理异常的，例如查找当前作用域上是否有对异常抛出点进行`try ... catch ... finally`。

这个`handler`我们不去细讲，我们主要讲一讲和异常处理有关的结构体，搞明白了结构体里面各各属性的作用，就知道这个`handler`做了些什么事情了。

## 与异常有关的数据结构

`zend_object *zend_executor_globals::exception`，保留当前异常的信息，例如`message`、`file`、`lineno`等。

`zend_op *zend_executor_globals::opline_before_exception`，用来回溯异常抛出时候的`opline`。例如，我们有如下异常回溯关系：

```bash
graph-easy <<< "
graph { flow: up; }
[ZEND_RECV] -> 
[DO_FCALL]
"

+-----------+
| DO_FCALL  |
+-----------+
    ^
    |
    |
+-----------+
| ZEND_RECV |
+-----------+
```

那么，`opline_before_exception`依次是`ZEND_RECV`和`DO_FCALL`对应的`opline`。

那么，有如下计算：

```cpp
uint32_t throw_op_num = EG(opline_before_exception) - EX(func)->op_array.opcodes
```

`throw_op_num`则是抛异常的`opline`的索引。

```cpp
typedef struct _zend_try_catch_element {
    uint32_t try_op;
    uint32_t catch_op;
    uint32_t finally_op;
    uint32_t finally_end;
} zend_try_catch_element;
```

这几个字段什么意思呢？假设我们有如下代码：

```php
<?php

function test(array $arr)
{
}

try {
    test(10000);
} catch (\TypeError $e) {
    echo "handle error\n";
} finally {
    echo "finally\n";
}
```

`main`函数对应的`opcodes`如下：

```bash
[Stack in /Users/codinghuang/.phpbrew/build/php-8.0.1/test.php (11 ops)]
L1-14 {main}() /Users/codinghuang/.phpbrew/build/php-8.0.1/test.php - 0x108a066f0 + 11 ops
 L8    #0     INIT_FCALL<1>           96                   "test"
 L8    #1     SEND_VAL                10000                1
 L8    #2     DO_FCALL
 L8    #3     JMP                     J6
 L9    #4     CATCH<9>                "TypeError"                               $e
 L10   #5     ECHO                    "handle error\n"
 L11   #6     FAST_CALL               J8                                        ~0
 L11   #7     JMP                     J10
 L12   #8     ECHO                    "finally\n"
 L12   #9     FAST_RET                ~0
 L14   #10    RETURN<-1>              1
```

那么`zend_try_catch_element`里面的内容如下：

```cpp
try_op -> 0;
catch_op -> 4;
finally_op -> 8;
finally_end -> 9;
```

如果`try`后面没有跟`catch`，那么，`catch_op`为`0`；如果`try`后面没有跟`finally`，那么`finally_op`为`0`。

所以，`try_op`表示`try`里面的第一条`opline`的索引；`catch_op`表示`ZEND_CATCH`这条`opline`的索引；`finally_op`表示`finally`里面的第一条`opline`的索引；`finally_end`表示`ZEND_FAST_RET`这条`opline`的索引。

我们发现，`try_op`和`finally_op`都是表示它们里面的`opline`的位置，而`catch_op`却是表示`catch`这条`opline`本身的位置。这是因为我们不能给`try`和`finally`传参，但是可以给`catch`传参。例如，不能这么写：

```php
try (something) {
    test(10000);
} catch (\TypeError $e) {
    echo "handle error\n";
} finally (something) {
    echo "finally\n";
}
```

所以，我们发现，这里只有`catch`生成了`ZEND_CATCH`，但是却没有`ZEND_TRY`和`ZEND_FINALLY`这样的`opline`。

`zend_op_array::last_try_catch`表示当前作用域有几组`try ... catch ... finally`。例如：

```php
<?php

function test(array $arr)
{
}

try {
    throw new Exception("Error Processing Request", 1);
} catch (\Exception $e) {
    echo "handle error\n";
}

try {
    test(10000);
} catch (\TypeError $e) {
    echo "handle error\n";
} finally {
    echo "finally\n";
}
```

因为`main`作用域有两组`try ... catch`，所以`zend_op_array::last_try_catch`是`2`。

有了上面的基础之后，那么`ZEND_HANDLE_EXCEPTION_SPEC_HANDLER`里面查找`zend_try_catch_element`的流程就好理解了：

```cpp
const zend_op *throw_op = EG(opline_before_exception);
uint32_t throw_op_num = throw_op - EX(func)->op_array.opcodes;
int i, current_try_catch_offset = -1;

// 省略其他代码

/* Find the innermost try/catch/finally the exception was thrown in */
for (i = 0; i < EX(func)->op_array.last_try_catch; i++) {
    zend_try_catch_element *try_catch = &EX(func)->op_array.try_catch_array[i];
    if (try_catch->try_op > throw_op_num) {
        /* further blocks will not be relevant... */
        break;
    }
    if (throw_op_num < try_catch->catch_op || throw_op_num < try_catch->finally_end) {
        current_try_catch_offset = i;
    }
}
```

这段代码就是通过异常抛出的`opline`来找到对应的`zend_try_catch_element`。

> 因此，异常处理的核心就是，通过改变`EG(current_execute_data)->opline`，达到执行`ZEND_HANDLE_EXCEPTION_SPEC_HANDLER`的目的。
