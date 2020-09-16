---
title: 使用tsw_co协程库编写同步代码实现异步效果
date: 2019-02-28 00:54:47
tags:
- 协程
---

这里，我通过一个简单的回射服务器的例子来展示一下：

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/ip.h>
#include <string.h>
#include <errno.h>
#include "log.h"
#include "coroutine.h"
#include "socket.h"
#include "net.h"
#include "fd.h"

#define HOST "127.0.0.1"
#define PORT 9501
#define MAX_BUF_SIZE 1024
#define LISTENQ 10

void client_handle(tswCo_schedule *S, void *ud) {
    int n;
    int connfd;
    char buf[MAX_BUF_SIZE];

    connfd = (int)(uintptr_t)ud;
    while (1) {
        if ((n = tswCo_recv(S, connfd, buf, MAX_BUF_SIZE, 0)) < 0) {
            tswWarn("tswCo_recv error");
        }
        if (n == 0) {
            if (tswCo_close(S, connfd) < 0) {
                tswWarn("tswCo_shutdown error");
            }
            break;
        } else {
            if (tswCo_send(S, connfd, buf, n, 0) < 0) {
                tswWarn("tswCo_send error");
            }
        }
    }
}

void listen_service(tswCo_schedule *S, void *ud)
{
    int sockfd;
    int connfd;
    struct sockaddr_in cliaddr;
    socklen_t len;

    sockfd = (int)(uintptr_t)ud;

    while ((connfd = tswCo_accept(S, sockfd, (struct sockaddr *)&cliaddr, &len)) > 0) {
        tswDebug("a new connection [%d]", connfd);
        tswCo_create(S, TSW_CO_DEFAULT_ST_SZ, client_handle, (void *)(uintptr_t)connfd);
    }
}

int start_service(tswCo_schedule *S)
{
    int sockfd;

    sockfd = tswSocket_create(TSW_SOCK_TCP);
    if (sockfd < 0) {
        tswWarn("tswSocket_create error");
        return -1;
    }
    if (tswSocket_bind(sockfd, TSW_SOCK_TCP, HOST, PORT) < 0) {
		tswWarn("tswSocket_bind error");
        return -1;
	}
    if (listen(sockfd, LISTENQ) < 0) {
        tswWarn("%s", strerror(errno));
    }

    tswCo_create(S, TSW_CO_DEFAULT_ST_SZ, listen_service, (void *)(uintptr_t)sockfd);
    tswCo_run(S);

    return 0;
}

/*
 * main coroutine
*/
int main(int argc, char const *argv[])
{
    tswCo_schedule *S;

    S = tswCo_open();
    if (S == NULL) {
        tswWarn("tswCo_open error");
        return -1;
    }
    start_service(S);
    tswCo_destroy(S);

    return 0;
}
```

我们可以开启多个终端来进行测试，发现可以处理多个客户端的请求，

终端1：

```shell
~/codeDir/phpCode # nc 127.0.0.1 9501
1
1
2
2
```

终端2：

```shell
~/codeDir/phpCode # nc 127.0.0.1 9501
33
33
44
44
```

终端3：

```c
~/codeDir/phpCode # nc 127.0.0.1 9501
55
55
6
6
7
7
8
8
```

如果没有使用协程，那么我们直接通过阻塞的方式调用accept、recv、send是不可能在一个线程里面处理多个客户端的。可见，通过协程，使得我们编写代码变得轻松起来了。

[仓库地址](https://github.com/huanghantao/tsw_co)

