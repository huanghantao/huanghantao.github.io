---
title: 在开发tinyswoole的时候，worker进程里面执行send函数时候遇到的问题
date: 2018-12-25 17:57:16
tags:
- swoole
---

我们知道，tinyswoole的架构是多个线程（一个master线程，多个reactor线程），多个worker进程。master线程负责处理客户端发来的连接请求，获得connfd，然后把connfd交给reactor线程来进行处理。

master线程选择哪个reactor线程来处理connfd的算法是：

```c
reactor_id = connfd % serv->reactor_num;
```

```c
/*
 * reactor: Used to manage handle in tswEvent
*/
int tswServer_master_onAccept(tswReactor *reactor, tswEvent *tswev)
{
	int connfd;
	socklen_t len;
	struct sockaddr_in cliaddr;
	tswServer *serv = reactor->ptr;
	tswReactor *sub_reactor;

	len = sizeof(cliaddr);
	connfd = accept(tswev->fd, (struct sockaddr *)&cliaddr, &len);
	if (connfd < 0) {
		tswWarn("%s", "accept error");
		return TSW_ERR;
	}

	serv->status->accept_count++;

	sub_reactor = &(serv->reactor_threads[connfd % serv->reactor_num].reactor);

	serv->connection_list[connfd].connfd = connfd;
	serv->connection_list[connfd].session_id = serv->status->accept_count;
	serv->connection_list[connfd].from_reactor_id = sub_reactor->id;
	serv->connection_list[connfd].serv_sock = serv->serv_sock;

	serv->session_list[serv->status->accept_count].session_id = serv->status->accept_count;
	serv->session_list[serv->status->accept_count].connfd = connfd;
	serv->session_list[serv->status->accept_count].reactor_id = sub_reactor->id;
	serv->session_list[serv->status->accept_count].serv_sock = serv->serv_sock;

	serv->onConnect(serv->status->accept_count);
	
	if (sub_reactor->add(sub_reactor, connfd, TSW_EVENT_READ, tswServer_reactor_onReceive) < 0) {
		tswWarn("%s", "reactor add error");
		return TSW_ERR;
	}

	return TSW_OK;
}
```

之后，reactor线程就可以通过connfd获取客户端请求过来的数据。再然后，reactor线程对这些数据进行封装，发送给了worker进程。

ok，此时轮到worker进程开始处理reactor线程发来的数据了。

worker进程处理数据这个过程很简单，只需要在worker进程里面去调用php用户空间的代码即可。例如：

```php
<?php

function onReceive($serv, $fd, $data)
{
    print_r("receive data from client[{$fd}]: {$data}");
    $serv->send($fd, "hello client");
}

$serv = new TinySwoole\Server('127.0.0.1', 9501, TSWOOLE_TCP);

$serv->set([
    'reactor_num' => 4,
    'worker_num' => 4,
]);

$serv->on("Receive", "onReceive");
$serv->start();

```

`Receive`事件处理函数中，`$data`是worker进程需要处理的数据。我们来看一看worker进程是如何调用用户空间的`onReceive`事件处理函数：

```c
static int tswWorker_onPipeReceive(tswReactor *reactor, tswEvent *tswev)
{
    int n;
	tswEventData event_data;

	// tswev->fd represents the fd of the pipe
    n = read(tswev->fd, &event_data, sizeof(event_data));
    if (event_data.info.len > 0) {
		TSwooleG.serv->onReceive(TSwooleG.serv, &event_data);
    }

	return TSW_OK;
}
```

可以看出，当worker进程收到reactor线程封装的数据（event_data）之后，就调用了全局变量`serv`（worker进程的serv全局变量是worker进程在master进程fork的时候拷贝过来的一个副本）的

`onReceive`事件处理函数来对`event_data`进行处理。我们来看一看serv的onReceive做了些什么事情：

```c
/**
 * @fd: session_id
 * 
 * Event_data saves the data sent from the client
 */
void php_tswoole_onReceive(tswServer *serv, tswEventData *event_data)
{
	zval *zfd;
	zval *zdata;
	zval retval;
	zval args[3];

	TSW_MAKE_STD_ZVAL(zfd);
	ZVAL_LONG(zfd, event_data->info.fd);
	TSW_MAKE_STD_ZVAL(zdata);
	ZVAL_STRINGL(zdata, event_data->data, event_data->info.len);

	args[0] = *server_object;
	args[1] = *zfd;
	args[2] = *zdata;

	call_user_function_ex(EG(function_table), NULL, php_tsw_server_callbacks[TSW_SERVER_CB_onReceive], &retval, 3, args, 0, NULL);
}
```

这里的关键代码是PHP提供的`call_user_function_ex`。这个函数是用来执行PHP用户空间定义好的一个函数。我们很容易知道，`php_tsw_server_callbacks[TSW_SERVER_CB_onReceive]`就是上面PHP代码的`onReceive`事件处理函数。这里，按照用户空间定义的样子，分别传递了三个参数serv, ​fd, ​data给`onReceive`函数。OK，到了这里，一切都是很容易做到的。但是，当PHP代码执行到：

```php
$serv->send($fd, "hello client");
```

的时候，麻烦事就来了。

如果之前小伙伴们没有学习过网络编程的话，或许看不出问题所在。但是，如果你学习过网络编程，你很容易的就会发现，这里的$fd是从1开始递增的，不符合文件打开的一些行为（因为如果你没有对文件描述符0，1，2关闭的话，fd应该是从3开始计的）。而且，当客户端关闭的时候，fd并不会使用这个fd，而是会在最大的那个fd的基础上加1。所以，我们应该要立刻反应过来，这里的fd不是connfd，而是session_id，用来标识一个连接（当然，这个session_id是要和connfd进行对应的）。我们现在来看看这个如何实现。

我们来看看数据结构：

```c
struct _tswServer {
    // other attribute
    tswConnection *connection_list;
    tswSession *session_list;
    tswServerStatus *status;
};
```

```c
struct _tswServerStatus {
    uint32_t accept_count;
};
```

也就是说，这里有一个属性accept_count来记录当前的session_id是多少。然后我们来看看tswConnection和tswSession这两个数据结构：

```c
struct _tswConnection {
	int connfd;
	uint32_t session_id;
	uint32_t from_reactor_id;
	int serv_sock;
};

struct _tswSession {
    uint32_t session_id;
    int connfd;
    uint32_t reactor_id;
	int serv_sock;
};
```

我们可以看到，这两个数据结构是一样的，那么为什么需要定义两个一样的数据结构呢？因为我们的服务器经常会需要在session_id和connfd之间进行转换。例如，我需要从session_id转换为connfd，有如下代码：

```c
session_id = event_data.info.fd;
session = &(TSwooleG.serv->session_list[session_id]);

send(session->connfd, event_data.data, event_data.info.len, 0);
```

同理，从connfd转换为session_id也是一样的。假设我们只有connection_list，如果我们要通过session_id找到connfd，那么需要遍历一遍connection_list数组。