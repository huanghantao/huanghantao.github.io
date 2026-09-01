---
title: MacOS下无法对PHP扩展进行strip的问题
date: 2021-03-16 17:16:06
tags:
- clang
- PHP扩展
---

如果我们使用`MacOS`自带的`strip`命令，那么回报这个错误：

```bash
strip test.so
/Library/Developer/CommandLineTools/usr/bin/strip: error: symbols referenced by indirect symbol table entries that can't be stripped in: test.so
```

我们需要用`llvm-strip`来进行`strip`：

```bash
llvm-strip test.so
```
