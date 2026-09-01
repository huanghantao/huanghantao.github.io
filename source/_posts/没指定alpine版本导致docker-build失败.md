---
title: 没指定alpine版本导致docker build失败
date: 2019-06-13 16:04:10
tags:
- Docker
---

这两天因为这件事情折腾了比较久，浪费了比较多的时间，写下这篇文章提醒自己和大家。

情况是这样的，5个月前，我们写了一份Dockerfile，主要内容是这样的：

```
FROM php:7.2-cli-alpine

RUN echo "http://mirrors.ustc.edu.cn/alpine/v3.8/main/" > /etc/apk/repositories \
        && echo "http://mirrors.ustc.edu.cn/alpine/v3.8/community/" >> /etc/apk/repositories

RUN apk add --no-cache --virtual .phpize-deps $PHPIZE_DEPS linux-headers libpng-dev && \
    apk add --no-cache libpng libstdc++ && \
    pecl install swoole && \
    docker-php-ext-install \
        pdo_mysql \
        mysqli \
        sockets \
        pcntl \
        gd \
        zip && \
    docker-php-ext-enable swoole && \
    apk del .phpize-deps
```

5个月前我用这份Dockerfile编译了一次，可以编译成功。

![img](https://pic3.zhimg.com/80/v2-87893d34a05977797a2157c24eacc821_hd.png)



（可以看出，基础镜像是5个月前的）

那个时候，php:7.2-cli-alpine的alpine的版本还是 3.8 。

但是，前些日子在同事的电脑上用这份Dockerfile编译就过不了了，报了一个错误：

![img](https://pic2.zhimg.com/80/v2-cb92dd90f9bdab48eba11d071e76c0b0_hd.png)

说是一个函数没有找到，一开始，我一直以为是PHP的一个bug，但是一直没解决（事实上并不是PHP的bug）。 

但是，在我的电脑上就可以编译通过。（一开始觉得这件事非常的神奇）

尝试解决也没有解决成功，也没有问组里的大佬，所以，别人要编译的话，就拿我的机器进行编译。。。。。。

今天下午，和大佬说了一下，大佬一眼看出了问题（在别人的电脑进行编译的）：

![img](https://pic1.zhimg.com/80/v2-3220469a173c90fe8e543d00c3e51144_hd.png)

发现，这里拉的是alpine 3.8 的包。因为，我们的Dockerfile里面有如下内容：

```
RUN echo "http://mirrors.ustc.edu.cn/alpine/v3.8/main/" > /etc/apk/repositories \
        && echo "http://mirrors.ustc.edu.cn/alpine/v3.8/community/" >> /etc/apk/repositories
```

用的是alpine3.8。 

但是，现在的alpine已经更新到了3.9。而我们的Dockerfile里面 FROM php:7.2-cli-alpine 没有指定alpine的版本，那么基础镜像默认用的是最新的3.9的版本，所以就报错了。 

那么，为什么在同事的电脑用这份Dockerfile不可以进行编译，在我的 电脑上就可以呢。因为我们电脑上的php:7.2-cli-alpine镜像一直没有删掉，所以，alpine的版本一直是5个月前的3.8。。。。。。 

