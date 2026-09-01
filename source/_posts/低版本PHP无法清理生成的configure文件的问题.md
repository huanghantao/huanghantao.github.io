---
title: 低版本PHP无法清理生成的configure文件的问题
date: 2021-03-15 16:38:56
tags:
- PHP
---

今天在用`PHP7.1`的时候，执行`phpize`报错：

```bash
Cleaning..
Configuring for:
PHP Api Version:         20160303
Zend Module Api No:      20160303
Zend Extension Api No:   320160303
autoconf: warning: both `configure.ac' and `configure.in' are present.
autoconf: warning: proceeding with `configure.ac'.
/usr/bin/m4:configure.ac:6: cannot open `build/ax_gcc_func_attribute.m4': No such file or directory
/usr/bin/m4:configure.ac:8: cannot open `build/php_cxx_compile_stdcxx.m4': No such file or directory
/usr/bin/m4:configure.ac:9: cannot open `build/php.m4': No such file or directory
/usr/bin/m4:configure.ac:10: cannot open `build/pkg.m4': No such file or directory
autom4te: /usr/bin/m4 failed with exit status: 1
```

执行完`phpize --clean`之后，依然还是报这个错。发现是`phpize --clean`无法清理干净之前其他版本`PHP`生成的`configure`，所以，得手动删除：

```bash
rm configure*
```

然后重新执行`phpize`即可。
