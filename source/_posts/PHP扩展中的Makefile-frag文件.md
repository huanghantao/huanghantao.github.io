---
title: PHP扩展中的Makefile.frag文件
date: 2020-07-10 17:31:41
tags:
- PHP
---

> 本文基于PHP8的commit为：9fa1d1330138ac424f990ff03e62721120aaaec3

首先，我们来创建一个扩展骨架：

```bash
$ php ext_skel.php --ext frag

Copying config scripts... done
Copying sources... done
Copying tests... done

Success. The extension is now ready to be compiled. To do so, use the
following steps:

cd /path/to/php-src/ext/frag
phpize
./configure
make

Don't forget to run tests once the compilation is done:
make test

Thank you for using PHP!
```

然后，我们进入`frag`目录，并且创建文件`Makefile.frag`：

```makefile
generate: show-generate-info
    touch generate.c

show-generate-info:
    @echo   "  +----------------------------------------------------------------------+"
    @echo   "  |                                                                      |"
    @echo   "  |                            GENERATE FILE                             |"
    @echo   "  |                            =============                             |"
    @echo   "  |                                                                      |"
    @echo   "  +----------------------------------------------------------------------+"
    @echo
    @echo
```

可以看到，这里实现的功能是创建文件`generate.c`。

然后，我们修改`config.m4`文件，完整替换为如下内容

```bash
PHP_ARG_ENABLE([frag],
  [whether to enable frag support],
  [AS_HELP_STRING([--enable-frag],
    [Enable frag support])],
  [no])

if test "$PHP_FRAG" != "no"; then
  AC_DEFINE(HAVE_FRAG, 1, [ Have frag support ])

  PHP_ADD_MAKEFILE_FRAGMENT(Makefile.frag)

  PHP_NEW_EXTENSION(frag, frag.c generate.c, $ext_shared)
fi
```

我们注意到，我们需要编译的文件除了`frag.c`之外，还有待生成的`generate.c`文件。

接着，我们开始编译我们的扩展：

```bash
$ phpize
Configuring for:
PHP Api Version:         20190128
Zend Module Api No:      20190128
Zend Extension Api No:   420190128

$ ./configure
creating libtool
appending configuration tag "CXX" to libtool
configure: patching config.h.in
configure: creating ./config.status
config.status: creating config.h
```

到这里，我们的`Makefile`文件已经生成了，我们会发现，在我们的`Makefile`文件里面，有如下内容：

```makefile
.NOEXPORT:
generate: show-generate-info
    touch generate.c

show-generate-info:
    @echo   "  +----------------------------------------------------------------------+"
    @echo   "  |                                                                      |"
    @echo   "  |                            GENERATE FILE                             |"
    @echo   "  |                            =============                             |"
    @echo   "  |                                                                      |"
    @echo   "  +----------------------------------------------------------------------+"
    @echo
    @echo
```

这实际上就是我们`Makefile.frag`里面的内容了。

所以，接下来我们要先创建`generate.c`文件：

```bash
$ make generate
  +----------------------------------------------------------------------+
  |                                                                      |
  |                            GENERATE FILE                             |
  |                            =============                             |
  |                                                                      |
  +----------------------------------------------------------------------+


touch generate.c
```

然后编译扩展：

```bash
make
----------------------------------------------------------------------
Libraries have been installed in:
   /Users/hantaohuang/codeDir/cCode/php-src/ext/frag/modules

If you ever happen to want to link against installed libraries
in a given directory, LIBDIR, you must either use libtool, and
specify the full pathname of the library, or use the `-LLIBDIR'
flag during linking and do at least one of the following:
   - add LIBDIR to the `DYLD_LIBRARY_PATH' environment variable
     during execution

See any operating system documentation about shared libraries for
more information, such as the ld(1) and ld.so(8) manual pages.
----------------------------------------------------------------------

Build complete.
Don't forget to run 'make test'.
```

此时，我们完成了这次扩展的编译。

这个技巧在我们需要动态生成源文件的时候可以使用，比如`json`扩展它有一个词法分析的文件，他就是通过这个方法来实现构建的。
