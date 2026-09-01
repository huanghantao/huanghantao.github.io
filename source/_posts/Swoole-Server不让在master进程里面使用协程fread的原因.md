---
title: Swoole Server不让在master进程里面使用协程fread的原因
date: 2020-04-14 09:49:07
tags:
- Swoole
- PHP
---

如果大家要和我一起做实验的话，可以切换`Swoole`的`commit`：

```bash
git checkout ec23323b1ec4effc3df764eed6fdf39c13b18dda
```

（记得重新编译、安装扩展）

我们现在来写一段`PHP`脚本：

```php
<?php

use Swoole\Coroutine;
use Swoole\Coroutine\System;
use Swoole\Server;

$serv = new Server('127.0.0.1', 9501);

Coroutine::create(function () {
    $fp = fopen(__FILE__, 'r');
    System::fread($fp);
});

$serv->on('Receive', function () {
});

$serv->start();
```

执行结果如下：

```bash
[root@2121d596e844 server]# php test.php
PHP Fatal error:  Uncaught Swoole\Error: can not create server after using async file operation in /root/codeDir/phpCode/swoole/server/test.php:17
Stack trace:
#0 /root/codeDir/phpCode/swoole/server/test.php(17): Swoole\Server->start()
#1 {main}
  thrown in /root/codeDir/phpCode/swoole/server/test.php on line 17

Fatal error: Uncaught Swoole\Error: can not create server after using async file operation in /root/codeDir/phpCode/swoole/server/test.php:17
Stack trace:
#0 /root/codeDir/phpCode/swoole/server/test.php(17): Swoole\Server->start()
#1 {main}
  thrown in /root/codeDir/phpCode/swoole/server/test.php on line 17
[2020-04-14 01:55:25 @52968.0]	ERROR	php_swoole_server_rshutdown (ERRNO 503): Fatal error: Uncaught Swoole\Error: can not create server after using async file operation in /root/codeDir/phpCode/swoole/server/test.php:17
Stack trace:
#0 /root/codeDir/phpCode/swoole/server/test.php(17): Swoole\Server->start()
#1 {main}
  thrown in /root/codeDir/phpCode/swoole/server/test.php on line 17
[root@2121d596e844 server]#
```

可以看到，这里报了一个错误：

```bash
can not create server after using async file operation

在使用异步文件操作后不能创建server
```

而且，我们注意到，这个错误是`Fatal`级别的，说明`Swoole`是禁止用户这么使用`System::fread`，我们**必须**对这样的代码进行修改。

我们再来写一个`PHP`脚本：

```php
<?php

use Swoole\Coroutine;
use Swoole\Coroutine\Client;
use Swoole\Server;

$serv = new Server('127.0.0.1', 9501);

Coroutine::create(function () {
    $client = new Client(SWOOLE_SOCK_TCP);
    $client->connect('localhost', 8088);
});

$serv->on('Receive', function () {
});

$serv->start();
```

执行结果如下：

```bash
[root@2121d596e844 server]# php test.php
PHP Fatal error:  Uncaught Swoole\Error: can not create server after using async file operation in /root/codeDir/phpCode/swoole/server/test.php:17
Stack trace:
#0 /root/codeDir/phpCode/swoole/server/test.php(17): Swoole\Server->start()
#1 {main}
  thrown in /root/codeDir/phpCode/swoole/server/test.php on line 17

Fatal error: Uncaught Swoole\Error: can not create server after using async file operation in /root/codeDir/phpCode/swoole/server/test.php:17
Stack trace:
#0 /root/codeDir/phpCode/swoole/server/test.php(17): Swoole\Server->start()
#1 {main}
  thrown in /root/codeDir/phpCode/swoole/server/test.php on line 17
[2020-04-14 02:01:16 @56567.0]	ERROR	php_swoole_server_rshutdown (ERRNO 503): Fatal error: Uncaught Swoole\Error: can not create server after using async file operation in /root/codeDir/phpCode/swoole/server/test.php:17
Stack trace:
#0 /root/codeDir/phpCode/swoole/server/test.php(17): Swoole\Server->start()
#1 {main}
  thrown in /root/codeDir/phpCode/swoole/server/test.php on line 17
[root@2121d596e844 server]#
```

依然报这个错误。

而这个错误的导致，是因为`Swoole`实现`Swoole\Coroutine\System::fread`用到了异步`IO`线程，而且异步`IO`线程是多线程的。那为什么使用了异步`IO`线程之后，不能创建`Server`了呢？因为异步`Server`是多进程的，即`master`进程会`fork`出`worker`进程。我们可以看一下`swoole_fork`的代码：

```cpp
pid_t swoole_fork(int flags)
{
    if (!(flags & SW_FORK_EXEC))
    {
        if (swoole_coroutine_is_in())
        {
            swFatalError(SW_ERROR_OPERATION_NOT_SUPPORT, "must be forked outside the coroutine");
        }
        if (SwooleTG.aio_init)
        {
            swFatalError(SW_ERROR_OPERATION_NOT_SUPPORT, "can not create server after using async file operation");
        }
    }
    // 省略其他代码
}
```

这里，会判断`SwooleTG.aio_init`是否被设置了，即是否用了异步`IO`线程。如果用了，那么就报错。

所以问题就变成了在多线程的情况下创建子进程会有什么问题。有什么问题可以看[云风的博客](https://blog.codingnow.com/2011/01/fork_multi_thread.html)。

我们这里主要是通过一个例子来复现他的博客所说的问题。我们编写如下`c++`代码：

```cpp
#include <thread>
#include <mutex>
#include <iostream>
#include <unistd.h>

std::mutex              g_mutex;

void foo1() {
    std::cout << "foo1 thread try lock" << std::endl;
    std::lock_guard<std::mutex> lock(g_mutex);
    std::cout << "foo1 thread lock success" << std::endl;
    sleep(5);
    std::cout << "foo1 thread exit, unlock" << std::endl;
}

void foo2() {
    pid_t pid;
    std::cout << "foo2 fork child process" << std::endl;
    pid = fork();

    if (pid == 0) // child
    {
        std::cout << "child process try lock" << std::endl;
        std::lock_guard<std::mutex> lock(g_mutex);
        std::cout << "child process lock success" << std::endl;
        sleep(1000);
    }
    else
    {
        sleep(1000);
    }
}

int main(int argc, char const *argv[])
{
    std::thread thread1(foo1);
    sleep(2);
    std::thread thread2(foo2);
    thread1.join();
    thread2.join();

    return 0;
}
```

编译：

```bash
[root@2121d596e844 test]# g++ test.cc -std=c++11 -pthread
```

执行代码：

```bash
[root@2121d596e844 test]# ./a.out
foo1 thread try lock
foo1 thread lock success
foo2 fork child process
child process try lock
foo1 thread exit, unlock

```

我们会发现子进程死锁了，一直没能获得锁。所以，这就是问题点了。多线程下`fork`子进程会有一定的问题，会丢失线程。

那么，为什么使用`$client->connect('localhost', 8088);`也是不行的呢？

因为这行代码涉及到了`DNS`查询，而`Swoole`进行`DNS`查询会用到`AIO`线程，所以这么用`connect`也会报错。如果我们把`localhost`换成`127.0.0.1`就没问题了。
