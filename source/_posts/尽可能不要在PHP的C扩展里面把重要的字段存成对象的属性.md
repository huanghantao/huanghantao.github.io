---
title: 尽可能不要在PHP的C扩展里面把重要的字段存成对象的属性
date: 2020-08-18 20:58:09
tags:
- PHP
---

我们有如下例子：

```php
<?php

use Swoole\Server;

$serv = new Server('127.0.0.1', 9580);

$serv->on('Receive', function ($serv, $fd, $reactorId, $data) {
    array_walk($serv, function (&$property) {
        if (isset($property[0]) && $property[0] instanceof Swoole\Server\Port) {
            $primaryPort = $property[0];
            array_walk($primaryPort, function (&$callback) {
                $callback = null;
            });
        }
    });
});

$serv->start();
```

这个代码看起来会非常的绕，但是，为了解释我们的问题，这个写法还是很具有代表性的。

因为`PHP`的设计问题，我们可以在类的外面通过`array_walk`来访问一个对象的私有属性，并且修改它。我们来看一下`Swoole\Server`底层是如何存成`port`的：

```cpp
zend_declare_property_null(swoole_server_port_ce, ZEND_STRL("onConnect"), ZEND_ACC_PRIVATE);
zend_declare_property_null(swoole_server_port_ce, ZEND_STRL("onReceive"), ZEND_ACC_PRIVATE);
zend_declare_property_null(swoole_server_port_ce, ZEND_STRL("onClose"), ZEND_ACC_PRIVATE);
zend_declare_property_null(swoole_server_port_ce, ZEND_STRL("onPacket"), ZEND_ACC_PRIVATE);
zend_declare_property_null(swoole_server_port_ce, ZEND_STRL("onBufferFull"), ZEND_ACC_PRIVATE);
zend_declare_property_null(swoole_server_port_ce, ZEND_STRL("onBufferEmpty"), ZEND_ACC_PRIVATE);
zend_declare_property_null(swoole_server_port_ce, ZEND_STRL("onRequest"), ZEND_ACC_PRIVATE);
zend_declare_property_null(swoole_server_port_ce, ZEND_STRL("onHandShake"), ZEND_ACC_PRIVATE);
zend_declare_property_null(swoole_server_port_ce, ZEND_STRL("onOpen"), ZEND_ACC_PRIVATE);
zend_declare_property_null(swoole_server_port_ce, ZEND_STRL("onMessage"), ZEND_ACC_PRIVATE);
```

我们发现，这里把回调函数设置成了`private`属性，但是终究是无法避免被修改的下场。

只要我们跑这个服务器，并且给这个服务器发送数据。那么，我们就可以让这个`Server coredump`。这是我的测试结果：

```bash
PHP Fatal error:  Uncaught Error: Cannot use object of type Swoole\Server as array in /Users/hantaohuang/codeDir/phpCode/library/test.php:10
Stack trace:
#0 {main}
  thrown in /Users/hantaohuang/codeDir/phpCode/library/test.php on line 10

Fatal error: Uncaught Error: Cannot use object of type Swoole\Server as array in /Users/hantaohuang/codeDir/phpCode/library/test.php:10
Stack trace:
#0 {main}
  thrown in /Users/hantaohuang/codeDir/phpCode/library/test.php on line 10
[2020-08-18 21:16:56 *13285.3]  ERROR   php_swoole_server_rshutdown (ERRNO 503): Fatal error: Uncaught Error: Cannot use object of type Swoole\Server as array in /Users/hantaohuang/codeDir/phpCode/library/test.php:10
Stack trace:
#0 {main}
  thrown in /Users/hantaohuang/codeDir/phpCode/library/test.php on line 10
[2020-08-18 21:16:56 $12938.0]  WARNING check_worker_exit_status: worker#3[pid=13285] abnormal exit, status=255, signal=0
```

我们可以看到，`worker`进程挂了。
