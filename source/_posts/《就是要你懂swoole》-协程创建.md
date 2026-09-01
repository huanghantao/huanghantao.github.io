---
title: 《就是要你懂swoole》--协程创建
date: 2019-05-04 17:56:35
tags:
- swoole
- PHP
---

这篇文章我们来调试一下swoole的协程创建过程。

PHP版本是`7.2.14`，Swoole的版本是`4.4.0-alpha`。

我们测试的代码：

```php
<?php
// file: test.php

go(function(){
    echo "co1\n";
});

go(function(){
    echo "co2\n";
});
```

需要打断点地方有：

```c
1、zif_swoole_coroutine_create
2、swoole::PHPCoroutine::create
3、swoole::Coroutine::create
```

OK，开始调试：

```shell
cgdb php
GNU gdb (GDB) 8.0.1
Copyright (C) 2017 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-alpine-linux-musl".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from php...(no debugging symbols found)...done.
(gdb) 
```

分别在上面的两个函数打断点：

```shell
(gdb) b zif_swoole_coroutine_create
Function "zif_swoole_coroutine_create" not defined.
Make breakpoint pending on future shared library load? (y or [n]) y
Breakpoint 1 (zif_swoole_coroutine_create) pending.
(gdb) b swoole::PHPCoroutine::create
Function "swoole::PHPCoroutine::create" not defined.
Make breakpoint pending on future shared library load? (y or [n]) y
Breakpoint 2 (swoole::PHPCoroutine::create) pending.
(gdb) b swoole::Coroutine::create
Function "swoole::Coroutine::create" not defined.
Make breakpoint pending on future shared library load? (y or [n]) y
Breakpoint 3 (swoole::Coroutine::create) pending.
```

然后运行测试脚本：

```shell
(gdb) r test.php 
Starting program: /usr/local/bin/php test.php

Breakpoint 1, zif_swoole_coroutine_create (execute_data=0x7ffff681c0d0, return_value=0x7fffffffb090) at /root/codeDir/cCode/swoole-src/swoole_coroutine_util.cc:416
warning: Source file is more recent than executable.
(gdb) 
```

```c
415│ PHP_FUNCTION(swoole_coroutine_create)
 416├>{
 417│     zend_fcall_info fci = empty_fcall_info;
 418│     zend_fcall_info_cache fci_cache = empty_fcall_info_cache;
 419│
 420│     ZEND_PARSE_PARAMETERS_START(1, -1)
 421│         Z_PARAM_FUNC(fci, fci_cache)
 422│         Z_PARAM_VARIADIC('*', fci.params, fci.param_count)
 423│     ZEND_PARSE_PARAMETERS_END_EX(RETURN_FALSE);
```

接下来我们会看到`zend_fcall_info`和`zend_fcall_info_cache`，这两个结构体用来保存协程函数的信息（也就是`go()`这个函数里面的那个函数）。而`empty_fcall_info`也是`zend_fcall_info`类型的变量，它是只读的，并且里面的内容都为空：

```shell
(gdb) p &empty_fcall_info
$1 = (const zend_fcall_info *) 0x5555561d42e0 <empty_fcall_info>
(gdb) p empty_fcall_info
$2 = {size = 0, function_name = {value = {lval = 0, dval = 0, counted = 0x0, str = 0x0, arr = 0x0, obj = 0x0, res = 0x0, ref = 0x0, ast = 0x0, zv = 0x0, ptr = 0x0, ce = 0x0, func = 0x0, ww = {w1 = 0, w2 =
 0}}, u1 = {v = {type = 0 '\000', type_flags = 0 '\000', const_flags = 0 '\000', reserved = 0 '\000'}, type_info = 0}, u2 = {next = 0, cache_slot = 0, lineno = 0, num_args = 0, fe_pos = 0, fe_iter_idx = 0
, access_flags = 0, property_guard = 0, extra = 0}}, retval = 0x0, params = 0x0, object = 0x0, no_separation = 0 '\000', param_count = 0}
(gdb) 
```

