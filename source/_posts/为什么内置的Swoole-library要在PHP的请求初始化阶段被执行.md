---
title: 为什么内置的Swoole library要在PHP的请求初始化阶段被执行
date: 2020-06-06 15:16:48
tags:
- PHP
- Swoole
---

> 这篇文章基于Swoole的commit为：916478bc7457f05c9cb1b96fe97ce0279e02c50d

我们知道，`Swoole`现在可以通过PHP代码来编写`Swoole`扩展，并且，`Swoole library`是在`PHP`的请求初始化阶段被执行的。

那么，我们可能就会想，每次请求初始化的时候，都要加载一遍，为何不放在`PHP`的模块初始化的阶段呢？因为在`PHP`模块初始化的时候，`PHP`的`compiler`还没有被初始化。

我们可以做一个实验，把请求初始化中执行`library`的这段代码：

```cpp
if (
    SWOOLE_G(enable_library) && SWOOLE_G(cli)
#ifdef ZEND_COMPILE_PRELOAD
    /* avoid execution of the code during RINIT of preloader */
    && !(CG(compiler_options) & ZEND_COMPILE_PRELOAD)
#endif
)
{
    php_swoole_load_library();
}
```

放到模块初始化的这个函数的最后：


```cpp
#if defined(PHP_PCRE_VERSION) && defined(HAVE_PCRE_JIT_SUPPORT) && PHP_VERSION_ID >= 70300 && __MACH__ && !defined(SW_DEBUG)
    PCRE_G(jit) = 0;
#endif

// 增加的地方
    if (
        SWOOLE_G(enable_library) && SWOOLE_G(cli)
#ifdef ZEND_COMPILE_PRELOAD
        /* avoid execution of the code during RINIT of preloader */
        && !(CG(compiler_options) & ZEND_COMPILE_PRELOAD)
#endif
    )
    {
        php_swoole_load_library();
    }
// 增加的地方

    return SUCCESS;
```

然后重新编译安装`Swoole`扩展，然后执行如下命令：

```bash
[root@bceb11389603 client]# php -v
Segmentation fault
[root@bceb11389603 client]#
```

发现直接段错误了。我们通过`gdb`来跟踪下：

```bash
[root@bceb11389603 client]# gdb php
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-119.el7
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /usr/bin/php...done.
```

然后运行：

```bash
(gdb) r -v
Starting program: /usr/bin/php -v
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".

Program received signal SIGSEGV, Segmentation fault.
0x000000000087b57f in zend_hash_find_bucket (ht=0x14242e8 <compiler_globals+72>, key=0x7ffff5e05000, known_hash=0 '\000') at /root/php-src/Zend/zend_hash.c:631
631		idx = HT_HASH_EX(arData, nIndex);
Missing separate debuginfos, use: debuginfo-install cyrus-sasl-lib-2.1.26-23.el7.x86_64 glibc-2.17-307.el7.1.x86_64 keyutils-libs-1.5.8-3.el7.x86_64 krb5-libs-1.15.1-37.el7_7.2.x86_64 libcom_err-1.42.9-16.el7.x86_64 libcurl-7.29.0-57.el7.x86_64 libgcc-4.8.5-39.el7.x86_64 libidn-1.28-4.el7.x86_64 libselinux-2.5-14.1.el7.x86_64 libssh2-1.8.0-3.el7.x86_64 libstdc++-4.8.5-39.el7.x86_64 libxml2-2.9.1-6.el7.4.x86_64 nspr-4.21.0-1.el7.x86_64 nss-3.44.0-7.el7_7.x86_64 nss-softokn-freebl-3.44.0-8.el7_7.x86_64 nss-util-3.44.0-4.el7_7.x86_64 openldap-2.4.44-21.el7_6.x86_64 openssl-libs-1.0.2k-19.el7.x86_64 pcre-8.32-17.el7.x86_64 sqlite-3.7.17-8.el7_7.1.x86_64 xz-libs-5.2.2-1.el7.x86_64 zlib-1.2.7-18.el7.x86_64
(gdb)
```

然后看下函数调用栈：

