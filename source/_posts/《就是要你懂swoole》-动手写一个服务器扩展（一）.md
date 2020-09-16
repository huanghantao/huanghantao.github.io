---
title: 《就是要你懂swoole》-动手写一个服务器扩展（一）
date: 2018-11-19 15:45:51
tags:
- swoole
- PHP
---

小伙伴们大家好，这篇博客教大家如何开发一个服务器扩展雏形。

我们的环境如下：

```
操作系统：alpine（小伙伴们如果不是这个Linux发行版的话也没事）
PHP版本：7.2.10
```

## 初始化目录结构

PHP官方给我们提供了构建PHP扩展的一个工具`ext_skel`，我们可以在PHP的源码（你可以在PHP的官网得到PHP源码）里面找到。

我们找到这个工具之后，执行命令：

```shell
./ext_skel --extname=tinyswoole
cd tinyswoole
```

然后我们执行一下tree命令查看一下生成的目录结构：

```shell
tree
.
├── CREDITS
├── EXPERIMENTAL
├── config.m4
├── config.w32
├── tinyswoole.c
├── tinyswoole.php
├── php_tinyswoole.h
└── tests
    └── 001.phpt

1 directory, 8 files
```

（OK，文件很多，但是我们只需要关注一部分即可）

我们首先在`tinyswoole`目录下面创建一个`src`目录用来存放我们开发的扩展源码。

```shell
mkdir src
touch src/server.c
```

编辑一下`config.m4`文件，把这个文件的内容替换为如下内容：

```
PHP_ARG_ENABLE(tinyswoole, whether to enable tinyswoole support,
dnl Make sure that the comment is aligned:
[  --enable-tinyswoole           Enable tinyswoole support])

if test "$PHP_TINYSWOOLE" != "no"; then
  PHP_SUBST(TINYSWOOLE_SHARED_LIBADD)
  source_file="tinyswoole.c tinyswoole_client.c src/client.c src/server.c"
  PHP_NEW_EXTENSION(tinyswoole, $source_file, $ext_shared,, -DZEND_ENABLE_STATIC_TSRMLS_CACHE=1)
fi
```

我这里说一点，`source_file`这个变量里面的文件直接使用空格隔开的，不要用逗号哈。

OK，到了这一步，我们的准备工作已经做完了。我们可以开始开发服务器扩展了。

## 开发扩展

我们编写我们的`src/server.c`文件：

```shell
vim src/server.c
```

然后，把这个文件的内容替换为如下内容：

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>

#define LISTENQ 10
#define MAX_BUF_SIZE 1024


int server(char *ip, int port)
{
    int listenfd;
    int connfd;
    int n;
    socklen_t len;
    struct sockaddr_in servaddr;
    struct sockaddr_in cliaddr;
    char buffer[MAX_BUF_SIZE];

    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    inet_aton(ip, &(servaddr.sin_addr));
    servaddr.sin_port = htons(port);
    bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));
    listen(listenfd, LISTENQ);

    for (;;) {
        len = sizeof(cliaddr);
        connfd = accept(listenfd, (struct sockaddr *)&cliaddr, &len);
        for (;;) {
            n = read(connfd, buffer, MAX_BUF_SIZE);
            if (n <= 0) {
                close(connfd);
                break;
            }
            write(connfd, buffer, n);  
        }
    }

    close(listenfd);
}

```

通过之前我对swoole服务器的讲解，小伙伴们应该是可以读懂这个代码的。这里我解释一下。

1、创建一个监听socket

```c
listenfd = socket(AF_INET, SOCK_STREAM, 0);
```

2、指定服务器绑定的端口

```c
bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));
```

3、监听客户端的连接

```
listen(listenfd, LISTENQ);
```

4、获取一个连接

```c
connfd = accept(listenfd, (struct sockaddr *)&cliaddr, &len);
```

5、读取客户端发来的数据

```c
n = read(connfd, buffer, MAX_BUF_SIZE);
```

6、返回客户端数据

```c
write(connfd, buffer, n);
```

所以，这个服务器做的事情很简单，客户端发给服务器什么数据，服务器就原样返回给客户端。

## 注册这个server函数

然后，我们编辑`tinyswoole.c`文件：

```shell
vim tinyswoole.c
```

把里面的内容替换为如下内容：

```c
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "php.h"
#include "php_ini.h"
#include "ext/standard/info.h"
#include "php_tinyswoole.h"

zend_class_entry *tinyswoole_ce;


ZEND_BEGIN_ARG_INFO_EX(arginfo_tinyswoole__construct, 0, 0, 2)
	ZEND_ARG_INFO(0, ip)
    ZEND_ARG_INFO(0, port)
