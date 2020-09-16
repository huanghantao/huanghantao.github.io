---
title: vscode调试swoole源码
date: 2020-01-14 18:39:08
tags:
- Swoole
- 调试
---

> 环境：Mac OS X
>
> 搭建环境工具：Docker

思路是通过`Docker`起一个`Linux`容器，然后通过`vscode`的`Remote-SSH`插件登陆进这个`Linux`容器。这样就可以在`Mac OS X`或者`Windows`下调试和`Linux`系统相关的代码了。

首先，`git clone`下我准备好的`Dockerfile`：

```shell
git clone git@github.com:dockero/php_centos.git
```

`php_centos`的目录结构如下：

```shell
tree
.
├── Dockerfile
├── README.md
├── docker-compose.yml
└── etc
    └── php.d
        ├── curl.ini
        ├── openssl.ini
        ├── swoole.ini
        ├── zip.ini
        └── zlib.ini

2 directories, 8 files
```

其中，

`Dockerfile`用来构建镜像的，已经预先安装了`php`、`swoole`以及其他基础的扩展。

`docker-compose.yml`主要是在创建容器的时候设置一些环境变量。

`etc`目录则是存放`php`扩展的`.ini`文件。这里，我们一个扩展对应一个`.ini`文件。

除此之外，你在目录`php_centos`下需要自己创建一个`.env`文件，用来控制`swoole`的版本，以及设置公钥等。

这里是我的一份模板：

```shell
HTTP_PROXY=http://127.0.0.1:8080
HTTPS_PROXY=http://127.0.0.1:8080
CODEDIR_VOLUME=~/codeDir:/root/codeDir
HOST_SSH_PORT=9522
PASSWORD=123456
SWOOLE_VERSION=4.4.12
SSH_PUB_KEY=填写你的公钥
```

每个参数在`README`里面都有说明。

`OK`，我们编译镜像：

```shell
docker-compose build
```

然后启动容器：

```shell
docker-compose up -d
```

之后，我们执行命令：

```shell
ssh php
```

即可登陆容器了。（如果在登陆容器的时候，需要输入密码，你可以执行命令`ssh-add 私钥`来免密登陆）

然后，我们需要在`vscode`的同一个工作区里面存放`PHP`源码、`Swoole`源码、`PHP`测试脚本、`.vscode`目录。所有需要的东西如下：

```shell
tree -L 1
.
├── test.php
├── php-7.3.12
├── swoole-src
└── .vscode
```

我们在`.vscode`目录里面创建文件`launch.json`：

```json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "debug swoole",
            "type": "cppdbg",
            "request": "launch",
            "program": "/usr/bin/php",
            "args": ["${file}"],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
```

接下来，我们用`vscode`打开测试脚本`test.php`，并且停留在这个窗口（注意，必须在停留`test.php`窗口的时候点击调试才行），然后点击调试按钮即可进行`Swoole`源码的调试了。
