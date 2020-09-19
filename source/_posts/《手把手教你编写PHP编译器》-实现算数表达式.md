---
title: 《手把手教你编写PHP编译器》--实现算数表达式
date: 2020-09-19 16:43:44
tags:
- PHP
- 编译原理
---

# 实现算数表达式

这篇文章，我们来用`flex`和`bison`实现算数表达式。

几乎所有的编译原理教程都会以这个为例子进行讲解，因为算数表达式的例子是比较复杂一点的，主要是因为它的语法会比其他语法难一点，这其中会涉及到递归，优先级等问题。而关于优先级问题，我们可以使用`bison`自带的功能来解决，但是，我们也会去讲如何自己手动来实现优先级。

首先，我们来写一下`zend_language_scanner.l`：

```flex
%option noyywrap
%option nounput
%option noinput
%{
#include <iostream>
#include "zend_language_parser.h"
%}

%%
echo {
    return T_ECHO;
}

[;(),+*/-] {
    return *yytext;
}

[0-9]+ {
    yylval = atoi(yytext);
    return T_LNUMBER;
}

[#].* /* ignore comment */

[\t\n ]+  /* ignore \t, \n, whitespace */

. {
    std::cerr << "Lexical error. Unrecognized character: '" << *yytext << "'" << std::endl;
    exit(EXIT_FAILURE);
}
%%
```

我们逐一进行分析。

```flex
%{
#include <iostream>
#include "zend_language_parser.h"
%}
```

我们发现，这里有一对：

```flex
%{

%}
```

我们可以在这个地方去引入一些头文件。这里的重点是`zend_language_parser.h`这个文件，我们在之前的文章已经发现，`zend_language_parser.h`是由`zend_language_parser.y`通过`bison`生成的。那么我们为什么要引入这个头文件呢？因为我们会去使用`zend_language_parser.h`的一些东西（至于是什么，我们后面会讲）。

```flex
%%
echo {
    return T_ECHO;
}

[;(),+*/-] {
    return *yytext;
}

[0-9]+ {
    yylval = atoi(yytext);
    return T_LNUMBER;
}

[#].* /* ignore comment */

[\t\n ]+  /* ignore \t, \n, whitespace */

. {
    std::cerr << "Lexical error. Unrecognized character: '" << *yytext << "'" << std::endl;
    exit(EXIT_FAILURE);
}
%%
```

我们发现，这里有一对：

```flex
%%

%%
```

我们可以在这里面去定义我们需要解析的`token`。那么`token`是什么呢？我们来看这么一个例子。我们有如下`php`文件：

```php
<?php
$tokens = token_get_all('<?php 1 + 2 + 3;');

foreach ($tokens as $token) {
    if (is_array($token)) {
        echo token_name($token[0]), " ('{$token[1]}')", PHP_EOL;
    } else {
        echo $token, PHP_EOL;
    }
}
```

执行结果如下：

```bash
T_OPEN_TAG ('<?php ')
T_LNUMBER ('1')
T_WHITESPACE (' ')
+
T_WHITESPACE (' ')
T_LNUMBER ('2')
T_WHITESPACE (' ')
+
T_WHITESPACE (' ')
T_LNUMBER ('3')
;
```

可以发现，从

```php
<?php 1 + 2 + 3;
```

代码里面得到的`token`包括了`T_OPEN_TAG`、`T_LNUMBER`、`T_WHITESPACE`、`+`、`T_LNUMBER`、`;`。

所以，我们的`%% %%`里面的一些列正则，实际上做的事情就和`PHP`类似了。它会根据这些正则表达式，对输入的代码（也就是字符串）进行解析，如果匹配到了某条正则，那么就拿到对应的`token`。

其中：

```flex
[;(),+*/-] {
    return *yytext;
}
```

