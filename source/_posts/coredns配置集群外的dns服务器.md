---
title: coredns配置集群外的dns服务器
date: 2021-12-24 16:42:09
tags:
---

原来的配置文件如下：

```conf
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    hosts {
        192.168.1.175 	git.swoole.com
        192.168.1.188  registry.swoole.inc
        192.168.1.136	 gitlab.swoole.inc

        fallthrough
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

假设，我们有一个内网的`dns`服务器`192.168.1.129`，专门解析`other.inc`域，我们可以加这么一段配置：

```conf
other.inc.:53 {
    errors
    log
    prometheus :9153
    loadbalance
    forward . 192.168.1.129
    cache 30
}
```

最终配置文件如下：

```conf
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    hosts {
        192.168.1.175 	git.swoole.com
        192.168.1.188  registry.swoole.inc
        192.168.1.136	 gitlab.swoole.inc

        fallthrough
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}

other.inc.:53 {
    errors
    log
    prometheus :9153
    loadbalance
    forward . 192.168.1.129
    cache 30
}
```
