---
title: PHP实现GRPC -- 传输层使用Protobuf进行数据传输
date: 2020-03-18 18:52:03
tags:
- PHP
- Protobuf
- GRPC
- Swoole
---

我们本章的目标是通过`Protobuf`让客户端和服务器进行通信，需要用到`Swoole`扩展，版本是`v4.4.16`。

我们安装`Swoole`官方的`IDE helper`：

```bash
[root@592b0366acbf grpc-demo]# composer require swoole/ide-helper --dev
```

我们现在创建目录`RPC`：

```bash
[root@592b0366acbf grpc-demo]# mkdir RPC
```

然后再创建文件`RPC/Server.php`：

```bash
[root@592b0366acbf grpc-demo]# touch RPC/Server.php
```

内容如下：

```php
<?php

namespace RPC;

use Message\User;
use Swoole\Coroutine\Server as CoroutineServer;
use Swoole\Coroutine\Server\Connection;

class Server extends CoroutineServer
{
    public function __construct($host, $port)
    {
        parent::__construct($host, $port);
    }

    public function handler(Connection $conn)
    {
        while(true) {
            $data = $conn->recv();
            if ($data === '') {
                echo 'the client is disconnected' . PHP_EOL;
                break;
            } else if ($data === false) {
                echo socket_strerror($this->errCode);
            }
            $user = new User();
            $user->mergeFromString($data);
            echo $user->getId() . PHP_EOL;
            echo $user->getAge() . PHP_EOL;
            echo $user->getUsername() . PHP_EOL;
        }
    }

    public function start(): bool
    {
        $this->handle([$this, 'handler']);
        parent::start();
        return true;
    }
}
```

然后我们为`composer.json`增加对应的`autoload`：

```json
"autoload": {
    "psr-4": {
        "Message\\": "Protobuf/Message",
        "GPBMetadata\\": "Protobuf/GPBMetadata",
        "RPC\\": "RPC"
    }
}
```

我们为`RPC`命名空间增加了一项。

接着执行命令：

```bash
[root@592b0366acbf grpc-demo]# composer dump-autoload
```

然后创建文件``
