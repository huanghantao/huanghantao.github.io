---
title: 《就是要你懂swoole》-Server（二）
date: 2018-11-06 14:14:19
tags:
- swoole
- php
---

小伙伴们，大家好，这篇博客给大家带来的是有关swoole server的`set`方法的相关知识。对应的官方文档[在这里](https://wiki.swoole.com/wiki/page/13.html)。

这个方法是用来设置服务器启动时候的参数的。官网中给出了一个例子：

```php
$serv->set(array(
    'reactor_num' => 2, //reactor thread num
    'worker_num' => 4,    //worker process num
    'backlog' => 128,   //listen backlog
    'max_request' => 50,
    'dispatch_mode' => 1,
));
```

我们一个一个的来看看这几个参数是什么意思。

第一个参数说是设置Reactor线程的数目。说到Reactor线程，我们得先理清一下swoole进程、线程的架构了。

![来自官网的图片](https://wiki.swoole.com/static/image/process.jpg)

![](https://wiki.swoole.com/static/uploads/wiki/201808/03/635680420659.png)

（图片来自官网）

总结一下：

- Master线程对连接进行accept。
- Reactor线程处理连接，读取客户端发来的请求数据（Receive），将请求封装好后投递给work进程。
- Work进程用来处理业务数据，然后把处理后的结果返回给Reactor线程。
- Reactor线程将结果发送给客户端（Sendto）。

（至于其他的一些功能，我们这篇文章先不讲，我们聚焦官网给出的那5个参数）

第二个参数说的是设置Worker进程（进程的概念请看我[第二篇文章](https://zhuanlan.zhihu.com/p/40762899)）的数目。

第三个参数说的是设置监督的backlog大小。它指定了等待`accept`系统调用的已建立连接队列的长度。

第四个参数说的是每个worker进程处理50次请求之后就会自动重启。

第五个参数说的是worker进程数据包分配模式，1代表平均分配。

ok，我们接下来通过实践来直观的感受一下。

```php
<?php

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

然后启动服务器：

```shell
php server2.php
```

```shell
[2018-11-06 16:35:16 @53435.0]	TRACE	php_swoole_server_before_start(:593): Create swoole_server host=0.0.0.0, port=9501, mode=2, type=1
[2018-11-06 16:35:16 @53435.0]	TRACE	swReactorKqueue_add(:152): [THREAD #0]EP=14|FD=4, events=1
[2018-11-06 16:35:16 @53435.0]	TRACE	swReactorKqueue_add(:152): [THREAD #1]EP=15|FD=9, events=5
[2018-11-06 16:35:16 @53435.0]	TRACE	swReactorKqueue_add(:152): [THREAD #0]EP=16|FD=7, events=5
[2018-11-06 16:35:16 @53435.0]	TRACE	swReactorKqueue_add(:152): [THREAD #1]EP=15|FD=13, events=5
[2018-11-06 16:35:16 @53435.0]	TRACE	swReactorKqueue_add(:152): [THREAD #0]EP=16|FD=11, events=5
[2018-11-06 16:35:17 #53435.2]	TRACE	swTimer_add(:171): id=1, exec_msec=1000, msec=1000, round=0
[2018-11-06 16:35:17 @53437.0]	TRACE	swReactorKqueue_add(:152): [THREAD #0]EP=4|FD=5, events=517
[2018-11-06 16:35:17 @53438.0]	TRACE	swReactorKqueue_add(:152): [THREAD #0]EP=4|FD=8, events=517
[2018-11-06 16:35:17 @53439.0]	TRACE	swReactorKqueue_add(:152): [THREAD #0]EP=4|FD=10, events=517
[2018-11-06 16:35:17 @53440.0]	TRACE	swReactorKqueue_add(:152): [THREAD #0]EP=4|FD=12, events=517
```

之后，我们通过命令`pstree`来查看一下进程之间的关系：

```shell
pstree | grep server2
```

```shell
 | |   \-+= 53435 hantaohuang php server2.php
 | |     \-+- 53436 hantaohuang php server2.php
 | |       |--- 53437 hantaohuang php server2.php
 | |       |--- 53438 hantaohuang php server2.php
 | |       |--- 53439 hantaohuang php server2.php
 | |       \--- 53440 hantaohuang php server2.php
```

可以看出，我们启动服务器之后，有6个进程与服务器有关，进程id分别为53435、53436、53437、53438、53439、53440。它们的层次关系就体现出了谁是父进程，谁是子进程。结合上面的架构图，我们可以很轻易的分析出这四个进程分别对应着：

```
53435 --> Master进程
53436 --> Manager进程
53437 --> Worker1进程
53438 --> Worker2进程
53439 --> Worker3进程
53440 --> Worker4进程
```

然后我们查看一下Reactor线程的个数。我们从上面的架构图已经知道了Reactor线程是属于Master进程的。所以，我们可以通过如下命令查看：

```shell
ps M 53435
```

```shell
USER          PID   TT   %CPU STAT PRI     STIME     UTIME COMMAND
hantaohuang 53435 s006    2.1 S    31T   0:02.44   0:00.93 php server2.php
            53435         0.0 S    31T   0:00.00   0:00.00 
            53435         0.0 S    31T   0:00.00   0:00.00 
```

可以看出，这里有3个线程，一个是Master线程，两个Reactor线程。

这里，我们又抛出了线程的概念。如果同学们之前理解了进程的概念，然后再来理解线程的概念，就会比较简单了。目前，我们只需要这样简单理解此时的进程和线程模型即可：

![1(1)](/Users/hantaohuang/Library/Containers/com.tencent.WeWorkMac/Data/Library/Application Support/WXWork/Data/1688850523145108/Cache/Image/2018-11/1(1).PNG)









