---
title: PHP8编译lex编译不过的问题
date: 2020-07-01 19:48:46
tags:
- PHP
---

> 本文基于的php-src commit为：915abeb6995bad124c325c69b8c44de65da36879

由于我经常需要去拉`php-src`的`master`分支代码的代码，然后时不时需要重新编译`php`，然后出了一个这个问题：

```bash
Zend/zend_language_scanner.l:309:15 error conflicting types for 'zend_lex_tstring'
    ZEND_API int zend_lex_tstring(zval *zv, zend_lexer_ident_ref ident_ref)

In file included from Zend/zend_language_scanner.l:40:0:
php-src/Zend/zend_language_scanner.h:81:14: note: previous declaration of 'zend_lex_tstring' was here
    ZEND_API int zend_lex_tstring(zval *zv, zend_lexer_ident_ref ident_ref)
```

这个问题是因为在`php-src`下面跑`make clean`无法清理完所有编译出来的东西（包括词法分析器编译出来的`.c`文件），所以，我们这里跑完`make clean`后需要自己手动去删除这些没删干净的东西（例如`zend_language_scanner.c`）。如何判断要删除哪些呢？也很简单，只要这个文件没有加入到`git`仓库，我们就可以删除。