ZEND_END_ARG_INFO()

static inline zval* tsw_zend_read_property(zend_class_entry *class_ptr, zval *obj, const char *s, int len, int silent)
{
    zval rv;
    return zend_read_property(class_ptr, obj, s, len, silent, &rv);
}

PHP_METHOD(tinyswoole_server, __construct)
{
	char *ip;
	size_t ip_len;
	long port;
	
	if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "sl", &ip, &ip_len, &port) == FAILURE) {
		RETURN_NULL();
	}

	zend_update_property_string(tinyswoole_ce, getThis(), "ip", sizeof("ip") - 1, ip);
	zend_update_property_long(tinyswoole_ce, getThis(), "port", sizeof("port") - 1, port);
}

PHP_METHOD(tinyswoole, start)
{
	zval *ip;
	zval *port;

	printf("running server...\n");

	ip = tsw_zend_read_property(tinyswoole_ce, getThis(), "ip", sizeof("ip") - 1, 0);
	port = tsw_zend_read_property(tinyswoole_ce, getThis(), "port", sizeof("port") - 1, 0);

	server(Z_STRVAL(*ip), Z_LVAL(*port));
}

zend_function_entry tinyswoole_method[]=
{
	ZEND_ME(tinyswoole_server, __construct, arginfo_tinyswoole__construct, ZEND_ACC_PUBLIC)
	ZEND_ME(tinyswoole_server, start, NULL, ZEND_ACC_PUBLIC)
	{NULL, NULL, NULL}
};

PHP_MINIT_FUNCTION(tinyswoole)
{
	zend_class_entry ce;
	INIT_CLASS_ENTRY(ce, "tinyswoole_server", tinyswoole_method);
	tinyswoole_ce = zend_register_internal_class(&ce TSRMLS_CC);

	zend_declare_property_null(tinyswoole_ce, "ip", sizeof("ip") - 1, ZEND_ACC_PRIVATE);
	zend_declare_property_null(tinyswoole_ce, "port", sizeof("port") - 1, ZEND_ACC_PRIVATE);

	return SUCCESS;
}

PHP_MSHUTDOWN_FUNCTION(tinyswoole)
{
	return SUCCESS;
}

PHP_RINIT_FUNCTION(tinyswoole)
{
#if defined(COMPILE_DL_TINYSWOOLE) && defined(ZTS)
	ZEND_TSRMLS_CACHE_UPDATE();
#endif
	return SUCCESS;
}

PHP_RSHUTDOWN_FUNCTION(tinyswoole)
{
	return SUCCESS;
}

PHP_MINFO_FUNCTION(tinyswoole)
{
	php_info_print_table_start();
	php_info_print_table_header(2, "tinyswoole support", "enabled");
	php_info_print_table_end();
}

const zend_function_entry tinyswoole_functions[] = {
	PHP_FE_END
};

zend_module_entry tinyswoole_module_entry = {
	STANDARD_MODULE_HEADER,
	"tinyswoole",
	tinyswoole_functions,
	PHP_MINIT(tinyswoole),
	PHP_MSHUTDOWN(tinyswoole),
	PHP_RINIT(tinyswoole),
	PHP_RSHUTDOWN(tinyswoole),
	PHP_MINFO(tinyswoole),
	PHP_TINYSWOOLE_VERSION,
	STANDARD_MODULE_PROPERTIES
};

#ifdef COMPILE_DL_TINYSWOOLE
#ifdef ZTS
ZEND_TSRMLS_CACHE_DEFINE()
#endif
ZEND_GET_MODULE(tinyswoole)
#endif

```

## 测试扩展

然后，我们编译这个扩展：

```shell
phpize
./configure
make
make install
```

然后，我们在php.ini配置文件里面加上这个扩展：

```
extension=tinyswoole
```

然后，我们编写一个脚本来测试一下这个扩展：

```php
<?php

$serv = new tinyswoole_server('127.0.0.1', 9501);
$serv->start();
```

执行这个脚本，你将会得到如下输出：

```shell
php server.php 
running server...
```

然后，我们用一个客户端去连接我们写好的服务器。可以使用nc命令去连接它：

```shell
nc 127.0.0.1 9501
```

然后，输入字符串`hello world`，你将会收到服务器发回给你的`hello world`：

```shell
nc 127.0.0.1 9501
hello world
hello world
```

OK，到了这里，我们的服务器扩展算是开发完成了，尽管他的功能很弱。但是，随着我们的学习，这个扩展会一步一步的变得更加的强大。







