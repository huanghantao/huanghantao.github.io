---
title: PHP扩展如何去检查依赖的C++库是否存在
date: 2020-12-16 17:11:35
tags:
- PHP
- PHP扩展
---

找了一圈，发现`PHP`没有去实现这样的一个宏来进行检测。并且，发现所有`C++ wrapper`扩展都没有去实现这个功能，只是在文档里面说了一下依赖了这个`C++`库。这样不太好，容易让开发者在编译的途中，发现漏了一个依赖库。

所以，我就写了一个简单的宏来实现这个功能：

```bash
AC_DEFUN([YASD_CHECK_CXX_LIB], [
    AC_LANG_PUSH([C++])
    LIBNAME=$1
    AC_MSG_CHECKING([for boost])
    AC_TRY_COMPILE(
        [ #include $2 ],
        [],
        [ AC_MSG_RESULT(yes) ],
        [ AC_MSG_ERROR([lib $LIBNAME not found.  Try: install $LIBNAME library]) ]
    )
    AC_LANG_POP([C++])
])
```

那么怎么去用呢？我们只需要随便去找一个库的头文件就好了，例如：

```bash
YASD_CHECK_CXX_LIB([boost], [<boost/algorithm/string/constants.hpp>])
```

第一个参数填写依赖库的名字，第二个参数填头文件。