表示，如果当前输入的是字符是`;(),+*/-`里面的某一个，那么，我们直接返回这个字符作为它的`token`。这和我们上面的`PHP`代码行为一致，`PHP`没有为它们单独写一个`T_*`。

接着，我们来写一下我们的`zend_language_parser.y`文件：

```bison
%{
#include <stdio.h>
#include <string.h>

extern int yylex(void);
extern int yyparse(void);
extern FILE *yyin;
extern int yylineno;

int yywrap()
{
    return 1;
}

void yyerror(const char *s)
{
    printf("[error] %s, in %d\n", s, yylineno);
}

int main(int argc, char const *argv[])
{
    const char *file = argv[1];
    FILE *fp = fopen(file, "r");

    if(fp == NULL)
    {
        printf("cannot open %s\n", file);
        return -1;
    }

    yyin = fp;
    yyparse();

    return 0;
}
%}

%token T_ECHO T_LNUMBER

%%

statement: %empty
|   T_ECHO additive_expr    { printf("%d\n", $2); }
;

additive_expr: %empty
|   multiplicative_expr                                 {$$ = $1;}
|   additive_expr '+' multiplicative_expr                 {$$ = $1 + $3;}
|   additive_expr '-' multiplicative_expr                 {$$ = $1 - $3;}
;

multiplicative_expr: %empty
|   primary                             {$$ = $1;}
|   multiplicative_expr '*' primary   {$$ = $1 * $3;}
|   multiplicative_expr '/' primary   {$$ = $1 / $3;}

primary: %empty
|   T_LNUMBER    {$$ = $1;}
|   '(' additive_expr ')'    {$$ = $2;}

%%
```

我们发现，这个文件的结构和`zend_language_scanner.l`极其相似。在`%{ %}`中写一点`cpp`的东西。在`%%  %%`中写一点语法（只不过`zend_language_scanner.l`是写一些词法）。我们逐一来看。

首先是：

```bison
int yywrap()
{
    return 1;
}
```

这在多文件解析的时候会用到。如果返回`1`，表示我们解析文件完毕。如果返回`0`，说明我们还有文件需要解析。简单起见，我们先只支持单个文件的解析。

```bison
void yyerror(const char *s)
{
    printf("[error] %s, in %d\n", s, yylineno);
}
```

当语法分析出错的时候，会调用这个函数。其中`yylineno`表示哪一行解析失败了。

```bison
int main(int argc, char const *argv[])
{
    const char *file = argv[1];
    FILE *fp = fopen(file, "r");

    if(fp == NULL)
    {
        printf("cannot open %s\n", file);
        return -1;
    }

    yyin = fp;
    yyparse();

    return 0;
}
```

表示我们输入一个`yaphp`文件，然后对这个文件进行解析。`yyin`里面存了当前要解析的文件。

```bison
%token T_ECHO T_LNUMBER
```

定义了两个`token`。这是提供给`zend_language_scanner.l`文件使用的。我们可以在文件`zend_language_parser.h`里面找到这两个`token`具体会被生成什么：

```cpp
/* Token kinds.  */
#ifndef YYTOKENTYPE
# define YYTOKENTYPE
  enum yytokentype
  {
    YYEMPTY = -2,
    YYEOF = 0,                     /* "end of file"  */
    YYerror = 256,                 /* error  */
    YYUNDEF = 257,                 /* "invalid token"  */
    T_ECHO = 258,                  /* T_ECHO  */
    T_LNUMBER = 259                /* T_LNUMBER  */
  };
  typedef enum yytokentype yytoken_kind_t;
#endif
```

可以发现，实际上，我们在文件`zend_language_parser.y`里面定义的`token`，最终会被生成一个枚举类型的值。这也很好的解释了为什么我们需要在`zend_language_scanner.l`里面引入头文件`zend_language_parser.h`。

接着，就是我们的重点了：

