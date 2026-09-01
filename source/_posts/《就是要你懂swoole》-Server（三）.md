---
title: 《就是要你懂swoole》-Server（三）
date: 2018-11-29 15:27:49
tags:
- PHP
- Swoole
---

上一篇文章我们讲解完了下面这段代码的`$config`含义：

```php
<?php
    
// filename: server.php

$config = [
    'reactor_num' =>2, // Reactor线程个数
    'worker_num' => 4, // Worker进程个数
];

$serv = new swoole_server("0.0.0.0", 9501, SWOOLE_PROCESS, SWOOLE_SOCK_TCP);

$serv->set($config);

$serv->on('connect', function ($serv, $fd) {  
    echo "Client: Connect.\n";
});

$serv->on('receive', function ($serv, $fd, $from_id, $data) {
    $serv->send($fd, "Server: ".$data);
});

$serv->on('close', function ($serv, $fd) {
    echo "Client: Close.\n";
});

$serv->start();

```

OK，这一篇文章我们来解释剩下的部分 -- serv的on方法。对应的官方文档[在这里](https://wiki.swoole.com/wiki/page/142.html)。

on方法的声明如下：

```php
bool Server->on(string $event, mixed $callback);
```

on方法是用来给我们的服务器serv**注册事件回调函数**的。其中，`$event`是事件名字，`$callback`是事件对应的回调函数。在上面这段代码中，分别注册了connect、receive、close，3个事件的回调函数。

这里，小伙伴们要注意了，**on方法是用来注册事件回调函数的，不是用来注册事件的**。为什么？因为这些connect、receive、close等等事件都是已经被服务器支持了的，我们通过swoole的源码可以很容易的分析出来，我后面也会给我们的tinyswoole服务器支持这些事件。

那么注册这些事件回调函数是为了什么？很容易理解，就是当我们的服务器遇到了这些事件之后，触发这些函数。OK，现在我们运行一下这个服务器：

```shell
php server.php
```

我们新开一个shell，输入如下命令：

```shell
nc 127.0.0.1 9501
```

此时，会启动一个客户端连接我们的服务器。此时，我们查看一下服务器的输出，可以看到：

```shell
Client: Connect.
```

被打印了出来。说明，服务器的`connect`事件被触发了，所以执行了回调函数：

```php
function ($serv, $fd) {  
    echo "Client: Connect.\n";
}
```

也就是说，connect是在客户端连接了服务器时触发的。

OK，我们在执行nc命令的这个终端中输入一个字符串`hello world`：

```shell
nc 127.0.0.1 9501
hello world
Server: hello world
```

你将会发现，这个nc客户端接收到了字符串`Server: hello world`。说明，我们的`receive`事件被触发了。也就是说，receive事件是在服务器收到了客户端的消息时触发。

OK，我们在执行nc命令的这个终端中按下`ctrl + c`。此时，nc这个客户端就会挂了（结束进程）。此时，我们查看一下服务器的输出，可以看到：

```shell
Client: Close.
```

被打印了出来。说明，服务器的`close`事件被触发了。也就是说，close事件是在服务器知道客户端断开了连接时触发的。

OK，接下来，我们就来给我们tinyswoole服务器扩展支持on方法吧。你将会学习到如下知识点：

- 如何在php扩展中注册事件回调函数。

- 事件触发的时候如何去执行用户定义的函数。

如果小伙伴们想提前感受一下这部分功能如何去写，可以[点击这里](https://github.com/huanghantao/tinyswoole/tree/dev)。

