---
title: Swoole自定义进程与混合端口服务的配合思路
date: 2020-06-07 17:44:43
tags:
- PHP
- Swoole
---

举一个很常见的场景，我们要实现一个监控服务，当这个服务监控到了数据变化的时候，需要通知订阅了监控的那些客户端。但是，这些客户端可能是`TCP`客户端，也可能是`Websocket`客户端。那么，我们可以做如下的设计。

首先，我们需要一个自定义进程来作为我们的监控进程。

然后，服务器需要支持`TCP`和`Websocket`协议。

然后，当监控进程监控到了数据变化的时候，需要把这些数据发送给`Worker`进程。

最后，`Worker`进程把收到的数据推送给`TCP`和`Websocket`客户端。

我们通过伪代码来进行实现：


```php
<?php


$websocketServer = new Swoole\WebSocket\Server('127.0.0.1', 9501);

$websocketServer->on('PipeMessage', function (Swoole\Server $server, int $srcWorkerID, string $data) {
    foreach ($websocketSubscribe as $key => $fd) {
        $server->push($fd, $data);
    }
    foreach ($websocketSubscribe as $key => $fd) {
        $server->send($fd, pack('N', strlen($data)) . $data);
    }
});

$websocketServer->on('Message', function (Swoole\WebSocket\Server $server, Swoole\WebSocket\Frame $frame) {
    $websocketSubscribe[] = $frame->fd;
});

/**@var Swoole\Server */
$tcpServer = $websocketServer->addlistener('127.0.0.1', 9502, SWOOLE_SOCK_TCP);

$tcpServer->on('Receive', function (Swoole\Server $server, int $fd, int $reactorID, string $data) {
    $tcpSubscribe[] = $fd;
});

$process = new Swoole\Process(function ($process) use ($tcpServer) {
    while (true) {
        watch(function (string $data) use ($tcpServer) {
            for ($i = 0; $i < $maxWorkerId; $i++) {
                $tcpServer->sendMessage($data, $i);
            }
        });
    }
});

$websocketServer->addProcess($process);

$websocketServer->start();
```
