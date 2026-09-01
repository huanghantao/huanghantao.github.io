---
title: Dockerfile中WORKDIR的工作原理
date: 2019-08-29 16:54:03
tags:
- Docker
- Dockerfile
- docker-compose
---

今天，同事问到我一个问题，就是`Dockerfile`中的`WORKDIR`怎么工作的？说实话，刚开始我还真没思考过。我只是想当然的认为`Dockerfile`里面设置了`WORKDIR`，然后启动容器的时候当然就会进入这个目录了。

但是，我们知道`docker-compose`也有一个类似的配置：

```
working_dir
```

那么，当我们在`Dockerfile`里面设置了`WORKDIR`，又在`docker-compose`里面设置了`working_dir`，那么，为什么我们创建一个容器的时候，进入的就是`working_dir`里面配置的那个目录呢？

我们实战分析一下，我们有如下`Dockerfile`：

```dockerfile
FROM alpine:3.8

WORKDIR /root
```

很简单，就是以`alpine:3.8`为基础镜像，然后配置`WORKDIR`为`/root`。然后，我们编写对应的`docker-compose.yml`文件：

```yaml
version: '3'

services:
  test:
    build: .
    image: test
    container_name: test
    working_dir: /tmp
    restart: always
    command: /sbin/init
```

我们配置`working_dir`为`/tmp`，而不是`Dockerfile`中的`/root`。

现在，我们来编译一下进行：

```shell
hantaohuang@~/tmp/test docker-compose build
Building test
Step 1/2 : FROM alpine:3.8
 ---> dac705114996
Step 2/2 : WORKDIR /root
 ---> Running in 463b6fa13304
Removing intermediate container 463b6fa13304
 ---> f5ae5eef35b3
Successfully built f5ae5eef35b3
Successfully tagged test:latest
hantaohuang@~/tmp/test 
```

首先，我理解`build`的过程实际上是起一个临时容器，然后在容器里面跑对应的指令，然后再把这个容器的`aufs` `merged`层`commit`成一个镜像，不断的重复这个过程，直到`commit`出最后一个镜像，即我最终需要的镜像`test`。

我思考到了这一步，发现我就产生了一个问题：

```
在之前的临时容器里面执行的指令WORKDIR为什么会作用到我最终的镜像？
```

我在内网里面提了这个问题，有大佬解答说镜像里面除了基本的文件系统之外，还有一些元数据，我们可以通过`inspect`命令来查看。我们来看看镜像的这些数据：

```shell
hantaohuang@~/tmp/test docker inspect f5ae5eef35b3
[
    {
        "Id": "sha256:f5ae5eef35b36914df42d3170a800f6461b06b2f6fa600a539806fb9216b6b0f",
        "RepoTags": [
            "test:latest"
        ],
        "RepoDigests": [],
        "Parent": "sha256:dac7051149965716b0acdcab16380b5f4ab6f2a1565c86ed5f651e954d1e615c",
        "Comment": "",
        "Created": "2019-08-29T09:02:15.96601Z",
        "Container": "463b6fa1330400120536ffb408b3ec5797077cdcaa7249ffc353c4e6fe8bcbb0",
        "ContainerConfig": {
            "Hostname": "463b6fa13304",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/bin/sh",
                "-c",
                "#(nop) WORKDIR /root"
            ],
            "ArgsEscaped": true,
            "Image": "sha256:dac7051149965716b0acdcab16380b5f4ab6f2a1565c86ed5f651e954d1e615c",
            "Volumes": null,
            "WorkingDir": "/root",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {}
        },
    }
]
```

我们发现，在`ContainerConfig`中有一项`WorkingDir`就记录了我们在`Dockerfile`里面设置的工作目录。所以，当我们通过这个镜像创建容器的时候，我们会进入`WorkingDir`指定的路径，也就是`/root`。我们来测试一下：