我们继续执行到420行之前：

```shell
(gdb) u 420
zif_swoole_coroutine_create (execute_data=0x7ffff681c0d0, return_value=0x7fffffffb090) at /root/codeDir/cCode/swoole-src/swoole_coroutine_util.cc:420
(gdb) 
```

```c
 420├───> ZEND_PARSE_PARAMETERS_START(1, -1)
 421│         Z_PARAM_FUNC(fci, fci_cache)
 422│         Z_PARAM_VARIADIC('*', fci.params, fci.param_count)
 423│     ZEND_PARSE_PARAMETERS_END_EX(RETURN_FALSE);
```

这段代码是用来获取协程函数然后保存它的信息到`fci`和`fci_cache`两个变量里面。我们执行完这3行代码：

```shell
(gdb) u 425
zif_swoole_coroutine_create (execute_data=0x7ffff681c0d0, return_value=0x7fffffffb090) at /root/codeDir/cCode/swoole-src/swoole_coroutine_util.cc:425
(gdb) 
```

然后打印一下里面的内容：

```shell
(gdb) p fci
$3 = {size = 56, function_name = {value = {lval = 140737329383616, dval = 6.9533479535888483e-310, counted = 0x7ffff68648c0, str = 0x7ffff68648c0, arr = 0x7ffff68648c0, obj = 0x7ffff68648c0, res = 0x7ffff
68648c0, ref = 0x7ffff68648c0, ast = 0x7ffff68648c0, zv = 0x7ffff68648c0, ptr = 0x7ffff68648c0, ce = 0x7ffff68648c0, func = 0x7ffff68648c0, ww = {w1 = 4135995584, w2 = 32767}}, u1 = {v = {type = 8 '\b', t
ype_flags = 4 '\004', const_flags = 0 '\000', reserved = 0 '\000'}, type_info = 1032}, u2 = {next = 0, cache_slot = 0, lineno = 0, num_args = 0, fe_pos = 0, fe_iter_idx = 0, access_flags = 0, property_gua
rd = 0, extra = 0}}, retval = 0x0, params = 0x0, object = 0x0, no_separation = 1 '\001', param_count = 0}
(gdb) 
```

然后函数调用的参数以及参数个数保存了下来，我们查看一下：

```shell
(gdb) p fci.params
$9 = (zval *) 0x0
(gdb) p fci.param_count
$10 = 0
(gdb) 
```

因为我们这里只给`go()`函数传递了一个协程函数，所以参数为空。

OK，我们继续往下走：

```shell
(gdb) c
Continuing.

Breakpoint 2, 0x00007ffff485ad00 in swoole::PHPCoroutine::create(_zend_fcall_info_cache*, unsigned int, _zval_struct*)@plt () from /usr/local/lib/php/extensions/no-debug-non-zts-20170718/swoole.so
(gdb) n
Single stepping until exit from function _ZN6swoole12PHPCoroutine6createEP22_zend_fcall_info_cachejP12_zval_struct@plt,
which has no line number information.

Breakpoint 2, swoole::PHPCoroutine::create (fci_cache=fci_cache@entry=0x7fffffffafa0, argc=0, argv=0x0) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:418
warning: Source file is more recent than executable.
(gdb) 
```

```c
417│ long PHPCoroutine::create(zend_fcall_info_cache *fci_cache, uint32_t argc, zval *argv)
418├>{
419│     if (unlikely(!active))
420│     {
421│         if (zend_hash_str_find_ptr(&module_registry, ZEND_STRL("xdebug")))
422│         {
423│             swoole_php_fatal_error(E_WARNING, "Using Xdebug in coroutines is extremely dangerous, please notice that it may lead to coredump!");
424│         }
425│         php_swoole_check_reactor();
426│         // PHPCoroutine::enable_hook(SW_HOOK_ALL); // TODO: enable it in version 4.3.0
427│         active = true;
428│     }
```

