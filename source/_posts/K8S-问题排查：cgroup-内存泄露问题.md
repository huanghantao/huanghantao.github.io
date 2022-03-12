---
title: K8S 问题排查：cgroup 内存泄露问题
date: 2022-03-12 16:48:33
tags:
---

公司测试环境的`k8s`集群突然创建不了新的`pod`了。报错如下：

```bash
unable to ensure pod container exists: failed to create container for [kubepods besteffort podca64cf9d-cbe4-4bf3-ac5b-85ad793b4805] : mkdir /sys/fs/cgroup/memory/kubepods/besteffort/podca64cf9d-cbe4-4bf3-ac5b-85ad793b4805: cannot allocate memory
```

原因：cgroup 的 kmem account 特性在 Linux 3.x 内核上有内存泄露问题，然后`k8s`用了这个特性，导致后面创建不出新的`pod`来了。

解决的办法看了下有比较多，我这里选择直接禁用`cgroup`的`kmem`特性，毕竟这个特性也是实验性质的：

```bash
# 修改/etc/default/grub 为
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet cgroup.memory=nokmem"

# 生成配置
/usr/sbin/grub2-mkconfig -o /boot/grub2/grub.cfg

# 重启机器
reboot
```

无输出即可：

```bash
cat /sys/fs/cgroup/memory/kubepods/burstable/pod*/*/memory.kmem.slabinfo

cat: /sys/fs/cgroup/memory/kubepods/burstable/podb096d657-583a-4400-aee2-8454050dcd29/2c77b07a295d72db97b7179d71b61e3ecebb2ddb501e76d12e109c357ea0cb98/memory.kmem.slabinfo: Input/output error
cat: /sys/fs/cgroup/memory/kubepods/burstable/podb096d657-583a-4400-aee2-8454050dcd29/777da0d32fd867895b0cd5f2710538758748dd2c7807afdf1b412c92640919c2/memory.kmem.slabinfo: Input/output error
cat: /sys/fs/cgroup/memory/kubepods/burstable/podbef17dfd-1625-4cbf-9617-5c1187365241/bd45c4bad9346c99e87441706563a3f9d1a9fba83da2b9005149cfac16983d70/memory.kmem.slabinfo: Input/output error
cat: /sys/fs/cgroup/memory/kubepods/burstable/podbef17dfd-1625-4cbf-9617-5c1187365241/c89a64afc1dbdb10a73bd8e8df61c8e03fd6e087b84c2899578b90f8840a9d83/memory.kmem.slabinfo: Input/output error
cat: /sys/fs/cgroup/memory/kubepods/burstable/podc0a765a7-918a-444f-b05a-4fc93a3631b5/7f533121574f96927542288c7babe3f2e2b58440e61fe84f371311d888d12b0b/memory.kmem.slabinfo: Input/output error
cat: /sys/fs/cgroup/memory/kubepods/burstable/podc0a765a7-918a-444f-b05a-4fc93a3631b5/e0165ed665cee3d9d42b22c8efa68ea28d151183c73e82027971a41eee54a774/memory.kmem.slabinfo: Input/output error
cat: /sys/fs/cgroup/memory/kubepods/burstable/pode685b3ee-ae22-44be-986d-e9338994f8f6/33f284c99520d9e307057decef9148a17b02bfad3fede697d425f614283af234/memory.kmem.slabinfo: Input/output error
cat: /sys/fs/cgroup/memory/kubepods/burstable/pode685b3ee-ae22-44be-986d-e9338994f8f6/48e06b2628076f083994b4d7e2078f5f984f434961ca0c90d329b467279f1387/memory.kmem.slabinfo: Input/output error
cat: /sys/fs/cgroup/memory/kubepods/burstable/pode8699faf-6068-454c-b232-3868f81e2fff/3215ef97074731414a5641a550b4e5f2697107c612e4b49df34d70fa2543d8a1/memory.kmem.slabinfo: Input/output error
cat: /sys/fs/cgroup/memory/kubepods/burstable/pode8699faf-6068-454c-b232-3868f81e2fff/d08e473be78cbee1eb5ed8b9036200d3722cc5c708962b27b96333734377424e/memory.kmem.slabinfo: Input/output error
```
