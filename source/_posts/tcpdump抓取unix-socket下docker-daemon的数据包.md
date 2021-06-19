---
title: tcpdump抓取unix socket下docker daemon的数据包
date: 2021-06-19 12:08:45
tags:
- Docker
---

工作原理如下：

```bash
unix socket -> proxy -> unix socket
```

当我们给`unix socket`发送数据的时候，数据包就会经过我们的代理，我们只需要在代理处抓包即可。

按照以下步骤执行：

```bash
sudo mv /var/run/docker.sock /var/run/docker.sock.copy
sudo socat TCP-LISTEN:8080,reuseaddr,fork UNIX-CONNECT:/var/run/docker.sock.copy
sudo socat UNIX-LISTEN:/var/run/docker.sock,fork TCP-CONNECT:127.0.0.1:8080
```

然后，我们尝试访问下`docker api`是否可以工作：

```bash
sudo curl --unix-socket /var/run/docker.sock http://localhost/containers/json
```

如果可以拿到响应，那么说明配置代理成功了。

接着，启动`tcpdump`：

```bash
tcpdump -i lo0 port 8080
```

如果要还原回去，那么记得把复制出来的`unix socket`复制回去：

```bash
sudo mv /var/run/docker.sock.copy /var/run/docker.sock
```
