---
title: 在编写PHP扩展的时候，编译通过，但是报错某个方法的symbol not found
date: 2019-03-05 15:33:51
tags:
- PHP扩展开发
- PHP
---

在编写PHP扩展的时候，`make`和`make install`没有问题：

```shell
~/codeDir/cCode/encryptor # make && make install
/bin/sh /root/codeDir/cCode/encryptor/libtool --mode=compile cc -DZEND_ENABLE_STATIC_TSRMLS_CACHE=1 -I. -I/root/codeDir/cCode/encryptor -DPHP_ATOM_INC -I/root/codeDir/cCode/encryptor/include -I/root/codeDir/cCode/encryptor/main -I/root/codeDir/cCode/encryptor -I/usr/local/include/php -I/usr/local/include/php/main -I/usr/local/include/php/TSRM -I/usr/local/include/php/Zend -I/usr/local/include/php/ext -I/usr/local/include/php/ext/date/lib  -DHAVE_CONFIG_H  -g -O2   -c /root/codeDir/cCode/encryptor/encryptor_aes128.c -o encryptor_aes128.lo 
 cc -DZEND_ENABLE_STATIC_TSRMLS_CACHE=1 -I. -I/root/codeDir/cCode/encryptor -DPHP_ATOM_INC -I/root/codeDir/cCode/encryptor/include -I/root/codeDir/cCode/encryptor/main -I/root/codeDir/cCode/encryptor -I/usr/local/include/php -I/usr/local/include/php/main -I/usr/local/include/php/TSRM -I/usr/local/include/php/Zend -I/usr/local/include/php/ext -I/usr/local/include/php/ext/date/lib -DHAVE_CONFIG_H -g -O2 -c /root/codeDir/cCode/encryptor/encryptor_aes128.c  -fPIC -DPIC -o .libs/encryptor_aes128.o
/bin/sh /root/codeDir/cCode/encryptor/libtool --mode=link cc -DPHP_ATOM_INC -I/root/codeDir/cCode/encryptor/include -I/root/codeDir/cCode/encryptor/main -I/root/codeDir/cCode/encryptor -I/usr/local/include/php -I/usr/local/include/php/main -I/usr/local/include/php/TSRM -I/usr/local/include/php/Zend -I/usr/local/include/php/ext -I/usr/local/include/php/ext/date/lib  -DHAVE_CONFIG_H  -g -O2   -o encryptor.la -export-dynamic -avoid-version -prefer-pic -module -rpath /root/codeDir/cCode/encryptor/modules  encryptor.lo encryptor_aes128.lo 
rm -fr  .libs/encryptor.la .libs/encryptor.lai .libs/encryptor.so
cc -shared  .libs/encryptor.o .libs/encryptor_aes128.o   -Wl,-soname -Wl,encryptor.so -o .libs/encryptor.so
creating encryptor.la
(cd .libs && rm -f encryptor.la && ln -s ../encryptor.la encryptor.la)
/bin/sh /root/codeDir/cCode/encryptor/libtool --mode=install cp ./encryptor.la /root/codeDir/cCode/encryptor/modules
cp ./.libs/encryptor.so /root/codeDir/cCode/encryptor/modules/encryptor.so
cp ./.libs/encryptor.lai /root/codeDir/cCode/encryptor/modules/encryptor.la
PATH="$PATH:/sbin" ldconfig -n /root/codeDir/cCode/encryptor/modules
----------------------------------------------------------------------
Libraries have been installed in:
   /root/codeDir/cCode/encryptor/modules

If you ever happen to want to link against installed libraries
in a given directory, LIBDIR, you must either use libtool, and
specify the full pathname of the library, or use the `-LLIBDIR'
flag during linking and do at least one of the following:
   - add LIBDIR to the `LD_LIBRARY_PATH' environment variable
     during execution
   - add LIBDIR to the `LD_RUN_PATH' environment variable
     during linking
   - use the `-Wl,--rpath -Wl,LIBDIR' linker flag

See any operating system documentation about shared libraries for
more information, such as the ld(1) and ld.so(8) manual pages.
----------------------------------------------------------------------

Build complete.
Don't forget to run 'make test'.

Installing shared extensions:     /usr/local/lib/php/extensions/no-debug-non-zts-20170718/
```

但是在测试这个扩展的某个方法的时候：

```php
<?php
	
	$encryptor = new Encryptor\Aes128();
	var_dump($encryptor);
```

报错：

```shell
~/codeDir/cCode/encryptor # php encryptor.php 
PHP Warning:  PHP Startup: Unable to load dynamic library 'encryptor.so' (tried: /usr/local/lib/php/extensions/no-debug-non-zts-20170718/encryptor.so (Error relocating /usr/local/lib/php/extensions/no-debug-non-zts-20170718/encryptor.so: zim_encryptor_aes128___construct: symbol not found), /usr/local/lib/php/extensions/no-debug-non-zts-20170718/encryptor.so.so (Error loading shared library /usr/local/lib/php/extensions/no-debug-non-zts-20170718/encryptor.so.so: No such file or directory)) in Unknown on line 0

Fatal error: Uncaught Error: Class 'Encryptor\Aes128' not found in /root/codeDir/cCode/encryptor/encryptor.php:3
Stack trace:
#0 {main}
  thrown in /root/codeDir/cCode/encryptor/encryptor.php on line 3
```

原因：只是声明了`___construct`方法：

```c
PHP_METHOD(encryptor_aes128, __construct);
```

并没有去实现它，所以导致了报错。

所以需要去实现它：

```c
PHP_METHOD(encryptor_aes128, __construct)
{
    // specific implementation
}
```

