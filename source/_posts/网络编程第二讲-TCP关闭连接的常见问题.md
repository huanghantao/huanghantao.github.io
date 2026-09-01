---
title: 网络编程第二讲-TCP关闭连接的常见问题
date: 2019-08-22 14:21:36
tags:
- Swoole
- PHP
---

# TCP关闭的过程（四次握手）

```
1、客户端 发送FIN包给 服务端，此时客户端处于FIN_WAIT1状态
2、服务端 发送ACK包给 客户端，此时服务器处于CLOSE_WAIT状态，并且客户端在等待ACK包的时候，处于FIN_WAIT2状态
3、服务端 发送FIN包给 客户端，此时服务端处于LAST_ACK状态
4、客户端 发送ACK包给 服务端，此时客户端处于TIME_WAIT状态
```

## 第一次握手细节

客户端在应用层调用`close`方法时，操作系统底层会放送`FIN`包给服务端，此时客户端处于`FIN_WAIT1`状态。然后当服务端收到了`FIN`包之后，服务端在应用层的表现是调用`recv`方法得到的返回值是`0`，代表客户端关闭了连接。在`Swoole`中的表现是会触发`OnClose`回调函数。然后我们可以在`OnClose`回调函数里面写一些清除用户信息的代码。

## 第二次握手细节

服务端发送`ACK`包给客户端，此时服务端处于`CLOSE_WAIT`状态。顾名思义，就是服务端也要去调用`close`函数，才能把这个连接彻底的关闭（因为`TCP`是一个全双工的协议，客户端可以给服务端发，服务端也可以给客户端发）。所以这段等待的状态就叫做`CLOSE_WAIT`。

## 第三次握手细节

服务端应用层调用`close`函数，操作系统底层发送`FIN`包给客户端，此时服务端处于`LAST_ACK`状态。顾名思义，服务端需要去等待客户端发来的`ACK`。

## 第四次握手细节

客户端发送`ACK`包给服务端，此时客户端处于`TIME_WAIT`状态，而`TIME_WAIT`状态的时间长达`1`分钟。服务端收到了`ACK`包之后，整个`TCP`连接的生命周期就算结束了。

## TIME_WAIT状态

### 为什么要有TIME_WAIT状态

那么，这里客户端在发送`ACK`包之后为什么需要处于一个`TIME_WAIT`状态，而不是立马结束了呢？（我们知道，如果连接处于`TIME_WAIT`状态的话，一般情况下是不可以重新使用同一个`IP`和端口的。这在一些情况下就会给我们造成不愉快。）

因为服务端可能会没有收到客户端发来的`ACK`包。如果，服务端没有收到`ACK`包，那么服务端会重发一个`FIN`包给客户端，如果没有`TIME_WAIT`状态，那么客户端就会丢失服务端重传的`FIN`包。而客户端发送`ACK`包以及服务端重传`FIN`包的时间，就是`TIME_WAIT`的时长。如果在这个时长（`1`分钟）里面，客户端没有收到服务端重传的`FIN`包，那么客户端就认为服务端收到了`ACK`包，可以结束`TIME_WAIT`状态了。

### 为什么TIME_WAIT状态持续1分钟

那么，为什么`TIME_WAIT`的时长是`1`分钟之久呢？

因为一个`IP`存活的时间是`1`分钟，为了防止下一个连接收到之前连接的数据包。

如果客户端在发送完`ACK`包之后，立马结束掉。但是，紧接着又重新使用了同一个`IP`和端口，那么很有可能就会收到上一个连接发来的数据。（因为由于网络的原因，服务端发送的数据包到达客户端可能会有一些延迟）。收到上一个连接的包，这显然是我们不希望看到的。

所以，客户端在发送`ACK`包之后，会去处于一个`TIME_WAIT`状态，等待`1`分钟。而这`1`分钟内，一般情况下，客户端是不可以立马重用同一个`IP`和端口的。这样，就避免了客户端起了第二个连接，却收到了之前的连接的包。因为`1`分钟结束后，之前连接的包已经不存在了。

### Cannot assign requested address

这个问题是针对客户端随机选取一个端口的时候，发现没有端口可用会报这个错误，即所谓的端口用光了。

如果我们客户端并发连接数很大，而每个连接最后都处于`TIME_WAIT`状态，那么我们机器的端口迟早会被用完，因为我们的机器端口数量是有限的。

所以，端口用光的根本原因是有大量处于`TIME_WAIT`状态的连接。

而`PHP`传统的`FPM`模式下，会比较常见这种问题，因为客户端每次请求结束，我们服务的连接数据库的连接也会断开。而一旦并发量大了上来，就会出现端口不够的情况了。

### Address already in use