此时就进入了`swoole::PHPCoroutine::create`这个函数。

421行是查找是否安装了`xdebug`这个扩展，因为这个扩展可能会导致`coredump`。

然后我们继续运行到425行之前：

```shell
(gdb) u 425
(gdb) 
```

```c
425├─555555> php_swoole_check_reactor();
426│         // PHPCoroutine::enable_hook(SW_HOOK_ALL); // TODO: enable it in version 4.3.0
427│         active = true;
```

`php_swoole_check_reactor`是用来检测是否开启`reactor`，如果没有开启，那么会在这个函数里面开启`reactor`。我们进入这个函数：

```shell
(gdb) s
php_swoole_check_reactor () at /root/codeDir/cCode/swoole-src/php_swoole.h:368
(gdb) 
```

```c
 366│ static sw_inline void php_swoole_check_reactor()
 367│ {
 368├───> if (unlikely(!SwooleWG.reactor_init))
 369│     {
 370│         php_swoole_reactor_init();
 371│     }
 372│ }
```

我们进入`php_swoole_reactor_init`这个函数看看：

```shell
(gdb) b php_swoole_reactor_init
Breakpoint 3 at 0x7ffff4825eb6: file /root/codeDir/cCode/swoole-src/swoole_event.c, line 190.
(gdb) c
Continuing.

Breakpoint 3, php_swoole_reactor_init () at /root/codeDir/cCode/swoole-src/swoole_event.c:190
(gdb) 
```

```c
190├───> if (!SWOOLE_G(cli))
191│     {
192│         swoole_php_fatal_error(E_ERROR, "async-io must be used in PHP CLI mode");
193│         return;
194│     }
```

190到194是用来判断当前的环境是否是`cli`模式。因为异步IO必须被使用在`cli`模式。

继续往下走：

```shell
(gdb) u 196
php_swoole_reactor_init () at /root/codeDir/cCode/swoole-src/swoole_event.c:196
(gdb) 
```

```c
196├───> if (SwooleG.serv)
197│     {
198│         if (swIsTaskWorker() && !SwooleG.serv->task_enable_coroutine)
199│         {
200│             swoole_php_fatal_error(E_ERROR, "Unable to use async-io in task processes, please set `task_enable_coroutine` to true");
201│             return;
202│         }
203│         if (swIsManager())
204│         {
205│             swoole_php_fatal_error(E_ERROR, "Unable to use async-io in manager process");
206│             return;
207│         }
208│     }
```

196到208行之间的代码在开启了`Swoole`服务器的情况下会执行。在调用了`swServer_init()`的时候`SwooleG.serv`的值会被赋值为全局的那个`serv`，我们可以在`swoole_server.cc`里面的`PHP_METHOD(swoole_server, __construct)`这个构造函数里面找到调用了`swServer_init()`这个函数。

我们继续往下走：

```shell
(gdb) n
(gdb) 
```

```c
210├───> if (SwooleG.main_reactor == NULL)
211│     {
212│         swTraceLog(SW_TRACE_PHP, "init reactor");
213│
214│         SwooleG.main_reactor = (swReactor *) sw_malloc(sizeof(swReactor));
215│         if (SwooleG.main_reactor == NULL)
216│         {
217│             swoole_php_fatal_error(E_ERROR, "malloc failed");
218│             return;
219│         }
220│         if (swReactor_create(SwooleG.main_reactor, SW_REACTOR_MAXEVENTS) < 0)
221│         {
222│             swoole_php_fatal_error(E_ERROR, "failed to create reactor");
223│             return;
224│         }
225│ 
226│         SwooleG.main_reactor->can_exit = php_coroutine_reactor_can_exit;
227│ 
228│         //client, swoole_event_exit will set swoole_running = 0
229│         SwooleWG.in_client = 1;
230│         SwooleWG.reactor_wait_onexit = 1;
231│         SwooleWG.reactor_ready = 0;
232│         //only client side
233│         php_swoole_register_shutdown_function_prepend("swoole_event_wait");
234│     }
```

