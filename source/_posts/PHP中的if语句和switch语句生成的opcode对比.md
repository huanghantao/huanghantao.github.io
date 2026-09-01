---
title: PHP中的if语句和switch语句生成的opcode对比
date: 2021-02-24 10:08:03
tags:
- PHP
- PHP内核
- 编译原理
---

我们有如下测试脚本：

```php
<?php

$variable = 1;

if ($variable == 1) {
    var_dump($variable);
} else if ($variable == 2) {
    var_dump($variable);
} else if ($variable == 3) {
    var_dump($variable);
}

var_dump(4);
```

生成的`opcodes`如下：

```bash
[Stack in /root/codeDir/phpCode/test/test.php (22 ops)]
L1-14 {main}() /root/codeDir/phpCode/test/test.php - 0x7fdcaca80000 + 22 ops
 L3    #0     ASSIGN                  $variable            1
 L5    #1     IS_EQUAL                $variable            1
 L5    #2     JMPZ                    ~1                   J7
 L6    #3     INIT_FCALL<1>           96                   "var_dump"
 L6    #4     SEND_VAR                $variable            1
 L6    #5     DO_ICALL
 L6    #6     JMP                     J18
 L7    #7     IS_EQUAL                $variable            2
 L7    #8     JMPZ                    ~3                   J13
 L8    #9     INIT_FCALL<1>           96                   "var_dump"
 L8    #10    SEND_VAR                $variable            1
 L8    #11    DO_ICALL
 L8    #12    JMP                     J18
 L9    #13    IS_EQUAL                $variable            3
 L9    #14    JMPZ                    ~5                   J18
 L10   #15    INIT_FCALL<1>           96                   "var_dump"
 L10   #16    SEND_VAR                $variable            1
 L10   #17    DO_ICALL
 L13   #18    INIT_FCALL<1>           96                   "var_dump"
 L13   #19    SEND_VAL                4                    1
 L13   #20    DO_ICALL
 L14   #21    RETURN<-1>              1
prompt>
```

我们来看`PHP`代码分别对应的`opcodes`（我没有完全按照基本块来划分，只是简单的按照代码结构来划分）：

第一段：

```php
$variable = 1;

L3    #0     ASSIGN                  $variable            1
```

第二段：

```php
if ($variable == 1) {
    var_dump($variable);
}

L5    #1     IS_EQUAL                $variable            1
L5    #2     JMPZ                    ~1                   J7
L6    #3     INIT_FCALL<1>           96                   "var_dump"
L6    #4     SEND_VAR                $variable            1
L6    #5     DO_ICALL
L6    #6     JMP                     J18
```

第三段：

```php
else if ($variable == 2) {
    var_dump($variable);
}

L7    #7     IS_EQUAL                $variable            2
L7    #8     JMPZ                    ~3                   J13
L8    #9     INIT_FCALL<1>           96                   "var_dump"
L8    #10    SEND_VAR                $variable            1
L8    #11    DO_ICALL
L8    #12    JMP                     J18
```

第四段：

```php
else if ($variable == 3) {
    var_dump($variable);
}

L9    #13    IS_EQUAL                $variable            3
L9    #14    JMPZ                    ~5                   J18
L10   #15    INIT_FCALL<1>           96                   "var_dump"
L10   #16    SEND_VAR                $variable            1
L10   #17    DO_ICALL
```

第五段：

```php
var_dump(4);

L13   #18    INIT_FCALL<1>           96                   "var_dump"
L13   #19    SEND_VAL                4                    1
L13   #20    DO_ICALL
L14   #21    RETURN<-1>              1
```

我们会发现，这里的`IS_EQUAL ... JMPZ`是分布在每一个块里面的。

我们翻译成对应的`switch`语句：

```php
<?php

$variable = 1;
switch ($variable) {
    case 1:
        var_dump($variable);
        break;
    case 2:
        var_dump($variable);
        break;
    case 3:
        var_dump($variable);
        break;
}
var_dump(4);
```

对应的`opcodes`如下：

