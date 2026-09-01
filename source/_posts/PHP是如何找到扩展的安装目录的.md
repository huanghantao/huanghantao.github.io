---
title: PHP是如何找到扩展的安装目录的
date: 2020-08-11 09:40:02
tags:
- PHP
---

有小伙伴可能会疑问，自己没有在`php.ini`文件里面配置`extension_dir`，那么`PHP`它是如何找到扩展的安装目录的呢？

实际上，我们在编译`PHP`的时候，这个时候就决定了扩展的安装目录。并且，这个路径在`php-src/main/build-defs.h`里面可以找到，例如，在我的机器上面，就是：

```cpp
#define PHP_EXTENSION_DIR       "/Users/hantaohuang/codeDir/cCode/php-src/tmp/usr/lib/php/extensions/debug-zts-20190128"
```

除了这个之外，还有很多其他的在编译`PHP`的时候就写入头文件的内容，例如：

```cpp
/*
+----------------------------------------------------------------------+
| Copyright (c) The PHP Group                                          |
+----------------------------------------------------------------------+
| This source file is subject to version 3.01 of the PHP license,      |
| that is bundled with this package in the file LICENSE, and is        |
| available through the world-wide-web at the following url:           |
| http://www.php.net/license/3_01.txt                                  |
| If you did not receive a copy of the PHP license and are unable to   |
| obtain it through the world-wide-web, please send a note to          |
| license@php.net so we can mail you a copy immediately.               |
+----------------------------------------------------------------------+
| Author: Stig Sæther Bakken <ssb@php.net>                             |
+----------------------------------------------------------------------+
*/

#define CONFIGURE_COMMAND " './configure'  '--disable-all' '--prefix=/Users/hantaohuang/codeDir/cCode/php-src/tmp/usr' '--with-config-file-path=/Users/hantaohuang/codeDir/cCode/php-src/tmp/etc' '--with-config-file-scan-dir=/Users/hantaohuang/codeDir/cCode/php-src/tmp/etc/php.d' '--enable-debug' '--enable-zts' '--with-ffi'"
#define PHP_ODBC_CFLAGS ""
#define PHP_ODBC_LFLAGS     ""
#define PHP_ODBC_LIBS       ""
#define PHP_ODBC_TYPE       ""
#define PHP_OCI8_DIR            ""
#define PHP_OCI8_ORACLE_VERSION     ""
#define PHP_PROG_SENDMAIL   "/usr/sbin/sendmail"
#define PEAR_INSTALLDIR         ""
#define PHP_INCLUDE_PATH    ".:"
#define PHP_EXTENSION_DIR       "/Users/hantaohuang/codeDir/cCode/php-src/tmp/usr/lib/php/extensions/debug-zts-20190128"
#define PHP_PREFIX              "/Users/hantaohuang/codeDir/cCode/php-src/tmp/usr"
#define PHP_BINDIR              "/Users/hantaohuang/codeDir/cCode/php-src/tmp/usr/bin"
#define PHP_SBINDIR             "/Users/hantaohuang/codeDir/cCode/php-src/tmp/usr/sbin"
#define PHP_MANDIR              "/Users/hantaohuang/codeDir/cCode/php-src/tmp/usr/php/man"
#define PHP_LIBDIR              "/Users/hantaohuang/codeDir/cCode/php-src/tmp/usr/lib/php"
#define PHP_DATADIR             "/Users/hantaohuang/codeDir/cCode/php-src/tmp/usr/share/php"
#define PHP_SYSCONFDIR          "/Users/hantaohuang/codeDir/cCode/php-src/tmp/usr/etc"
#define PHP_LOCALSTATEDIR       "/Users/hantaohuang/codeDir/cCode/php-src/tmp/usr/var"
#define PHP_CONFIG_FILE_PATH    "/Users/hantaohuang/codeDir/cCode/php-src/tmp/etc"
#define PHP_CONFIG_FILE_SCAN_DIR    "/Users/hantaohuang/codeDir/cCode/php-src/tmp/etc/php.d"
#define PHP_SHLIB_SUFFIX        "so"
#define PHP_SHLIB_EXT_PREFIX    ""
```

我们发现，`PHP_EXTENSION_DIR`的命名规范如下：

```bash
<install-dir>/lib/php/extensions/<debug-or-not>-<zts-or-not>-ZEND_MODULE_API_NO
```

其中，`<install-dir>`就是我们的`PHP_PREFIX`了。`<debug-or-not>`代表是否开启`PHP`的`debug`模式，我们可以在编译的时候指定`--enable-debug`。`<zts-or-not>`代表是否开启`zts`，`PHP8`中，可以通过指定`--enable-zts`来开启它。`ZEND_MODULE_API_NO`是一个可以表示`PHP`版本的数字，大概是由`年/月/日`组成。

正是因为`PHP_EXTENSION_DIR`的目录有多个在编译期间决定的变量，所以我们要注意，在编译扩展的时候，`PHP`版本要是一致的，要不然`PHP`找不到编译的扩展。
