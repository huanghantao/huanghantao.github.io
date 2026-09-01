---
title: 网络编程第一讲-Swoole与TCP三次握手
date: 2019-08-22 10:51:08
tags:
- PHP
- Swoole
---

这篇文章是总结自`Swoole`微课程《Swoole与TCP三次握手》。可以说，这套视频教程非常的棒，`Swoole`官方每天都会分享一个`10`到`20`分钟的视频，一天只需要`1`块钱！是夯实基础的好课程。微信搜索：`Swoole`微课程。

# 握手常见问题

```
1、连接拒绝
2、Operation now in progress
	多是因为丢包、错误ip、backlog满了&阻塞&tcp_abort_on_overflow=0
3、min(maxconn, backlog) ss -lt
```

## 连接拒绝

在`TCP`三次握手的时候，客户端发送`SYN`这个包给服务端，服务端不接受这个请求，操作系统直接返回了一个`RST`的包，来拒绝连接的请求。

最常见的情况就是客户端去请求某个服务器，服务端没有绑定对应的端口。

测试代码如下，服务端代码：

```php
<?php

$server = new \Swoole\Server('127.0.0.1', 9501);

$server->set([
    'work_num' => 2,
    'backlog' => 128,
]);

$server->on('connect', function ($server, $fd)
{
    echo "Client: Connect.\n";
});

$server->on('receive', function ($server, $fd, $reactor_id, $data)
{
    var_dump($data);
});

$server->on('close', function ()
{
    var_dump('close');
});

$server->start();
```

这里，服务端绑定的端口是`9501`。

启动服务器：

```shell
~/codeDir/phpCode/hyperf-skeleton # php server.php 

```

客户端代码：

```php
<?php

$client = new \Swoole\Client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_SYNC);
var_dump($client->connect('127.0.0.1', 9500));
```

这里，客户端请求的端口是`9500`。

启动客户端：

```shell
~/codeDir/phpCode/hyperf-skeleton # php client.php 

Warning: Swoole\Client::connect(): connect to server[127.0.0.1:9500] failed, Error: Connection refused[111] in /root/codeDir/phpCode/hyperf-skeleton/client.php on line 4
bool(false)
~/codeDir/phpCode/hyperf-skeleton # 
```

我们发现，报错：

```shell
Error: Connection refused[111]
```

## Operation now in progress

这个错误的绝大部分原因是因为连接超时了。

### 丢包

例如路由器、网关出现了故障，包被丢了。

### 错误ip

例如客户端请求了一个错误的`ip`，那么路由器自然也就路由不到。

测试代码如下，客户端代码：

```php
<?php

$client = new \Swoole\Client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_SYNC);
var_dump($client->connect('8.8.8.8', 9501));
```

这里，我访问的是谷歌的`DNS`服务器。因为我没有翻墙，所以是访问不了这个`IP`的。因此，我们发送的包是到达不了`8.8.8.8`服务器的。

启动客户端：

```shell
~/codeDir/phpCode/hyperf-skeleton # php client.php 

Warning: Swoole\Client::connect(): connect to server[8.8.8.8:9501] failed, Error: Operation in progress[115] in /root/codeDir/phpCode/hyperf-skeleton/client.php on line 4
bool(false)
~/codeDir/phpCode/hyperf-skeleton # 
```

我们发现，报错：

```shell
Error: Operation in progress[115]
```

### backlog

服务器在三次握手的最后一次，即收到客户端发来的`ACK`包的时候，会把建立好的连接放到`backlog`队列里面。如果`Swoole`一直不`accept`连接，那么这个`backlog`队列很快就会满。`backlog`队列满了之后，服务端就会丢弃三次握手的`SYN`包，让客户端重新去连接服务端。

测试代码如下，服务端代码：

```php
<?php

$server = new \Swoole\Server('127.0.0.1', 9501, SWOOLE_BASE);

$server->set([
    'work_num' => 2,
    'backlog' => 128,
]);

$server->on('connect', function ($server, $fd)
{
    echo "Client: Connect.\n";
    sleep(1000);
});

$server->on('receive', function ($server, $fd, $reactor_id, $data)
{
    var_dump($data);
});

$server->on('close', function ()
{
    var_dump('close');
});

$server->start();
```

要想测试`backlog`问题必须在`Swoole`的`SWOOLE_BASE`模式下，默认的`SWOOLE_PROCESS`模式是没有这个问题的。

这里，我们的`backlog`大小是`128`。

然后，我们通过`sleep(1000);`来阻塞住进程，使得`Swoole`不会继续`accept`连接，从而导致`backlog`队列在某个时刻变满。