```bash
[Stack in /root/codeDir/phpCode/test/test.php (24 ops)]
L1-15 {main}() /root/codeDir/phpCode/test/test.php - 0x7f7780480000 + 24 ops
 L3    #0     ASSIGN                  $variable            1
 L5    #1     IS_EQUAL                $variable            1
 L5    #2     JMPNZ                   ~1                   J8
 L8    #3     IS_EQUAL                $variable            2
 L8    #4     JMPNZ                   ~1                   J12
 L11   #5     IS_EQUAL                $variable            3
 L11   #6     JMPNZ                   ~1                   J16
 L11   #7     JMP                     J20
 L6    #8     INIT_FCALL<1>           96                   "var_dump"
 L6    #9     SEND_VAR                $variable            1
 L6    #10    DO_ICALL
 L7    #11    JMP                     J20
 L9    #12    INIT_FCALL<1>           96                   "var_dump"
 L9    #13    SEND_VAR                $variable            1
 L9    #14    DO_ICALL
 L10   #15    JMP                     J20
 L12   #16    INIT_FCALL<1>           96                   "var_dump"
 L12   #17    SEND_VAR                $variable            1
 L12   #18    DO_ICALL
 L13   #19    JMP                     J20
 L15   #20    INIT_FCALL<1>           96                   "var_dump"
 L15   #21    SEND_VAL                4                    1
 L15   #22    DO_ICALL
 L15   #23    RETURN<-1>              1
```

我们可以稍微划分一下。

第一段：

```php
$variable = 1;

L3    #0     ASSIGN                  $variable            1
```

第二段，所有的`switch ... case`组成一段：

```php
switch ... case

L5    #1     IS_EQUAL                $variable            1
L5    #2     JMPNZ                   ~1                   J8
L8    #3     IS_EQUAL                $variable            2
L8    #4     JMPNZ                   ~1                   J12
L11   #5     IS_EQUAL                $variable            3
L11   #6     JMPNZ                   ~1                   J16
L11   #7     JMP                     J20
```

第三段，我们可以把所有`case`里面的语句组成一段：

```php
L6    #8     INIT_FCALL<1>           96                   "var_dump"
L6    #9     SEND_VAR                $variable            1
L6    #10    DO_ICALL
L7    #11    JMP                     J20
L9    #12    INIT_FCALL<1>           96                   "var_dump"
L9    #13    SEND_VAR                $variable            1
L9    #14    DO_ICALL
L10   #15    JMP                     J20
L12   #16    INIT_FCALL<1>           96                   "var_dump"
L12   #17    SEND_VAR                $variable            1
L12   #18    DO_ICALL
L13   #19    JMP                     J20
```

第四段：

```php
var_dump(4);

L15   #20    INIT_FCALL<1>           96                   "var_dump"
L15   #21    SEND_VAL                4                    1
L15   #22    DO_ICALL
L15   #23    RETURN<-1>              1
```

我们会发现，`switch ... case`有一种`map`的感觉。无论有多少个`case`，我们这里都可以把`switch ... case`和`case`里面的语句划分成两部分。但是，如果是`if ... else`的话，随着分支的增加，段的数目也会跟着增加（再次提醒，我没有按照基本块来划分，因为按照基本块来划分，每一个`case`里面的语句都算一个基本块）。

那么，为什么我不把第一个脚本的代码里面的`if ... else`也划成一大段呢？因为我们会发现，如果我们划成一大段，`JMP`会在这一个大段里面跳来跳去。所以，我们也会发现，实际上，`switch ... case`是把`if ... else`的跳转关系集中放到了一块，而`if ... else`的跳转关系放在了每一个小段里面。

我们可以用如下流程图来描述这两种结构：

```bash
+----------------+
|       if       |
|                |
+----------------+
         |        
         |        
         |        
         |        
         v        
+----------------+
|    else if     |
|                |
+----------------+
         |        
         |        
         |        
         v        
+----------------+
|      else      |
|                |
+----------------+



                 +-------------------------+                
                 |         switch          |                
                 |                         |                
                 +-------------------------+                
                              |                             
                              |                             
         +--------------------+--------------------+        
         |                    |                    |        
         v                    v                    v        
+----------------+   +----------------+   +----------------+
|      case      |   |      case      |   |      case      |
|                |   |                |   |                |
+----------------+   +----------------+   +----------------+
```