210行到224行之间的代码是为全局的这个`SwooleG.main_reactor`分配内存以及完成一些基本的初始化操作。226行是设置回调函数`php_coroutine_reactor_can_exit`。这个函数内容如下：

```c++
int php_coroutine_reactor_can_exit(swReactor *reactor)
{
    return Coroutine::count() == 0;
}
```

很容易可以理解，当协程个数为0的时候，`main_reactor`可以退出了。

我们继续执行到236行之前：

```shell
(gdb) u 236
php_swoole_reactor_init () at /root/codeDir/cCode/swoole-src/swoole_event.c:236
(gdb) 
```

```c++
236├───> SwooleG.main_reactor->setHandle(SwooleG.main_reactor, SW_FD_USER | SW_EVENT_READ, php_swoole_event_onRead);
237│     SwooleG.main_reactor->setHandle(SwooleG.main_reactor, SW_FD_USER | SW_EVENT_WRITE, php_swoole_event_onWrite);
238│     SwooleG.main_reactor->setHandle(SwooleG.main_reactor, SW_FD_USER | SW_EVENT_ERROR, php_swoole_event_onError);
239│     SwooleG.main_reactor->setHandle(SwooleG.main_reactor, SW_FD_WRITE, swReactor_onWrite);
240│
241│     SwooleWG.reactor_init = 1;
```

这是设置`main_reactor`的一些事件处理器，用于处理不同状态的事件。最后，赋值`SwooleWG.reactor_init`为1，代表`SwooleG.main_reactor`初始化成功了。

我们继续执行：

```shell
(gdb) finish
Run till exit from #0  php_swoole_reactor_init () at /root/codeDir/cCode/swoole-src/swoole_event.c:236
swoole::PHPCoroutine::create (fci_cache=0x7fffffffafc0, argc=0, argv=0x0) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:427
(gdb) 
```

```c++
427├─777777> active = true;
```

赋值active为true代表协程可以使用了。

OK，继续执行：

```shell
(gdb) n
(gdb) 
```

```c
429├───> if (unlikely(Coroutine::count() >= max_num))
430│     {
431│         swoole_php_fatal_error(E_WARNING, "exceed max number of coroutine %zu", (uintmax_t) Coroutine::count());
432│         return SW_CORO_ERR_LIMIT;
433│     }
```

这个好理解，就是判断当前协程的个数是否已经达到了可以创建的协程最大值，如果超过了，就报PHP的`fetal error`。我们可以看看这个`max_num`默认是多大：

```shell
(gdb) p max_num
$2 = 3000
(gdb) 
```

所以，默认情况是3000。但是，理论上来说，只要你的内存是足够的，就可以创建无数个协程。很显然，此时的协程个数没有超过3000，可以看看：

```shell
(gdb) p Coroutine::count()
$3 = 0
(gdb) 
```

我们继续往下走：

```shell
(gdb) n
(gdb) 
```

```c++
434├───> if (unlikely(!fci_cache || !fci_cache->function_handler))
435│     {
436│         swoole_php_fatal_error(E_ERROR, "invalid function call info cache");
437│         return SW_CORO_ERR_INVALID;
438│     }
```

判断这个协程函数是否有效，如果无效会抛出PHP的`fetal error`。

我们继续往下走：

```shell
(gdb) n
(gdb) 
```

```c++
439├───> zend_uchar type = fci_cache->function_handler->type;
440│     if (unlikely(type != ZEND_USER_FUNCTION && type != ZEND_INTERNAL_FUNCTION))
441│     {
442│         swoole_php_fatal_error(E_ERROR, "invalid function type %u", fci_cache->function_handler->type);
443│         return SW_CORO_ERR_INVALID;
444│     }
```