`Address already in use`是我们自己决定使用某个端口，发现它被占用的时候会报这个问题。与`Cannot assign requested address`是有区别的，因为`Cannot assign requested address`是系统帮我们随机选择一个端口，发现没有端口可以选择了，才会报`Cannot assign requested address`的问题，即所谓的端口用光了。

可以通过`SO_RESUADDR`来解决。

## CLOSE_WAIT状态

如果我们被动关闭的一方没有调用`close`函数，那么被动关闭方就会处于`close_wait`状态。

`Swoole`服务器在触发完`OnClose`回调函数之后，维护连接的那个进程会自动的帮我们执行`close`函数。所以，就很少会见到`CLOSE_WAIT`状态。

但是，如果我们在`OnClose`回调函数里面阻塞了或者说在`OnReceive`里面阻塞了（只要主动关闭方关闭了连接，被动关闭方的`OnClose`回调函数无法顺利执行完或者没有被执行），那就会出现`CLOSE_WAIT`状态。

举个例子：

```php
<?php

$server = new \Swoole\Server('127.0.0.1', 9501, SWOOLE_PROCESS);

$server->set([
    'work_num' => 2,
]);

$server->on('connect', function ($server, $fd)
{
});

$server->on('receive', function ($server, $fd, $reactor_id, $data)
{
});

$server->on('close', function ()
{
    sleep(1000);
});

$server->start();
```

此时服务器在`OnClose`回调函数里面阻塞了，导致这个函数无法正常结束，导致`Swoole`底层无法在`OnClose`回调函数结束之后，自动帮我们执行`close`函数（本质上，是`worker`进程无法通知主进程去`close`）。

客户端代码：

```php
<?php

$client = new \Swoole\Client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_SYNC);
$client->connect('127.0.0.1', 9501);
$client->close();
```

这里，客户端连接完服务器之后，主动关闭了连接。

执行服务器：

```shell
~/codeDir/phpCode/hyperf-skeleton # php server.php 

```

执行客户端：

```shell
~/codeDir/phpCode/hyperf-skeleton # php client.php 
~/codeDir/phpCode/hyperf-skeleton # 
```

然后查看网络状态：

```shell
~/codeDir/phpCode/hyperf-skeleton # netstat -antp
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:9501          0.0.0.0:*               LISTEN      23308/php
tcp        0      0 127.0.0.11:37569        0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:54004         127.0.0.1:9501          FIN_WAIT2   -
tcp        0      0 127.0.0.1:9501          127.0.0.1:54004         CLOSE_WAIT  23308/php
~/codeDir/phpCode/hyperf-skeleton # 
```

我们发现：

```shell
tcp        0      0 127.0.0.1:9501          127.0.0.1:54004         CLOSE_WAIT  23308/php
```

此时，服务端这边的状态是`CLOSE_WAIT`。因为服务端没有发生`FIN`包，所以客户端这边处于`FIN_WAIT2`状态。

我们过一段时间，例如`2`分钟再次查看网络状态：

```shell
~/codeDir/phpCode/hyperf-skeleton # netstat -antp
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:9501          0.0.0.0:*               LISTEN      23308/php
tcp        0      0 127.0.0.11:37569        0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:9501          127.0.0.1:54004         CLOSE_WAIT  23308/php
~/codeDir/phpCode/hyperf-skeleton # 
```

我们发现服务端这边还是处于`CLOSE_WAIT`状态，一直没有消失。这个就叫做**连接泄漏**。

我们再换一种测试方法，我们在`OnClose`回调函数里面执行`die`，或者抛出异常，或者有致命错误导致`worker`进程直接结束了，此时`Swoole`底层也无法帮我们执行`close`方法：

```php
<?php

$server = new \Swoole\Server('127.0.0.1', 9501, SWOOLE_PROCESS);

$server->set([
    'work_num' => 2,
]);

$server->on('connect', function ($server, $fd)
{
});

$server->on('receive', function ($server, $fd, $reactor_id, $data)
{
});

$server->on('close', function ()
{
    die;
});

$server->start();
```

我们重新启动服务器：

```shell
~/codeDir/phpCode/hyperf-skeleton # php server.php 

```

然后，我们确认下网络状态：

```shell
~/codeDir/phpCode/hyperf-skeleton # netstat -antp
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:9501          0.0.0.0:*               LISTEN      26276/php
tcp        0      0 127.0.0.11:37569        0.0.0.0:*               LISTEN      -
~/codeDir/phpCode/hyperf-skeleton # 
```

此时服务端处于`LISTEN`状态。

然后，我们执行刚才的客户端代码：

```shell
~/codeDir/phpCode/hyperf-skeleton # php client.php 
```

此时，服务端报错：