```shell
hantaohuang@~/tmp/test docker run --rm -it f5ae5eef35b3 sh
~ # pwd
/root
~ # 
```

符合预期。

然后，如果我们通过`docker-compose`起一个容器：

```shell
~ # exit
hantaohuang@~/tmp/test docker-compose up -d
Creating network "test_default" with the default driver
Creating test ... 
Creating test ... done
hantaohuang@~/tmp/test 
```

我们查看一下容器的`id`：

```shell
hantaohuang@~/tmp/test docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
078a7370c21d        test                "/sbin/init"             17 seconds ago      Up 16 seconds                                test
e53716f972ec        php_alpine          "docker-php-entrypoi…"   31 hours ago        Up 31 hours         0.0.0.0:8000->8000/tcp   php
564c95acc209        gdb_swoole_alpine   "docker-php-entrypoi…"   2 weeks ago         Up 8 days           0.0.0.0:9501->9501/tcp   gdb_swoole_alpine
8be291961531        golang:alpine       "/sbin/init"             3 weeks ago         Up 8 days           0.0.0.0:8080->8080/tcp   go
hantaohuang@~/tmp/test 
```

然后进入容器：

```shell
hantaohuang@~/tmp/test docker exec -it test sh
/tmp # 
```

我们发现，进入的是`/tmp`目录而不是`/root`目录。为什么呢？因为我们的容器也是有一份和镜像类似的数据，我们可以查看一下`test`容器的信息：

```shell
hantaohuang@~/tmp/test docker inspect 078a7370c21d
[
    {
        "Id": "078a7370c21d453a4e59dab57c4a3ec4752d0535616a9200350c56ef91bc73e8",
        "Created": "2019-08-29T09:10:41.5564366Z",
        "Path": "/sbin/init",
        "Args": [],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 14784,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2019-08-29T09:10:42.1561139Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
        "Image": "sha256:f5ae5eef35b36914df42d3170a800f6461b06b2f6fa600a539806fb9216b6b0f",
        "ResolvConfPath": "/var/lib/docker/containers/078a7370c21d453a4e59dab57c4a3ec4752d0535616a9200350c56ef91bc73e8/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/078a7370c21d453a4e59dab57c4a3ec4752d0535616a9200350c56ef91bc73e8/hostname",
        "HostsPath": "/var/lib/docker/containers/078a7370c21d453a4e59dab57c4a3ec4752d0535616a9200350c56ef91bc73e8/hosts",
        "LogPath": "/var/lib/docker/containers/078a7370c21d453a4e59dab57c4a3ec4752d0535616a9200350c56ef91bc73e8/078a7370c21d453a4e59dab57c4a3ec4752d0535616a9200350c56ef91bc73e8-json.log",
        "Name": "/test",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "",
        "ExecIDs": null,
        "Mounts": [],
        "Config": {
            "Hostname": "078a7370c21d",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/sbin/init"
            ],
            "Image": "test",
            "Volumes": null,
            "WorkingDir": "/tmp",
            "Entrypoint": null,
            "OnBuild": null,
        },
    }
]
```

记录的信息比镜像多得多（我还省略了其他很多信息）。我们会发现，容器也是有一个`Config.WorkingDir`来进入进入容器的时候，应该到什么目录里面。

所以，我们可以得出结论，通过命令：

```shell
docker run 
```

起一个容器的话（如果不指定工作目录），那么容器`inspect`出来的信息里面的`workdir`是和镜像`inspect`出来的信息里面的`workdir`一致。

如果我们在容器启动的时候指定了`workdir`，那么容器`inspect`出来的信息里面的`workdir`就和镜像`inspect`出来的不一致。

然后，当我们进入一个处于运行状态的容器里面的时候，也就是执行命令`docker exec`的时候，就会去读取容器的那个`workdir`。因此，我们通过`docker-compose`管理的`workdir`实际上是容器的那个`workdir`而不是镜像的那个`wordir`。