439到444行代码是用来判断协程函数的类型。其中的`ZEND_USER_FUNCTION`是用户定义的函数，这种函数是用户在PHP脚本中定义的函数，`ZEND_INTERNAL_FUNCTION`是PHP内置的函数，这种函数是由扩展或者Zend/PHP内核提供的。也就是说，`Swoole`协程只支持这两种类型。

我们继续执行：

```shell
(gdb) u 446
```

```c++
446├───> php_coro_args php_coro_args;
447│     php_coro_args.fci_cache = fci_cache;
448│     php_coro_args.argv = argv;
449│     php_coro_args.argc = argc;
```

446行到449行是简单的赋值操作。我们继续执行：

```shell
(gdb) u 450
swoole::PHPCoroutine::create (fci_cache=0x7fffffffafc0, argc=0, argv=0x0) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:450
(gdb) 
```

```c++
450├───> save_task(get_task());
```

其中`get_task`的作用是获取当前正在执行的协程。我们进入这个函数看看：

```shell
(gdb) s
swoole::PHPCoroutine::get_task () at /root/codeDir/cCode/swoole-src/swoole_coroutine.h:113
warning: Source file is more recent than executable.
(gdb) 
```

```c++
111│     static inline php_coro_task* get_task()
112│     {
113├─333333> php_coro_task *task = (php_coro_task *) Coroutine::get_current_task();
114│         return task ? task : &main_task;
115│     }
```

因为当前还没有创建协程，所以` Coroutine::get_current_task`自然就获取不到当前正在运行的协程。我们继续执行，来看看`task`的值：

```shell
(gdb) p task
$5 = (php_coro_task *) 0x0
(gdb) 
```

```c++
114├─444444> return task ? task : &main_task;
```

此时返回`main_task`。实际上它是一个空的`php_coro_task`结构：

```shell
(gdb) p main_task
$6 = {bailout = 0x0, vm_stack_top = 0x0, vm_stack_end = 0x0, vm_stack = 0x0, vm_stack_page_size = 0, execute_data = 0x0, error_handling = EH_NORMAL, exception_class = 0x0, exception = 0x0, output_ptr = 0x
0, co = 0x0, defer_tasks = 0x0, pcid = 0, context = 0x0}
(gdb) 
```

我们继续执行：

```shell
(gdb) finish
Run till exit from #0  swoole::PHPCoroutine::get_task () at /root/codeDir/cCode/swoole-src/swoole_coroutine.h:115
0x00007ffff481721f in swoole::PHPCoroutine::create (fci_cache=0x7fffffffafc0, argc=0, argv=0x0) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:450
Value returned is $7 = (php_coro_task *) 0x7ffff4b82620 <swoole::PHPCoroutine::main_task>
(gdb) 
```

```c++
450├───> save_task(get_task());
451│
452│     return Coroutine::create(create_func, (void*) &php_coro_args);
453│ }
```

450行是用来保存当前的上下文内容到`get_task()`返回的`php_coro_task`结构里面，此时是`main_task`：

```shell
(gdb) s
swoole::PHPCoroutine::save_task (task=0x7ffff4b82620 <swoole::PHPCoroutine::main_task>) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:200
(gdb) 
```

```c++
198│ void PHPCoroutine::save_task(php_coro_task *task)
199│ {
200├───> save_vm_stack(task);
201│     save_og(task);
202│ }
```

`save_vm_stack`的作用是保存`zend`虚拟机的栈相关信息：

```shell
(gdb) s
swoole::PHPCoroutine::save_vm_stack (task=0x7ffff4b82620 <swoole::PHPCoroutine::main_task>) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:135
(gdb) 
```