```bison
statement: %empty
|   T_ECHO additive_expr    { printf("%d\n", $2); }
;

additive_expr: %empty
|   multiplicative_expr                                 {$$ = $1;}
|   additive_expr '+' multiplicative_expr                 {$$ = $1 + $3;}
|   additive_expr '-' multiplicative_expr                 {$$ = $1 - $3;}
;

multiplicative_expr: %empty
|   primary                             {$$ = $1;}
|   multiplicative_expr '*' primary   {$$ = $1 * $3;}
|   multiplicative_expr '/' primary   {$$ = $1 / $3;}

primary: %empty
|   T_LNUMBER    {$$ = $1;}
|   '(' additive_expr ')'    {$$ = $2;}
```

这就是我们算数表达式的语法了。这里，我们实现了优先级。其中括号`()`的优先级最高，乘法和除法的优先级其次，加法和减法的优先级最低。

那么，这个优先级是如何去实现的呢？我们可以看到，我们的加法表达式语法嵌套了乘法表达式语法；乘法表达式语法嵌套了我们的括号。

所以，我们通过文法的一个嵌套，实现了优先级。最下面的语法，它的优先级最高，最上面的语法，它的优先级最低。（如果对优先级这一块不太清楚的小伙伴，我建议自己去手写一遍优先级，然后就能够理解为什么了。我始终觉得，虽然我们是使用了`bison`做语法分析，但是，我们一定要具备没有这类工具，也可以去实现这些语法的能力，因为这是基本功）

好的，现在，让我们来运行一下我们的编译脚本：

```bash
./rebuild.sh
```

然后编写测试文件`test1.php`：

```php
echo 2 + 2 * 3 / 2
```

执行我们的`yaphp`：

```bash
./build/yaphp tests/test1.php
5
```

我们发现，这里先计算了`2 * 3 / 2`，得到`3`之后，在进行`2 + 3`的运算，得到`5`。

我们再来一个例子：

```php
echo (2 + 2) * 3 / 2
```

执行我们的`yaphp`：

```bash
./build/yaphp tests/test1.php
6
```

我们发现，这里先计算了`(2 + 2)`得到`4`之后，在进行`4 * 3 / 2`的运算，得到`6`。

`OK`，我们算是实现了优先级。但是，这么去写语法规则是非常的难看的，一点也不优雅。

我们来借助`bison`的能力，实现一下优先级。我们修改一下我们的语法文件：

```bison
%{
#include <stdio.h>
#include <string.h>

extern int yylex(void);
extern int yyparse(void);
extern FILE *yyin;
extern int yylineno;

int yywrap()
{
    return 1;
}

void yyerror(const char *s)
{
    printf("[error] %s, in line %d\n", s, yylineno);
}

int main(int argc, char const *argv[])
{
    const char *file = argv[1];
    FILE *fp = fopen(file, "r");

    if(fp == NULL)
    {
        printf("cannot open %s\n", file);
        return -1;
    }

    yyin = fp;
    yyparse();

    return 0;
}
%}

%token T_ECHO T_LNUMBER

%left '+' '-'
%left '*' '/'
%left '(' ')'

%%

statement: %empty
|   T_ECHO expr    { printf("%d\n", $2); }
;

expr: %empty
|   expr '+' expr                 {$$ = $1 + $3;}
|   expr '-' expr                 {$$ = $1 - $3;}
|   expr '*' expr                 {$$ = $1 * $3;}
|   expr '/' expr                 {$$ = $1 / $3;}
|   '(' expr ')'                  {$$ = $2;}
|   T_LNUMBER                      {$$ = $1;}
;

%%
```

可以发现，我们这里多了一部分东西：

```bison
%left '+' '-'
%left '*' '/'
%left '(' ')'
```

这实际上就是我们的优先级定义了。自上而下，它的优先级越来越高。我们现在重新编译一下我们的`yaphp`：

```bash
./rebuild.sh
```

然后执行上面的两个例子，我们就可以得到和我们首先优先级一样的结果了。
