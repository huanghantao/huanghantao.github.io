---
title: 使用golang alpine镜像编译go文件执行报错no such file or directory
date: 2021-09-22 22:05:55
tags:
- Go
---

编译的命令如下：

```bash
docker run --rm -it -v $(pwd):$(pwd) -w $(pwd) -e GOPROXY=https://goproxy.cn golang:1.14.4-alpine3.12 go build main.go
```

执行：

```bash
no such file or directory: ./main
```

这个时候就很懵了，因为这个`main`文件是有可执行权限的。所以，这个`file`一定就不是指`main`文件了。分析后，猜测是它依赖的动态链接库不存在：

```bash
ldd main
    linux-vdso.so.1 (0x00007ffe19b80000)
    libc.musl-x86_64.so.1 => not found
```

果然，发现`musl`库不存在。`alpine`默认用的是`musl`库，并且，`go`编译的时候，默认会开启`CGO`，这个时候就会用到`libc`之类的库。所以，这里我们可以暂时关闭`CGO`，让编译通过：

```bash
docker run --rm -it -v $(pwd):$(pwd) -w $(pwd) -e CGO_ENABLED=0 -e GOPROXY=https://goproxy.cn golang:1.14.4-alpine3.12 go build main.go
```

当然，我们最好还是用和本机系统一样的`docker`环境来进行编译，这样，就可以正确的链接`C`库了。
