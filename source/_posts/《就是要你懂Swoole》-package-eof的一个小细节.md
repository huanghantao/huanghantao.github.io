---
title: 《就是要你懂Swoole》--package_eof的一个小细节
date: 2019-05-09 09:29:11
tags:
- Swoole
- PHP
---

有的同学在开启了`Swoole\Server`的`package_eof`的时候，发现并没有生效。`Server`端的代码如下：

```php
<?php

$serv = new Swoole\Server('127.0.0.1', 9501);

$serv->set(array(
    'worker_num' => 4,
    'package_eof' => "\r\n\r\n",
    'open_eof_check' => 1,
));

$serv->on('connect', function ($serv, $fd){
    echo "Client:Connect.\n";
});

$serv->on('receive', function ($serv, $fd, $reactor_id, $data) {
    $serv->send($fd, 'Swoole: '.$data);
    $serv->close($fd);
});

$serv->on('close', function ($serv, $fd) {
    echo "Client: Close.\n";
});

$serv->start();
```

`Client`端的代码如下：

```php
<?php

$client = new Swoole\Client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_ASYNC);

$client->on("connect", function(swoole_client $cli) {
    $cli->send('hello world\r\n\r\n');
});

$client->on("receive", function(swoole_client $cli, $data){
    echo "Receive: $data";
});

$client->on("error", function(swoole_client $cli){
    echo "error\n";
});

$client->on("close", function(swoole_client $cli){
    echo "Connection close\n";
});

$client->connect('127.0.0.1', 9501);
```

我们打开两个终端，分别启动`Server`和`Client`：

```shell
~/codeDir/phpCode/test # php server.php 
Client:Connect.

```

```shell
~/codeDir/phpCode/test # php client.php 

```

讲道理来说，`client`连接上`server`之后，是会给`server`发生字符串`'hello world\r\n\r\n'`的，但是看样子`server`好像并没有解析出这个字符串对吧。

问题出在了`server`端的`package_eof`的值用的是双引号，而`client`发送给`server`的字符串用的是单引号。而PHP在单引号和双引号中解析`\r\n\r\n`是不一样的（至少在字符串的长度来说都是不同的）。所以我们需要统一一下：

```php
<?php

$client = new Swoole\Client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_ASYNC);

$client->on("connect", function(swoole_client $cli) {
    $cli->send("hello world\r\n\r\n");
});

$client->on("receive", function(swoole_client $cli, $data){
    echo "Receive: $data";
});

$client->on("error", function(swoole_client $cli){
    echo "error\n";
});

$client->on("close", function(swoole_client $cli){
    echo "Connection close\n";
});

$client->connect('127.0.0.1', 9501);
```

```shell
~/codeDir/phpCode/test # php client.php 
Receive: Swoole: hello world

Connection close
~/codeDir/phpCode/test # 
```

此时，`server`就可以解析出这个`\r\n\r\n`了。