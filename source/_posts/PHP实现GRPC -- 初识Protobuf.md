---
title: PHP实现GRPC -- 初识Protobuf
date: 2020-03-18 16:35:18
tags:
- PHP
- Protobuf
- GRPC
---

需要提前准备好`protoc`这个工具，我们可以在[github](https://github.com/grpc/grpc)上面下载源码然后进行编译安装：

```bash
git clone https://github.com/grpc/grpc.git

cd grpc

make grpc_php_plugin
```

因为编译安装的过程会因为每个人的编译工具的不同而不同，且这个过程可能会遇到很多的问题，所以安装工具的过程就不多展开来说了。（编译的过程中如果遇到问题，可以在评论区进行提问）

`OK`，我们现在开始使用`Protobuf`。

首先，创建目录`grpc-demo`并且进入目录`grpc-demo`：

```bash
mkdir grpc-demo

cd grpc-demo/
```

然后我们通过`composer`来初始化我们的项目：

```bash
[root@592b0366acbf grpc-demo]# composer init
```

（出现了交互的信息直接回车就行）

然后安装`google/protobuf`依赖：

```bash
[root@592b0366acbf grpc-demo]# composer require google/protobuf
```

然后，我们编写我们的协议（文件名字是`user.proto`）：

```proto
syntax = "proto3";

package message;

message User {
    uint32 id = 1;
    string username = 2;
    uint32 age = 3;
}
```

`OK`，现在我们就可以通过`protoc`工具来生成对应的协议解析代码了：

```bash
[root@592b0366acbf grpc-demo]# mkdir Protobuf

[root@592b0366acbf grpc-demo]# protoc --php_out=Protobuf/ user.proto
```

执行完这条命令之后，`Protobuf`目录下面就会生成协议解析的代码了。

然后我们为`composer.json`增加`autoload`：

```json
"autoload": {
    "psr-4": {
        "Message\\": "Protobuf/Message",
        "GPBMetadata\\": "Protobuf/GPBMetadata"
    }
}
```

然后执行命令：

```bash
[root@592b0366acbf grpc-demo]# composer dump-autoload
```

接着，我们创建测试脚本`index.php`：

```bash
[root@592b0366acbf grpc-demo]# touch index.php
```

然后文件内容为：

```php
<?php

use Message\User;

require __DIR__ . '/vendor/autoload.php';

$user1 = new User();
$user2 = new User();

$user1->setId(10086);
$user1->setUsername('codinghuang');
$user1->setAge(22);
$data = $user1->serializeToString();
var_dump($data);

$user2->mergeFromString($data);
var_dump($user2->getId());
var_dump($user2->getUsername());
var_dump($user2->getAge());
```

执行结果如下：

```bash
[root@592b0366acbf grpc-demo]# php index.php
string(18) �N
             codinghuang"
int(10086)
string(11) "codinghuang"
int(22)
```
