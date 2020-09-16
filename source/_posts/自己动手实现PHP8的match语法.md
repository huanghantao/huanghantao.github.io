---
title: 自己动手实现PHP8的match语法
date: 2020-07-15 22:20:58
tags:
- 编译原理
---

我们现在来实现一下`PHP8`的`match`语法。大概形如：

```php
echo match (1) {
    1 => 2
    2 => 3
}
```

首先，我们来看看这里有哪些`token`：

```php
1. echo 对应 T_ECHO
2. match 对应 T_MATCH
3. ( 对应 T_LEFT_PARENTHESIS
4. ) 对应 T_RIGHT_PARENTHESIS
5. 1 对应 T_NUMBER
6. => 对应 T_DOUBLE_ARROW
7. { 对应 T_LEFT_BRACE
8. } 对应 T_RIGHT_BRACE
```

所以，我们可以很轻易的写出解析`token`的规则（`match.l`）：

```cpp
%{
#include "match.tab.h"
%}

%%
echo    {return T_ECHO;}
match   {return T_MATCH;}
[(]     {return T_LEFT_PARENTHESIS;}
[)]     {return T_RIGHT_PARENTHESIS;}
[{]     {return T_LEFT_BRACE;}
[}]     {return T_RIGHT_BRACE;}
[0-9]+  {yylval = atoi(yytext); return T_NUMBER;}
=>      {return T_DOUBLE_ARROW;}
\n      /* ignore end of line */
[ \t]+  /* ignore whitespace */
%%
```

接着，我们要去定义我们的语法规则（`match.y`）：

```cpp
%{
#include <stdio.h>
#include <string.h>

extern int yylex(void);
extern int yyparse(void);
extern FILE *yyin;
extern int yylineno;

int map[100] = {0};

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

%token T_ECHO T_MATCH T_LEFT_PARENTHESIS T_RIGHT_PARENTHESIS T_NUMBER T_DOUBLE_ARROW T_LEFT_BRACE T_RIGHT_BRACE

%%

statement:
|   T_ECHO echo_expr    { printf("%d\n", $2); }
;

echo_expr:
expr
{
    $$ = $1;
};

expr:
|   match_expr { $$ = $1;}
;

match_expr:
T_MATCH T_LEFT_PARENTHESIS T_NUMBER T_RIGHT_PARENTHESIS T_LEFT_BRACE match_arm_list T_RIGHT_BRACE
{
    $6 = map[$3];
    $$ = $6;
};

match_arm_list:
| match_arm_list match_arm
;

match_arm:
T_NUMBER T_DOUBLE_ARROW T_NUMBER
{
    map[$1] = $3;
}

%%
```

语法规则会稍微难理解一点，我们来解释一下。其中，`statement`是起始的非终结符，可以推导出我们的`echo`表达式`echo_expr`；而我们的`echo`表达式实际上也是一个表达式，这个表达式可以是我们的`match`表达式`match_expr`；`match_expr`需要匹配到`match`、`（`等等这些`token`。但是，我们的匹配列表`match_arm_list`它不是终结符，我们还可以继续推导，可以发现，实际上`match_arm_list`是一个递归的，知道推导出`match_arm`某一项。一旦我们匹配到了`match_arm`，我们就把`key`和`value`保存在`map`里面。

我们来编译一下：

```bash
lex match.l

bison -d match.y

cc -o match lex.yy.c match.tab.c
```

此时会生成可执行文件`match`。

我们来写一下我们的测试脚本 （`match.php`）：

```php
echo match (1) {
    1 => 2
    2 => 3
}
```

然后执行脚本：

```bash
./match match.php
2
```

此时会输出`2`。如果我们把匹配条件改一下，改成`2`：

```php
echo match (2) {
    1 => 2
    2 => 3
}
```

将会输出`3`：

```bash
./match match.php
3
```

符合我们的预期。
