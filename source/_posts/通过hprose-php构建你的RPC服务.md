---
title: 通过hprose-php构建你的RPC服务
date: 2019-04-03 20:58:09
tags:
- PHP
- RPC
---

RPC的概念我们不过多的解释，我们直接来演示一下如何使用`hprose-php`构建RPC服务。

我们通过`composer`来安装`hprose-php`。项目初始化：

```shell
mkdir rpc-server && cd rpc-server && touch composer.json && echo '{
    "require": {
        "hprose/hprose": ">=2.0.0"
    }
}' > composer.json && composer update
```

在rpc-server目录下创建文件`callee.php`，内容如下：

```php
<?php

require_once 'vendor/autoload.php';

use Hprose\Http\Server;

function say($words)
{
    return $words;
}

$server = new Server();

$server->addFunction('say');
$server->start();
```

然后在rpc-server目录下执行命令：

```shell
~/codeDir/phpCode/rpc-server # php -S localhost:80
PHP 7.2.14 Development Server started at Wed Apr  3 13:33:11 2019
Listening on http://localhost:80
Document root is /root/codeDir/phpCode/rpc-server
Press Ctrl-C to quit.
```

然后在rpc-server目录下创建文件`caller.php`，内容如下：

```php
<?php

require_once 'vendor/autoload.php';

use Hprose\Http\Client;
use Hprose\InvokeSettings;
use Hprose\ResultMode;

$client = new Client('localhost/callee.php', false);
var_dump($client->say('hello world'));
```

然后执行脚本：

```shell
~/codeDir/phpCode/rpc-server # php caller.php
string(11) "hello world"
```

发现，调用RPC服务的`say`方法成功。

