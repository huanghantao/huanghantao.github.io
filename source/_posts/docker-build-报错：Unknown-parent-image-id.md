---
title: 当Docker遇上傻CI
date: 2019-06-13 16:58:11
tags:
- Docker
---

昨天下午在用公司`ci`进行`docker build`的时候，报了一个错误，内容大致如下：

```shell
invalid from flag value ***: No such image: sha256:123456123456*************
```

说的是一个`image`没有找到。

其中，第一个`Dockerfile_1`结构如下：

```dockerfile
FROM composer AS builder

// 省略一些东西......
COPY 一些东西 构件1  // 产生了一个layer，假设id为：123456123456
```

第二个`Dockerfile_2`结构如下：

```dockerfile
FROM composer AS builder

COPY 一些东西 构件1 // 用的是第一个Dockerfile_1中的cache：123456123456

FROM nginx:alpine

COPY --from=builder 构件1 到某个目录 // 报错点就是这句，此时报错123456123456没找到
```

编译：

```shell
docker build -f dockerfile_1 .
docker build -f dockerfile_2 .
```

然后报错：

```shell
invalid from flag value ***: No such image: sha256:3490ffda0
```

这个问题很奇葩，几乎不会出现，但是，在使用公司的`ci`的时候就有可能报这个错。因为`ci`发现第一个`Dockerfile_1`编译出的`image`存留太久了就会把它删掉。。。。。。然后我们优化了第二个`Dockerfile_2`解决了这个问题。