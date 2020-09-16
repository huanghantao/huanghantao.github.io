---
title: 《就是要你懂Swoole》--为何Swoole4.4版本打算把lock模块给移除
date: 2019-05-29 18:38:29
tags:
- PHP
- Swoole
---

昨天，有人给`Swoole`提了一个问题：[What is the reason to deprecate `Lock` in swoole 4.4 ?](https://github.com/swoole/swoole-src/issues/2601)

然后，峰哥给出了一个例子：

```php
<?php
$lock = new Swoole\Lock();
$c = 2;

while ($c--) {
    go(function () use ($lock) {
        $lock->lock();
        Co::sleep(1);
        $lock->unlock();
    });
}
```

我们来看看执行这段代码会有什么影响。（使用`4.3.4`版本的`Swoole`）

```shell
~/codeDir/phpCode/test # php test.php 

```

我们会发现：进程一直卡着不退出。实际上，这是死锁导致的，而且是由于同一个线程里面的两个协程去竞争锁导致的。

我们来调试一下：

```
~/codeDir/phpCode/test # cgdb php
GNU gdb (GDB) 8.2
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
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

然后在创建锁以及上锁的地方打断点：

```
(gdb) b zim_swoole_lock___construct
Breakpoint 1 at 0x7ffff74d3ea0: file /root/codeDir/cCode/swoole-src-4.3.4/swoole_lock.c, line 88.
(gdb) b zim_swoole_lock_lock
Breakpoint 2 at 0x7ffff74d3ac0: file /usr/local/include/php/Zend/zend_types.h, line 411.
(gdb) 
```

我们执行一下程序：

```
(gdb) r test.php
The program being debugged has been started already.
Start it from the beginning? (y or n) y
The program being debugged has been started already.
Starting program: /usr/local/bin/php test.php

Breakpoint 1, zim_swoole_lock___construct (execute_data=0x7ffff761d110, return_value=0x7fffffffb0b0) at /root/codeDir/cCode/swoole-src-4.3.4/swoole_lock.c:88
warning: Source file is more recent than executable.
(gdb) 
```

```c++
 87│ static PHP_METHOD(swoole_lock, __construct)
 88├>{
 89│     long type = SW_MUTEX;
 90│     char *filelock;
 91│     size_t filelock_len = 0;
 92│     int ret;
 93│
 94│     if (zend_parse_parameters(ZEND_NUM_ARGS(), "|ls", &type, &filelock, &filelock_len) == FAILURE)
 95│     {
 96│         RETURN_FALSE;
 97│     }
```

在创建`Swoole\Lock`的地方触发了断点。

我们继续运行：

```
(gdb) c
Continuing.

Breakpoint 2, zim_swoole_lock_lock (execute_data=0x7ffff767d130, return_value=0x7ffff7040f30) at /usr/local/include/php/Zend/zend_types.h:411
(gdb) n
(gdb) 
```

```c++
162│ static PHP_METHOD(swoole_lock, lock)
163│ {
164│     swLock *lock = swoole_get_object(getThis());
165├───> SW_LOCK_CHECK_RETURN(lock->lock(lock));
166│ }
```

协程1停在了即将获取锁的地方。我们来看看此时的线程`id`是多少：

```
(gdb) thread
[Current thread is 1 (process 39810)]
(gdb) 
```

OK，此时协程1在线程1里面。我们在协程`sleep`的地方打一个断点：

```
(gdb) b swoole::Coroutine::sleep
Breakpoint 3 at 0x7ffff74579d0 (2 locations)
(gdb) 
```

我们继续执行：

```
(gdb) c
Continuing.

Breakpoint 3, 0x00007ffff74579d0 in swoole::Coroutine::sleep(double)@plt () from /usr/local/lib/php/extensions/no-debug-non-zts-20180731/swoole.so
(gdb) n
Single stepping until exit from function _ZN6swoole9Coroutine5sleepEd@plt,
which has no line number information.

Breakpoint 3, swoole::Coroutine::sleep (sec=1) at /root/codeDir/cCode/swoole-src-4.3.4/include/coroutine.h:145
(gdb) n
(gdb) 
```

```c++
 652│ int Coroutine::sleep(double sec)
 653│ {
 654│     Coroutine* co = Coroutine::get_current_safe();
 655├───> if (swTimer_add(&SwooleG.timer, (long) (sec * 1000), 0, co, sleep_timeout) == NULL)
 656│     {
 657│         return -1;
 658│     }
 659│     co->yield();
 660│     return 0;
 661│ }
```

此时，协程1即将被`yield`出去。我们再看看所处的线程：

```
(gdb) thread
[Current thread is 1 (process 39810)]
(gdb) 
```

此时协程1还是处于线程1里面。

我们继续执行：

```
(gdb) c
Continuing.

Breakpoint 2, zim_swoole_lock_lock (execute_data=0x7ffff767f130, return_value=0x7ffff6e3ff30) at /usr/local/include/php/Zend/zend_types.h:411
(gdb) n
(gdb) 
```

此时，已经切换到了协程2，轮到第2个协程获取锁：

```c++
162│ static PHP_METHOD(swoole_lock, lock)
163│ {
164│     swLock *lock = swoole_get_object(getThis());
165├───> SW_LOCK_CHECK_RETURN(lock->lock(lock));
166│ }
```

我们看看协程2现在所处的线程：

```
(gdb) thread
[Current thread is 1 (process 39810)]
(gdb) 
```

发现，协程2也处于线程1里面。

我们继续执行：

```
(gdb) n

```

```c++
162│ static PHP_METHOD(swoole_lock, lock)
163│ {
164│     swLock *lock = swoole_get_object(getThis());
165├───> SW_LOCK_CHECK_RETURN(lock->lock(lock));
166│ }
```

发现，此时协程2卡住。整个进程/线程也卡住了。

原因如下：

协程1一直没有释放锁，本质上会导致线程1一直占有锁。此时协程2去获取锁，本质上是线程1再去获取锁，是获取不到的。因为锁没有被释放掉，从而导致线程1挂起。导致线程1一直无法释放锁，从而造成了死锁。从而，所有的协程都无法继续执行任务。

那么，现如今的Swoole有没有避免死锁的办法呢？其实是可以的，代码如下：

```php
<?php

$lock = new Swoole\Lock();
$c = 2;

while ($c--) {
    go(function () use ($lock) {
        $lock->trylock();
        Co::sleep(1);
        $lock->unlock();
    });
}
```

只要不让线程阻塞起来就可以了。

但是，这样做终究是不好的，会浪费CPU。因为如果协程1需要一段时间释放锁，那么协程2每次执行的时候，都会做无用的`trylock`工作。

这个问题归根结底是`Swoole`的协程是跑在同一个线程里面的。

我个人认为，`Swoole`后期会增加协程锁。当协程获取不到锁的时候，不是去阻塞整个线程，而是`yield`这个协程。因为`Swoole`现在支持抢占式的协程调度，所以，协程的竞争问题是肯定要解决的。除非我们编写的代码不会导致协程之间竞争共享资源。













