---
title: CPU的分支预测
date: 2020-06-08 16:17:55
tags:
- 性能优化
---

我们在`Swoole`源码里面会看到很多的`likely`和`unlikely`宏，例如在创建协程的时候，有如下代码：

```cpp
long cid = PHPCoroutine::create(&fci_cache, fci.param_count, fci.params);
if (sw_likely(cid > 0))
{
    RETURN_LONG(cid);
}
else
{
    RETURN_FALSE;
}
```

这里，`Swoole`调用`PHPCoroutine::create`来创建协程，并且返回了协程的`id`。然后，接着是对`cid`使用了宏`likely`。这个用法是为了提升`CPU`指令缓存的命中率。

`CPU`缓存是几块离`CPU`近的存储空间。`CPU`通常分为三级缓存（不是代表只有三块缓存，而是等级有三类）。我们可以通过一下命令来查看`CPU`缓存的大小：

```bash
[root@bceb11389603 test]# lscpu | grep cache
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              4096K
[root@bceb11389603 test]#
```

可以看出，我的机器上有四块`CPU`缓存，其中有两块`L1`缓存，大小是`32K`；一块`L2`缓存，大小是`256KB`；一块`L3`缓存，大小是`4096KB`。

我们发现，`L3`缓存要比`L1`、`L2`级缓存大很多，因为现在的`CPU`都是多核的，每个核都有自己的`L1`、`L2`级缓存，但`L3`级缓存却是同一颗`CPU`上所有核心共享的。程序执行时，会先将内存中的数据载入到共享的`L3`级缓存中，再进入每颗核心独有的`L2`级缓存，最后进入最快的`L1`级缓存，最后才会被`CPU`的核使用。

那么为什么有两个`L1`级缓存呢？因为`CPU`核会对指令与数据进行区分。指令会放在`L1`级**指令**缓存中，而指令所需要的数据放在`L1`级**数据**缓存中。

所以，我们前面所说的`likely`核`unlikely`实际上就是为了提升指令缓存的命中率。而`likely`和`unlikely`宏是编译器为我们提供的显式预测分支概率的工具。实际上，`CPU`自身的条件预测已经非常准了，仅当我们确定`CPU`条件预测不准，且我们能够知晓实际概率时，才需要加入这两个宏。

> 如果我们需要查看指令缓存的命中率，我们可以通过perf这个工具来查看