```shell
~/codeDir/phpCode/hyperf-skeleton # php server.php 

Fatal error: Uncaught Swoole\ExitException: swoole exit in /root/codeDir/phpCode/hyperf-skeleton/server.php:19
Stack trace:
#0 {main}
  thrown in /root/codeDir/phpCode/hyperf-skeleton/server.php on line 19
[2019-08-22 08:08:54 *26286.0]	ERROR	php_swoole_server_rshutdown (ERRNO 503): Fatal error: Uncaught Swoole\ExitException: swoole exit in /root/codeDir/phpCode/hyperf-skeleton/server.php:19
Stack trace:
#0 {main}
  thrown in /root/codeDir/phpCode/hyperf-skeleton/server.php on line 19
[2019-08-22 08:08:54 $26277.0]	WARNING	swManager_check_exit_status: worker#0[pid=26286] abnormal exit, status=255, signal=0

```

然后，我们查看一下网络状态：

```shell
~/codeDir/phpCode/hyperf-skeleton # netstat -antp
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:9501          0.0.0.0:*               LISTEN      26276/php
tcp        0      0 127.0.0.11:37569        0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:54006         127.0.0.1:9501          FIN_WAIT2   -
tcp        0      0 127.0.0.1:9501          127.0.0.1:54006         CLOSE_WAIT  26276/php
~/codeDir/phpCode/hyperf-skeleton # 
```

此时，服务端这边处于`CLOSE_WAIT`状态，成功的导致了连接泄漏。

这是我们`Swoole`的服务器处于`SWOOLE_PROCESS`模式下会出现的问题。但是，`SWOOLE_BASE`模式下不会出现这个问题。

因为`SWOOLE_PROCESS`模式下连接是和`master`进程里面建立的，而`SWOOLE_BASE`模式下连接是和`worker`进程直接建立的，`worker`进程结束之后，操作系统底层会自动发一个`FIN`包给客户端。

我们来测试一下，服务端的代码：

```php
<?php

$server = new \Swoole\Server('127.0.0.1', 9501, SWOOLE_BASE);

$server->set([
    'work_num' => 2,
]);

$server->on('connect', function ($server, $fd)
{
});

$server->on('receive', function ($server, $fd, $reactor_id, $data)
{
});

$server->on('close', function ()
{
    die;
});

$server->start();
```

然后重新启动服务器：

```shell
~/codeDir/phpCode/hyperf-skeleton # php server.php 

```

此时确认网络状态：

```shell
~/codeDir/phpCode/hyperf-skeleton # netstat -antp
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:9501          0.0.0.0:*               LISTEN      30174/php
tcp        0      0 127.0.0.11:37569        0.0.0.0:*               LISTEN      -
~/codeDir/phpCode/hyperf-skeleton # 
```

服务端的连接处于`LISTEN`状态。

然后启动客户端：

```shell
~/codeDir/phpCode/hyperf-skeleton # php client.php 
~/codeDir/phpCode/hyperf-skeleton # 
```

然后查看网络状态：

```shell
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.11:37569        0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:54008         127.0.0.1:9501          TIME_WAIT   -
~/codeDir/phpCode/hyperf-skeleton # 
```

此时，服务器端的状态没了，因此没出现`CLOSE_TIME`状态。因为服务端发送了`FIN`包，所以客户端这边不再是`FIN_WAIT2`状态，而是`TIME_WAIT`状态了。

我们看看服务器端的输出：

```shell
~/codeDir/phpCode/hyperf-skeleton # php server.php 

Fatal error: Uncaught Swoole\ExitException: swoole exit in /root/codeDir/phpCode/hyperf-skeleton/server.php:19
Stack trace:
#0 {main}
  thrown in /root/codeDir/phpCode/hyperf-skeleton/server.php on line 19
[2019-08-22 08:19:27 *30174.0]	ERROR	php_swoole_server_rshutdown (ERRNO 503): Fatal error: Uncaught Swoole\ExitException: swoole exit in /root/codeDir/phpCode/hyperf-skeleton/server.php:19
Stack trace:
#0 {main}
  thrown in /root/codeDir/phpCode/hyperf-skeleton/server.php on line 19
~/codeDir/phpCode/hyperf-skeleton # 
```

我们发现，报错之后，`Swoole`服务器直接退出了。

同理，在`SWOOLE_BASE`模式下，因为连接的`accept`是在`Worker`进程里面进行的，所以`worker`进程一旦业务逻辑里面出现了阻塞，那么`backlog`就很容易塞满。但是`SWOOLE_PROCESS`模式下，因为`accept`是在主进程进行的，所以，如果我们在`worker`进程里面阻塞了，是不会影响到`backlog`的。这也是为什么`Swoole`会把连接的处理和业务逻辑的处理分成多个进程来处理。

（我们要清楚的知道，常用的事件回调函数是在主进程里面调用的还是`worker`进程里面调用的）