```c++
132│ inline void PHPCoroutine::save_vm_stack(php_coro_task *task)
133│ {
134│ #ifdef SW_CORO_SWAP_BAILOUT
135├───> task->bailout = EG(bailout);
136│ #endif
137│     task->vm_stack_top = EG(vm_stack_top);
138│     task->vm_stack_end = EG(vm_stack_end);
139│     task->vm_stack = EG(vm_stack);
140│ #if PHP_VERSION_ID >= 70300
141│     task->vm_stack_page_size = EG(vm_stack_page_size);
142│ #endif
143│     task->execute_data = EG(current_execute_data);
144│     task->error_handling = EG(error_handling);
145│     task->exception_class = EG(exception_class);
146│     task->exception = EG(exception);
147│     SW_SAVE_EG_SCOPE(task->scope);
148│ #ifdef SW_CORO_SCHEDULER_TICK
149│     task->ticks_count = EG(ticks_count);
150│ #endif
151│ }
```

我们继续执行：

```shell
(gdb) finish
Run till exit from #0  swoole::PHPCoroutine::save_vm_stack (task=0x7ffff4b82620 <swoole::PHPCoroutine::main_task>) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:135
swoole::PHPCoroutine::save_task (task=0x7ffff4b82620 <swoole::PHPCoroutine::main_task>) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:201
(gdb) 
```

```c++
198│ void PHPCoroutine::save_task(php_coro_task *task)
199│ {
200│     save_vm_stack(task);
201├───> save_og(task);
202│ }
```

`save_og`用来是保存与输出缓存管理相关的信息：

```shell
(gdb) s
swoole::PHPCoroutine::save_og (task=0x7ffff4b82620 <swoole::PHPCoroutine::main_task>) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:176
(gdb) 
```

```c++
174│ inline void PHPCoroutine::save_og(php_coro_task *task)
175│ {
176├───> if (OG(handlers).elements)
177│     {
178│         task->output_ptr = (zend_output_globals *) emalloc(sizeof(zend_output_globals));
179│         memcpy(task->output_ptr, SWOG, sizeof(zend_output_globals));
180│         php_output_activate();
181│     }
182│     else
183│     {
184│         task->output_ptr = NULL;
185│     }
186│ }
```

继续执行：

```shell
(gdb) finish
Run till exit from #0  swoole::PHPCoroutine::save_og (task=0x7ffff4b82620 <swoole::PHPCoroutine::main_task>) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:186
swoole::PHPCoroutine::save_task (task=0x7ffff4b82620 <swoole::PHPCoroutine::main_task>) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:202
(gdb) finish
Run till exit from #0  swoole::PHPCoroutine::save_task (task=0x7ffff4b82620 <swoole::PHPCoroutine::main_task>) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:202
swoole::PHPCoroutine::create (fci_cache=0x7fffffffafc0, argc=0, argv=0x0) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:452
(gdb) 
```

```c++
452├───> return Coroutine::create(create_func, (void*) &php_coro_args);
453│ }
```

然后，我们进入这个函数：

```shell
(gdb) s

Breakpoint 8, 0x00007ffff4767120 in swoole::Coroutine::create(void (*)(void*), void*)@plt () from /usr/local/lib/php/extensions/no-debug-non-zts-20170718/swoole.so
(gdb) n
Single stepping until exit from function _ZN6swoole9Coroutine6createEPFvPvES1_@plt,
which has no line number information.
swoole::Coroutine::create (fn=0x7ffff4817ab6 <swoole::PHPCoroutine::save_task(php_coro_task*)+36>, args=0x7fffffffaea0) at /root/codeDir/cCode/swoole-src/include/coroutine.h:133
(gdb) 
```

```c++
133├───> static inline long create(coroutine_func_t fn, void* args = nullptr)
134│     {
135│         return (new Coroutine(fn, args))->run();
136│     }
```

可以看出，这里创建了协程之后，立马让这个协程运行（这与进程的创建不同，子进程被创建之后，不一定会立马被调度）。

我们现在进入`swoole::Coroutine::Coroutine`这个构造函数：

