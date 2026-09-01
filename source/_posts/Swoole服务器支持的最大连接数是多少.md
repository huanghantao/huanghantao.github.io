---
title: Swoole服务器支持的最大连接数是多少
date: 2019-06-23 00:26:33
tags:
- Swoole
- PHP
---

前些日子，有人给`Swoole`提出了一个[Issue](https://github.com/swoole/swoole-src/issues/2638)，内容如下：

```
number of maximum connection we can use with swoole?

hello
i want to know how many is the maximum connection per second that swoole can support
we want use this framework for real-time game server and it is very important to know max number support connection/second for swoole framework.
thank you
```

峰哥回复如下：

```
It depends on the value of your system ulimit -n and your settings. The maximum limit is 1 million.

ulimit -n 100000
$server = new Swoole\Server;

$server->set(["max_connection" => 100000]);
```

说的是取决于我们设置的`ulimit -n`的值以及我们给`server`配置的值。我们来测试一下：

服务器代码：

```php
<?php // co_long_tcp_server.php 

$serv = new Swoole\Server("0.0.0.0", 9501);

$serv->on('receive', function (Swoole\Server $serv, $fd, $from_id, $data)
{
    $serv->send($fd, $data . $data);
});

$serv->start();
```

我们启动服务器：

```shell
php co_long_tcp_server.php 
```

然后，我们来看看我们的`ulimit -n`的值：

```shell
~/codeDir/phpCode/test/swoole # ulimit -a
-f: file size (blocks)             unlimited
-t: cpu time (seconds)             unlimited
-d: data seg size (kb)             unlimited
-s: stack size (kb)                8192
-c: core file size (blocks)        0
-m: resident set size (kb)         unlimited
-l: locked memory (kb)             82000
-p: processes                      unlimited
-n: file descriptors               1048576
-v: address space (kb)             unlimited
-w: locks                          unlimited
-e: scheduling priority            0
-r: real-time priority             0
```

我们发现，我们最大可以打开的文件个数是`1048576`。我们修改这个值：

```shell
~/codeDir/phpCode/test/swoole # ulimit -n 100
~/codeDir/phpCode/test/swoole # ulimit -a
-f: file size (blocks)             unlimited
-t: cpu time (seconds)             unlimited
-d: data seg size (kb)             unlimited
-s: stack size (kb)                8192
-c: core file size (blocks)        0
-m: resident set size (kb)         unlimited
-l: locked memory (kb)             82000
-p: processes                      unlimited
-n: file descriptors               100
-v: address space (kb)             unlimited
-w: locks                          unlimited
-e: scheduling priority            0
-r: real-time priority             0
~/codeDir/phpCode/test/swoole # 
```

此时，我们最大可以打开的文件个数是`100`个。好的，我们来进行测试。我使用我正在贡献的一个压测脚本来进行测试：

```shell
~/codeDir/cppCode/swoole-src/benchmark # php co_run.php -s tcp://127.0.0.1 -c 100 
============================================================
Swoole Version          4.4.0-alpha
============================================================


Warning: Swoole\Coroutine\Client::connect(): new Socket() failed, Error: No file descriptors available[24] in /root/codeDir/cppCode/swoole-src/benchmark/co_run.php on line 249

Warning: Swoole\Coroutine\Client::connect(): new Socket() failed, Error: No file descriptors available[24] in /root/codeDir/cppCode/swoole-src/benchmark/co_run.php on line 249

Warning: Swoole\Coroutine\Client::connect(): new Socket() failed, Error: No file descriptors available[24] in /root/codeDir/cppCode/swoole-src/benchmark/co_run.php on line 249

Warning: Swoole\Coroutine\Client::connect(): new Socket() failed, Error: No file descriptors available[24] in /root/codeDir/cppCode/swoole-src/benchmark/co_run.php on line 249


Concurrency Level:      100
Time taken for tests:   0.7476 seconds
Complete requests:      10,000
Failed requests:        400
Connect failed:         0
Total send:             9,830,400 bytes
Total reveive:          19,660,800 bytes
Requests per second:    12841.091492777
Connection time:        0.0091 seconds

~/codeDir/cppCode/swoole-src/benchmark # 
```

我们发现，此时报错了，说是没有可用的文件描述符。并且后面跟了一个错误码`24`，它对应如下含义：

```
Too many open files
```

为什么我们设置了最多打开`100`个文件，但是却无法维持`100`个客户端的连接呢？因为`Swoole`除了需要打开用于维持连接的套接字以外，还需要打开一些其他的文件，例如`epoll fd`等等。所以，我们设置`ulimit -n`为`100`不代表我们就可以维持`100`个连接。实际上要比`100`低。我们再次调低客户端连接的个数：

```shell
~/codeDir/cppCode/swoole-src/benchmark # php co_run.php -s tcp://127.0.0.1 -c 80 -v
============================================================
Swoole Version          4.4.0-alpha
============================================================

Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests


Concurrency Level:      80
Time taken for tests:   0.451 seconds
Complete requests:      10,000
Failed requests:        0
Connect failed:         0
Total send:             10,240,000 bytes
Total reveive:          20,480,000 bytes
Requests per second:    22172.949002217
Connection time:        0.0071 seconds

~/codeDir/cppCode/swoole-src/benchmark # 
```

我们发现，维持`80`个连接是可以的。

好的，我们设置回`ulimit -n`的值之后，修改一下服务器脚本：

```php
<?php

$serv = new Swoole\Server("0.0.0.0", 9501);
$serv->set(["max_connection" => 100]);

$serv->on('receive', function (Swoole\Server $serv, $fd, $from_id, $data)
{
    $serv->send($fd, $data . $data);
});

$serv->start();
```

设置最大连接为`100`。然后进行测试：

```shell
~/codeDir/cppCode/swoole-src/benchmark # php co_run.php -s tcp://127.0.0.1  -c 100 -v
============================================================
Swoole Version          4.4.0-alpha
============================================================

Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests


Concurrency Level:      100
Time taken for tests:   0.5277 seconds
Complete requests:      10,000
Failed requests:        0
Connect failed:         0
Total send:             10,240,000 bytes
Total reveive:          20,480,000 bytes
Requests per second:    18950.161076369
Connection time:        0.0105 seconds

~/codeDir/cppCode/swoole-src/benchmark # 
```

发现，维持`100`个连接是没问题的。