```bash
(gdb) bt
#0  0x000000000087b57f in zend_hash_find_bucket (ht=0x14242e8 <compiler_globals+72>, key=0x7ffff5e05000, known_hash=0 '\000') at /root/php-src/Zend/zend_hash.c:631
#1  0x00000000008808a0 in zend_hash_find (ht=0x14242e8 <compiler_globals+72>, key=0x7ffff5e05000) at /root/php-src/Zend/zend_hash.c:2217
#2  0x0000000000832f20 in zend_set_compiled_filename (new_compiled_filename=0x7ffff5e05000) at /root/php-src/Zend/zend_compile.c:403
#3  0x000000000080b64a in zend_prepare_string_for_scanning (str=0x7fffffffd1e0, filename=0x1627bb8 "@swoole-src/library/constants.php") at /root/php-src/Zend/zend_language_scanner.c:727
#4  0x000000000080b8de in compile_string (source_string=0x7fffffffd450, filename=0x1627bb8 "@swoole-src/library/constants.php") at /root/php-src/Zend/zend_language_scanner.c:775
#5  0x00007ffff14e2b28 in swoole_compile_string (source_string=0x7fffffffd450, filename=0x1627bb8 "@swoole-src/library/constants.php") at /root/codeDir/cppCode/swoole-src/php_swoole_cxx.cc:53
#6  0x0000000000850a44 in zend_eval_stringl (
    str=0x1627c08 "\n/**\n * This file is part of Swoole.\n *\n * @link     https://www.swoole.com\n * @contact  team@swoole.com\n * @license  https://github.com/swoole/library/blob/master/LICENSE\n */\n\ndeclare(strict_types=1)"..., str_len=235, retval_ptr=0x0, string_name=0x1627bb8 "@swoole-src/library/constants.php") at /root/php-src/Zend/zend_execute_API.c:1070
#7  0x00007ffff14e2bb7 in zend::eval (
    code="\n/**\n * This file is part of Swoole.\n *\n * @link     https://www.swoole.com\n * @contact  team@swoole.com\n * @license  https://github.com/swoole/library/blob/master/LICENSE\n */\n\ndeclare(strict_types=1)"..., filename="@swoole-src/library/constants.php") at /root/codeDir/cppCode/swoole-src/php_swoole_cxx.cc:66
#8  0x00007ffff15732a4 in php_swoole_load_library () at /root/codeDir/cppCode/swoole-src/php_swoole_library.h:6492
#9  0x00007ffff157930f in zm_startup_swoole (type=1, module_number=32) at /root/codeDir/cppCode/swoole-src/swoole.cc:637
#10 0x000000000086f9f4 in zend_startup_module_ex (module=0x14b55a0) at /root/php-src/Zend/zend_API.c:1859
#11 0x000000000086fa56 in zend_startup_module_zval (zv=0x14b7df0) at /root/php-src/Zend/zend_API.c:1874
#12 0x000000000087f3a5 in zend_hash_apply (ht=0x1424560 <module_registry>, apply_func=0x86fa33 <zend_startup_module_zval>) at /root/php-src/Zend/zend_hash.c:1812
#13 0x000000000087005b in zend_startup_modules () at /root/php-src/Zend/zend_API.c:1985
#14 0x00000000007d1abe in php_module_startup (sf=0x1407640 <cli_sapi_module>, additional_modules=0x0, num_additional_modules=0) at /root/php-src/main/main.c:2330
#15 0x000000000093a887 in php_cli_startup (sapi_module=0x1407640 <cli_sapi_module>) at /root/php-src/sapi/cli/php_cli.c:407
#16 0x000000000093c622 in main (argc=2, argv=0x1428c30) at /root/php-src/sapi/cli/php_cli.c:1319
(gdb)
```

通过调用栈，我们可以分析出，在`PHP`编译`Swoole library`的时候，会去查找`CG(filenames_table)`这个哈希表，但是因为`CG(filenames_table)`没有被初始化，所以查找的过程中就段错误了。

我们可以看一下`CG(filenames_table)`是在哪里初始化的：

```cpp
void init_compiler(void) /* {{{ */
{
	// 省略其他的代码
	zend_hash_init(&CG(filenames_table), 8, NULL, ZVAL_PTR_DTOR, 0);
	// 省略其他的代码
}
```

而`init_compiler`这个函数，是在`PHP`模块初始化之后执行的，所以在模块初始化的阶段，我们不能去编译、执行`PHP`代码。因此，`Swoole library`把`library`的编译和执行放在了扩展的`PHP_RINIT_FUNCTION(swoole)`这个函数里面。