```shell
(gdb) s

Breakpoint 8, swoole::Coroutine::create (fn=0x7ffff481633e <swoole::PHPCoroutine::create_func(void*)>, args=0x7fffffffaf00) at /root/codeDir/cCode/swoole-src/include/coroutine.h:135
(gdb) 
swoole::Coroutine::Coroutine (this=0x5555566de6c0, fn=0x7ffff481633e <swoole::PHPCoroutine::create_func(void*)>, private_data=0x7fffffffaf00) at /root/codeDir/cCode/swoole-src/include/coroutine.h:215
(gdb) 
```

```c++
214│     Coroutine(coroutine_func_t fn, void *private_data) :
215├─555555────> ctx(stack_size, fn, private_data)
216│     {
217│         cid = ++last_cid;
218│         coroutines[cid] = this;
219│         if (unlikely(count() > peak_num))
220│         {
221│             peak_num = count();
222│         }
223│     }
```

可以看出，这里会得到这个被创建的协程的id，是在最后一个协程id的基础上进行加一的。我们来看看最开始的情况`last_cid`的值：

```shell
(gdb) p last_cid
$11 = 0
(gdb) 
```

所以，我们Swoole创建的第一个协程的id是1，然后保存`this`指针到对应的`coroutines`里面，`coroutines`是一个无序的`map`，定义如下：

```c++
static std::unordered_map<long, Coroutine*> coroutines;
```

继续运行：

```shell
(gdb) finish
Run till exit from #0  swoole::Coroutine::Coroutine (this=0x5555566de6c0, fn=0x7ffff481633e <swoole::PHPCoroutine::create_func(void*)>, private_data=0x7fffffffaf00) at /root/codeDir/cCode/swoole-src/inclu
de/coroutine.h:215
0x00007ffff481752e in swoole::Coroutine::create (fn=0x7ffff481633e <swoole::PHPCoroutine::create_func(void*)>, args=0x7fffffffaf00) at /root/codeDir/cCode/swoole-src/include/coroutine.h:135
(gdb) 
```

```c++
133│     static inline long create(coroutine_func_t fn, void* args = nullptr)
134│     {
135├─555555> return (new Coroutine(fn, args))->run();
136│     }
```

继续运行进入`swoole::Coroutine::run()`：

```shell
(gdb) s
swoole::Coroutine::run (this=0x5555566de6c0) at /root/codeDir/cCode/swoole-src/include/coroutine.h:227
(gdb) 
```

```c++
225│     inline long run()
226│     {
227├─555555> long cid = this->cid;
228│         origin = current;
229│         current = this;
230│         ctx.swap_in();
231│         if (ctx.end)
232│         {
233│             close();
234│         }
235│         return cid;
236│     }
```

把当前的上下文换出去之后，目前的`current`就是上一个协程了；而此时需要调度的协程则是接下来需要被调度的协程了，所以`current`替换为`this`。	

230行是实现`origin`与`current`的切换，载入`current`的上下文。我们进入这个函数：

```shell
(gdb) s
swoole::Context::swap_in (this=0x5555566de6d8) at /root/codeDir/cCode/swoole-src/src/coroutine/context.cc:112
(gdb) 
```

```c++
110│ bool Context::swap_in()
111│ {
112├───> jump_fcontext(&swap_ctx_, ctx_, (intptr_t) this, true);
113│     return true;
114│ }
```

112行这个函数式用汇编协程的，是`boost`的一个库：

```shell
(gdb) s
jump_fcontext () at /root/codeDir/cCode/swoole-src/thirdparty/boost/asm/jump_x86_64_sysv_elf_gas.S:39
(gdb) 
```

```assembly
34│ .text
35│ .globl jump_fcontext
36│ .type jump_fcontext,@function
37│ .align 16
38│ jump_fcontext:
39├───> pushq  %rbp  /* save RBP */
40│     pushq  %rbx  /* save RBX */
41│     pushq  %r15  /* save R15 */
42│     pushq  %r14  /* save R14 */
43│     pushq  %r13  /* save R13 */
44│     pushq  %r12  /* save R12 */
```

