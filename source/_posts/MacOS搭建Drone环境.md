---
title: MacOS搭建Drone环境
date: 2021-05-23 16:40:53
tags:
- CI/CD
- Drone
---

最近想在自己机器上面搭建一个`CI/CD`系统，发现`Drone`比较好用。（之前尝试过`gitlab`，这个东西太占电脑的资源了，运行几分钟，就报资源不足了）

这篇文章是基于`MacOS`搭建的，因为`Linux`上面搭建网络问题比较好解决，所以就不给出`Linux`下的模板了。

这里就直接给出`docker-compose.yml`的配置：

```yml
version: '3'

services:
  gitea:
    image: gitea/gitea:latest
    ports:
      - "3000:3000"
      - "22:22"
    volumes: 
      - ./gitea-data:/data

  # 容器名称
  drone_server:
    # 构建所使用的镜像
    image: drone/drone:latest
    container_name: drone_server
    # 映射容器内80端口到宿主机的7079端口
    ports:
      - 8080:80
    # 容器随docker自动启动
    restart: always
    depends_on: 
      - gitea
    environment:
      # Gitea 服务器地址
      - DRONE_GITEA_SERVER=http://host.docker.internal:3000
      # Gitea OAuth2客户端ID
      - DRONE_GITEA_CLIENT_ID=48515522-4a19-4d5b-86e6-2a1269338829
      # Gitea OAuth2客户端密钥
      - DRONE_GITEA_CLIENT_SECRET=Z02uD6kHYerX8FnZY2gRUDpJx3IqTHTriIVuZsVpuGNe
      # drone的共享密钥
      - DRONE_RPC_SECRET=7a5fe2cad36b1d69f443c9aad4761bbe
      # drone的主机名
      - DRONE_SERVER_HOST=host.docker.internal:8080
      # 外部协议方案
      - DRONE_SERVER_PROTO=http
      # 创建管理员账户，这里对应为gitea的用户名
      - DRONE_USER_CREATE=username:codinghuang,admin:true
      - DRONE_GITEA_SKIP_VERIFY=true
      - DRONE_LOGS_TRACE=true
      - DRONE_AGENTS_ENABLED=true
      - DRONE_COOKIE_SECRET=correct-horse-battery-staple
      - DRONE_GIT_ALWAYS_AUTH=true

  drone_runner:
    image: drone/drone-runner-docker:latest
    container_name: drone_runner
    ports:
      - 7080:3000
    restart: always
    depends_on:
      - drone_server
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      # 用于连接到Drone服务器的协议。该值必须是http或https。
      - DRONE_RPC_PROTO=http
      # 用于连接到Drone服务器的主机名
      - DRONE_RPC_HOST=host.docker.internal:8080
      # Drone服务器进行身份验证的共享密钥，和上面设置一样
      - DRONE_RPC_SECRET=7a5fe2cad36b1d69f443c9aad4761bbe
      # 限制运行程序可以执行的并发管道数。运行程序默认情况下执行2个并发管道。
      - DRONE_RUNNER_CAPACITY=2
      # docker runner 名称
      - DRONE_RUNNER_NAME=drone-runner-1
      - DRONE_LOGS_TRACE=true
```

然后按照以下步骤即可完成搭建。

## 配置hosts

```hosts
127.0.0.1 host.docker.internal
```

## 使用方法

### 配置gitea

#### 创建容器

```bash
docker-compose up -d gitea
```

#### 初始化gitea网站信息

然后访问`gitea`[网站](http://host.docker.internal:3000/)

此时会进入`gitea`的初始化界面，我们需要把`localhost`全部换成`host.docker.internal`，然后确认配置即可。如果点击确认后跳转到登陆界面失败，那么我们可以手动在浏览器里面输入`http://host.docker.internal:3000/`访问`gitea`网站。接着，注册一个用户即可。

#### 授权的 OAuth2 应用

点击 "设置" -> "应用"。然后在 "管理 OAuth2 应用程序" 这一栏里面填写"应用名称"和"重定向 URI"，其中"应用名称"可以随便填，"重定向 URI"必须填写：

```bash
http://host.docker.internal:8080/login
```

填写完点击确认之后，会得到 "客户端ID" 和 "客户端密钥"，这两个得记下来，然后分别填写到`docker-compose.yml`里面的`DRONE_GITEA_CLIENT_ID`和`DRONE_GITEA_CLIENT_SECRET`这两个环境变量里面。

### 启动drone服务

```bash
docker-compose up -d
```

接着，访问`Drone`的管理[网站](http://host.docker.internal:8080)，第一次进入需要授权，点击授权，输入刚才注册的`gitea`用户名和密码。

然后，在`gitea`里面随便上传一个项目，然后刷新`Drone`网站，将会发现项目被同步到里面来了。