客户端代码：

```php
<?php

$i = 0;
while (true)
{
    $client = new \Swoole\Client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_SYNC);
    if ($client->connect('127.0.0.1', 9501) == false)
    {
        break;
    }
}
```

我们启动服务器：

```shell
~/codeDir/phpCode/hyperf-skeleton # php server.php 

```

然后启动客户端：

```shell
~/codeDir/phpCode/hyperf-skeleton # php client.php 
省略了其他的输出
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)
bool(true)

Warning: Swoole\Client::connect(): connect to server[127.0.0.1:9501] failed, Error: Operation in progress[115] in /root/codeDir/phpCode/hyperf-skeleton/client.php on line 7
bool(false)

Warning: Swoole\Client::connect(): connect to server[127.0.0.1:9501] failed, Error: Operation in progress[115] in /root/codeDir/phpCode/hyperf-skeleton/client.php on line 7
bool(false)
^C
~/codeDir/phpCode/hyperf-skeleton # 
```

我们会发现，过一段时间，客户端这边会报错：

```shell
Error: Operation in progress[115]
```

服务端这边输出：

```shell
~/codeDir/phpCode/hyperf-skeleton # php server.php 
Client: Connect.

```

因为当`Swoole`服务器从`backlog`队列里面`accept`一个连接的时候，才会触发`onReceive`回调函数。所以，当服务端`accept`一个连接之后，`Swoole`自己就会陷入阻塞，不会再`accept`了。但是需要注意的是，尽管`Swoole`服务器自身是阻塞的，操作系统还会继续去把建立好的连接放入`backlog`队列里面。所以，`backlog`队列会满。

## SYN Flood

除了三次握手成功之后会使用到的`backlog`队列，还有一个`SYN `队列。也就是在三次握手时候，客户端给服务端发送了`SYN`包，服务端会有一个`SYN`队列来维护。

与其有关的内核配置：

```
tcp_max_syn_backlog
tcp_synack_retries
tcp_syncookies
```

### tcp_max_syn_backlog

其中，`tcp_max_syn_backlog`就是这个`SYN`队列的长度。如果大量的`SYN`包把`SYN`队列塞满了，那么其他正常的连接过来，服务端就无法处理。所以，适当增大这个值，可以在压力大的时候提高握手的成功率。手册里推荐大于`1024`。

### tcp_synack_retries

`SYN Flood`攻击就是客户端疯狂的给服务端发送`SYN`包，然后服务端每次都会把请求放到`SYN`队列里面。但是，客户端不给服务端回`ACK`包。如果客户端不回`ACK`包，那么服务端就会给客户端回`SYN + ACK`包，即第二次握手发送的包。而回复`SYN + ACK`包的次数就是由`tcp_synack_retries`参数决定的。如果把`tcp_synack_retries`设置为0，那么如果服务端没有收到`ACK`包，那么服务端就不会重试发送`SYN + ACK`包了，这样就减少了`SYN`队列里面那个请求的存活时间。因为对于正常的客户端，如果它接收不到服务器回应的`SYN + ACK`包，它会再次发送`SYN`包，客户端还是能正常连接的，只是可能在某些情况下建立连接的速度变慢了一点。

### tcp_syncookies

`tcp_syncookies`是这样解释的：

```
tcp_syncookies (Boolean; since Linux 2.2)
    Enable TCP syncookies. The kernel must be compiled with CONFIG_SYN_COOKIES. Send out syncookies when the syn backlog queue of a socket overflows. The syncookies feature attempts to protect a socket from a SYN flood attack. This should be used as a last resort, if at all. This is a violation of the TCP protocol, and conflicts with other areas of TCP such as TCP extensions. It can cause problems for clients and relays. It is not recommended as a tuning mechanism for heavily loaded servers to help with overloaded or misconfigured conditions. For recommended alternatives see tcp_max_syn_backlog, tcp_synack_retries, and tcp_abort_on_overflow.
    启用 TCP syncookies。 内核必须使用CONFIG_SYN_COOKIES进行编译。套接字的syn backlog queue溢出时发出syncookies。 syncookies特性试图保护套接字免受SYN洪水攻击。如果有必要的话，这应该作为最后的手段。这违反了TCP 协议，并与TCP扩展等其他区域发生冲突。
```

`tcp_syncookies`的原理就是，客户端发送`SYN`包的时候，不会维护`SYN`队列，而是返回一个`cookie`给客户端。然后客户端发送第三次握手的时候，携带这个`cookie`值，只有这个`cookie`验证通过，服务端才会给连接分配资源。