（只展示了一部分汇编代码）

它会跳到一个协程入口函数`swoole::Context::context_func(void*)`，然后这个协程入口函数会调用协程函数，我们给这个函数打一个断点，然后继续执行：

```shell
(gdb) b swoole::Context::context_func(void*)
Breakpoint 9 at 0x7ffff477e6d6: file /root/codeDir/cCode/swoole-src/src/coroutine/context.cc, line 124.
(gdb) c 
Continuing.

Breakpoint 9, swoole::Context::context_func (arg=0x5555566de6d8) at /root/codeDir/cCode/swoole-src/src/coroutine/context.cc:124
(gdb) 
```

```c++
122│ void Context::context_func(void *arg)
123│ {
124├───> Context *_this = (Context *) arg;
125│     _this->fn_(_this->private_data_);
126│     _this->end = true;
127│     _this->swap_out();
128│ }
```

这个就是协程函数的入口函数。125行就是去调用协程函数。调用完之后，会在屏幕打印出：

```shell
co1
```

继续执行：

```shell
(gdb) n
(gdb) n
co1
(gdb) 
```

```c++
122│ void Context::context_func(void *arg)
123│ {
124│     Context *_this = (Context *) arg;
125│     _this->fn_(_this->private_data_);
126├───> _this->end = true;
127│     _this->swap_out();
128│ }
```

此时，协程1跑完了，所以设置`end`标志为`true`。然后切换上下文。

`ctx.end`是用来判断协程函数是执行完毕的标志。

继续运行：

```shell
(gdb) finish
Run till exit from #0  swoole::Coroutine::run (this=0x5555566de6c0) at /root/codeDir/cCode/swoole-src/include/coroutine.h:236
0x00007ffff4817536 in swoole::Coroutine::create (fn=0x7ffff481633e <swoole::PHPCoroutine::create_func(void*)>, args=0x7fffffffaf00) at /root/codeDir/cCode/swoole-src/include/coroutine.h:135
Value returned is $18 = 1
(gdb) 
```

```c++
133│     static inline long create(coroutine_func_t fn, void* args = nullptr)
134│     {
135├─555555> return (new Coroutine(fn, args))->run();
136│     }
```

```shell
(gdb) finish
Run till exit from #0  swoole::Coroutine::create (fn=0x7ffff481633e <swoole::PHPCoroutine::create_func(void*)>, args=0x7fffffffaf00) at /root/codeDir/cCode/swoole-src/include/coroutine.h:136
swoole::PHPCoroutine::create (fci_cache=0x7fffffffafc0, argc=0, argv=0x0) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:453
Value returned is $19 = 1
(gdb) 
```

```c++
452│     return Coroutine::create(create_func, (void*) &php_coro_args);
453├>}
```

```shell
(gdb) finish
Run till exit from #0  swoole::PHPCoroutine::create (fci_cache=0x7fffffffafc0, argc=0, argv=0x0) at /root/codeDir/cCode/swoole-src/swoole_coroutine.cc:453
0x00007ffff481d033 in zif_swoole_coroutine_create (execute_data=0x7ffff681c0c0, return_value=0x7fffffffb090) at /root/codeDir/cCode/swoole-src/swoole_coroutine_util.cc:435
Value returned is $20 = 1
(gdb) 
```

```c++
 435├───> long cid = PHPCoroutine::create(&fci_cache, fci.param_count, fci.params);
 436│     if (likely(cid > 0))
 437│     {
 438│         RETURN_LONG(cid);
 439│     }
 440│     else
 441│     {
 442│         RETURN_FALSE;
 443│     }
 444│ }
```

​	此时，可以获取到被创建的协程的id：

```shell
(gdb) n
(gdb) p cid
$21 = 1
(gdb) 
```

然后，`go()`函数返回协程的id。

到此，`Swoole`协程的创建分析完毕。