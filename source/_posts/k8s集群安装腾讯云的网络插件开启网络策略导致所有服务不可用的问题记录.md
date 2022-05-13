---
title: k8s集群安装腾讯云的网络插件开启网络策略导致所有服务不可用的问题记录
date: 2022-05-13 17:38:36
tags:
---

本来打算做每个命名空间之间的网络隔离，在公司测试环境验证没问题之后，在腾讯云开启网络隔离策略，发现不同节点之间的服务无法访问了。于是立马把网络插件给卸载了。然后发现不同节点之间的服务还是不可访问。删除了`NetworkPolicy`后，不同节点之间的服务还是不可访问。

然后发现`iptables`里面有好几千条网络插件配置的规则，所以怀疑是网络插件配置的规则还在起作用。于是先暂停了`iptables`，然后重启`iptables`，然后重启机器。网络插件配置的`iptables`的规则被清理干净了（应该是这些规则没有被持久化，所以重启可以清理这些规则）。最后，不同节点之间的服务可以访问了。

这里记录下`k8s`集群默认的`iptables`规则：

```bash
iptables --list

Chain INPUT (policy ACCEPT)
target     prot opt source               destination
cali-INPUT  all  --  anywhere             anywhere             /* cali:Cz_u1IQiXIMmKD4c */
KUBE-FIREWALL  all  --  anywhere             anywhere

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
cali-FORWARD  all  --  anywhere             anywhere             /* cali:wUHhoiAYhphO9Mso */
KUBE-FORWARD  all  --  anywhere             anywhere             /* kubernetes forwarding rules */

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
cali-OUTPUT  all  --  anywhere             anywhere             /* cali:tVnHkvAo15HuiPy0 */
KUBE-FIREWALL  all  --  anywhere             anywhere

Chain KUBE-FIREWALL (2 references)
target     prot opt source               destination
DROP       all  --  anywhere             anywhere             /* kubernetes firewall for dropping marked packets */ mark match 0x8000/0x8000
DROP       all  -- !loopback/8           loopback/8           /* block incoming localnet connections */ ! ctstate RELATED,ESTABLISHED,DNAT

Chain KUBE-FORWARD (1 references)
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere             /* kubernetes forwarding rules */ mark match 0x4000/0x4000
ACCEPT     all  --  anywhere             anywhere             /* kubernetes forwarding conntrack pod source rule */ ctstate RELATED,ESTABLISHED
ACCEPT     all  --  anywhere             anywhere             /* kubernetes forwarding conntrack pod destination rule */ ctstate RELATED,ESTABLISHED

Chain KUBE-KUBELET-CANARY (0 references)
target     prot opt source               destination
```

> 可以看到，还是比较干净的，没啥规则
