---
title: C代码从x86迁移到ARM需要注意的点
date: 2021-03-13 21:26:03
tags:
---

最近工作中遇到的一个问题，`PHP`的`hash`算法在`x86`下和`ARM`下计算结果不一样：

```cpp
static zend_always_inline zend_ulong zend_inline_hash_func(const char *str, size_t len)
{
    // 省略其他代码
    hash = ((hash << 5) + hash) + *str++;
    // 省略其他代码
}
```

这里会对`str`进行处理。在`x86`下面，`char`默认是`signed`的；在`ARM`下，`char`是`unsigned`的。那么，就有可能出现`str`在`x86`下是负数，在`ARM`下是正数。

这个问题，可以通过添加编译器的`-fsigned-char`选项得到解决。加在`PHP`的`Makefile`的`CFLAGS`后面即可：

```Makefile
CFLAGS = $(CFLAGS_CLEAN) -prefer-non-pic -static -fsigned-char
```

当然，我们也可以显式的定义`str`是`signed`。

除了这个问题之外，还有其他的差异：

```bash
graph-easy <<< "
[
    info\n
Explicitly define a variable of type char to be signed.
] -> 
[
    x86 code\n
char c = '*';
] -> 
[
    ARM code\n
signed c = '*';
]

[
    info\n
Use the builtin function that comes with the compiler.
] -> 
[
    x86 code\n
__builtin_ia32_crc32qi(__a, __b);
] -> 
[
    ARM code\n
__builtin_aarch64_crc32b(__a, __b);
]

[
    info\n
prefetch memory data into the cache.
] -> 
[
    x86 code\n
asm volatile(""prefetcht0 %0""::""m"" (*(unsigned long *)x));
] -> 
[
    ARM code\n
define prefetch(_x) __builtin_prefetch(_x);
]

[
    info\n
define the compiled program to be 64-bit.
] -> 
[
    x86 code\n
-m64;
] -> 
[
    ARM code\n
-mabi=lp64;
]

[
    info\n
the instruction set type is defined in the Makefile, from x86 to ARM.
] -> 
[
    x86 code\n
-march=broadwell;
] -> 
[
    ARM code\n
-march=armv8-a;
]

[
    info\n
replace the original x86 version of the compiled macros with the ARM version.
] -> 
[
    x86 code\n
__X86_64__;
] -> 
[
    ARM code\n
__ARM_64__;
]
"

+-------------------------------------------------------------------------------+     +-------------------------------------------------------+     +---------------------------------------------+
|                                     info                                      |     |                       x86 code                        |     |                  ARM code                   |
|            Explicitly define a variable of type char to be signed.            | --> |                     char c = '*';                     | --> |               signed c = '*';               |
+-------------------------------------------------------------------------------+     +-------------------------------------------------------+     +---------------------------------------------+
+-------------------------------------------------------------------------------+     +-------------------------------------------------------+     +---------------------------------------------+
|                                     info                                      |     |                       x86 code                        |     |                  ARM code                   |
|            Use the builtin function that comes with the compiler.             | --> |           __builtin_ia32_crc32qi(__a, __b);           | --> |     __builtin_aarch64_crc32b(__a, __b);     |
+-------------------------------------------------------------------------------+     +-------------------------------------------------------+     +---------------------------------------------+
+-------------------------------------------------------------------------------+     +-------------------------------------------------------+     +---------------------------------------------+
|                                     info                                      |     |                       x86 code                        |     |                  ARM code                   |
|                   define the compiled program to be 64-bit.                   | --> |                         -m64;                         | --> |                 -mabi=lp64;                 |
+-------------------------------------------------------------------------------+     +-------------------------------------------------------+     +---------------------------------------------+
+-------------------------------------------------------------------------------+     +-------------------------------------------------------+     +---------------------------------------------+
|                                     info                                      |     |                       x86 code                        |     |                  ARM code                   |
|                     prefetch memory data into the cache.                      | --> | asm volatile(prefetcht0 %0::m (*(unsigned long *)x)); | --> | define prefetch(_x) __builtin_prefetch(_x); |
+-------------------------------------------------------------------------------+     +-------------------------------------------------------+     +---------------------------------------------+
+-------------------------------------------------------------------------------+     +-------------------------------------------------------+     +---------------------------------------------+
|                                     info                                      |     |                       x86 code                        |     |                  ARM code                   |
| replace the original x86 version of the compiled macros with the ARM version. | --> |                      __X86_64__;                      | --> |                 __ARM_64__;                 |
+-------------------------------------------------------------------------------+     +-------------------------------------------------------+     +---------------------------------------------+
+-------------------------------------------------------------------------------+     +-------------------------------------------------------+     +---------------------------------------------+
|                                     info                                      |     |                       x86 code                        |     |                  ARM code                   |
|     the instruction set type is defined in the Makefile, from x86 to ARM.     | --> |                   -march=broadwell;                   | --> |               -march=armv8-a;               |
+-------------------------------------------------------------------------------+     +-------------------------------------------------------+     +---------------------------------------------+
```
