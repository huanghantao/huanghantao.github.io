---
title: PHP内核生成zend_vm_opcodes.h
date: 2020-10-21 23:44:53
tags:
- PHP内核
---

> 本文基于的PHP8 commit为：14806e0824ecd598df74cac855868422e44aea53

首先，`zend_vm_opcodes.h`这个文件是通过脚本`Zend/zend_vm_gen.php`来生成的。而`zend_vm_gen.php`这个脚本依赖`zend_vm_def.h`和`zend_vm_execute.skl`来生成文件`zend_vm_execute.h`和`zend_vm_opcodes.h`：

```bash
 +--------------------+                +--------------------+ 
 |                    |                |                    | 
 |   zend_vm_def.h    |                |zend_vm_execute.skl | 
 |                    |                |                    | 
 +--------------------+                +--------------------+ 
            |                                     |           
            +------------------+------------------+           
                               |                              
                               v                              
                    +--------------------+                    
                    |                    |                    
                    |  zend_vm_gen.php   |                    
                    |                    |                    
                    +--------------------+                    
                               |                              
           +-------------------+-------------------+          
           |                                       |          
           v                                       v          
+--------------------+                  +--------------------+
|                    |                  |                    |
| zend_vm_opcodes.h  |                  | zend_vm_execute.h  |
|                    |                  |                    |
+--------------------+                  +--------------------+
```

我们以文件`zend_vm_gen.php`分析的起点，来看看生成`zend_vm_opcodes.h`的`zend_vm_execute.h`的关键步骤。

首先，是函数`gen_vm`。这个函数会逐行扫描`zend_vm_def.h`里面的代码。

当扫描到`ZEND_VM_HELPER`的时候，就会执行下面的代码：

```php
if (strpos($line,"ZEND_VM_HELPER(") === 0 ||
            strpos($line,"ZEND_VM_INLINE_HELPER(") === 0 ||
            strpos($line,"ZEND_VM_COLD_HELPER(") === 0 ||
            strpos($line,"ZEND_VM_HOT_HELPER(") === 0) {
    // Parsing helper's definition
    if (preg_match(
            "/^ZEND_VM(_INLINE|_COLD|_HOT)?_HELPER\(\s*([A-Za-z_]+)\s*,\s*([A-Z_|]+)\s*,\s*([A-Z_|]+)\s*(?:,\s*SPEC\(([A-Z_|=,]+)\)\s*)?(?:,\s*([^)]*)\s*)?\)/",
            $line,
            $m) == 0) {
        die("ERROR ($def:$lineno): Invalid ZEND_VM_HELPER definition.\n");
    }
    $inline = !empty($m[1]) && $m[1] === "_INLINE";
    $cold   = !empty($m[1]) && $m[1] === "_COLD";
    $hot    = !empty($m[1]) && $m[1] === "_HOT";
    $helper = $m[2];
    $op1    = parse_operand_spec($def, $lineno, $m[3], $flags1);
    $op2    = parse_operand_spec($def, $lineno, $m[4], $flags2);
    $param  = isset($m[6]) ? $m[6] : null;
    if (isset($helpers[$helper])) {
        die("ERROR ($def:$lineno): Helper with name '$helper' is already defined.\n");
    }

    // Store parameters
    if (ZEND_VM_KIND == ZEND_VM_KIND_GOTO
        || ZEND_VM_KIND == ZEND_VM_KIND_SWITCH
        || (ZEND_VM_KIND == ZEND_VM_KIND_HYBRID && $hot)) {
        foreach (explode(",", $param) as $p) {
            $p = trim($p);
            if ($p !== "") {
                $params[$p] = 1;
            }
        }
    }

    $helpers[$helper] = array("op1"=>$op1,"op2"=>$op2,"param"=>$param,"code"=>"","inline"=>$inline,"cold"=>$cold,"hot"=>$hot);

    if (!empty($m[5])) {
        $helpers[$helper]["spec"] = parse_spec_rules($def, $lineno, $m[5]);
    }

    $handler = null;
    $list[$lineno] = array("helper"=>$helper);
```

这段代码具体的细节我们不去深究，总结起来就是去正则匹配`zend_vm_def.h`里面当前行的`ZEND_VM_HELPER`，然后把相关的信息存在全局变量`$helpers`里面。例如：

```bash
ZEND_VM_HELPER(zend_add_helper, ANY, ANY, zval *op_1, zval *op_2)
=>
[
    "zend_add_helper" =>
    [
        "op1" => [
            ANY:0
        ],
        "op2" => [
            ANY:0
        ],
        "param" => "zval *op_1, zval *op_2",
        "code" => "",
        "inline" => false,
        "cold" => false,
        "hot" => false,
    ]
]
```

然后

```cpp
else if ($handler !== null) {
    // Add line of code to current opcode handler
    $opcodes[$handler]["code"] .= $line;
} else if ($helper !== null) {
    // Add line of code to current helper
    $helpers[$helper]["code"] .= $line;
}
```

就是去拼接`zend_vm_def.h`里面的代码。如果是`ZEND_VM_HELPER`类型的代码，就执行`$helpers[$helper]["code"] .= $line;`。例如，当拼接完毕的时候，就会得到下面的信息：

```bash
ZEND_VM_HELPER(zend_add_helper, ANY, ANY, zval *op_1, zval *op_2)
{
    USE_OPLINE

    SAVE_OPLINE();
    if (UNEXPECTED(Z_TYPE_INFO_P(op_1) == IS_UNDEF)) {
        op_1 = ZVAL_UNDEFINED_OP1();
    }
    if (UNEXPECTED(Z_TYPE_INFO_P(op_2) == IS_UNDEF)) {
        op_2 = ZVAL_UNDEFINED_OP2();
    }
    add_function(EX_VAR(opline->result.var), op_1, op_2);
    if (OP1_TYPE & (IS_TMP_VAR|IS_VAR)) {
        zval_ptr_dtor_nogc(op_1);
    }
    if (OP2_TYPE & (IS_TMP_VAR|IS_VAR)) {
        zval_ptr_dtor_nogc(op_2);
    }
    ZEND_VM_NEXT_OPCODE_CHECK_EXCEPTION();
}
=>
[
    "zend_add_helper" =>
    [
        "op1" => [
            ANY:0
        ],
        "op2" => [
            ANY:0
        ],
        "param" => "zval *op_1, zval *op_2",
        "code" => " USE_OPLINE
                    SAVE_OPLINE();
                    if (UNEXPECTED(Z_TYPE_INFO_P(op_1) == IS_UNDEF)) {
                        op_1 = ZVAL_UNDEFINED_OP1();
                    }
                    if (UNEXPECTED(Z_TYPE_INFO_P(op_2) == IS_UNDEF)) {
                        op_2 = ZVAL_UNDEFINED_OP2();
                    }
                    add_function(EX_VAR(opline->result.var), op_1, op_2);
                    if (OP1_TYPE & (IS_TMP_VAR|IS_VAR)) {
                        zval_ptr_dtor_nogc(op_1);
                    }
                    if (OP2_TYPE & (IS_TMP_VAR|IS_VAR)) {
                        zval_ptr_dtor_nogc(op_2);
                    }
                    ZEND_VM_NEXT_OPCODE_CHECK_EXCEPTION();",
        "inline" => false,
        "cold" => false,
        "hot" => false,
    ]
]
```

当扫描到`ZEND_VM_HANDLER`的代码之后，就会执行下面的代码：

```php
if (strpos($line,"ZEND_VM_HANDLER(") === 0 ||
    strpos($line,"ZEND_VM_INLINE_HANDLER(") === 0 ||
    strpos($line,"ZEND_VM_HOT_HANDLER(") === 0 ||
    strpos($line,"ZEND_VM_HOT_NOCONST_HANDLER(") === 0 ||
    strpos($line,"ZEND_VM_HOT_NOCONSTCONST_HANDLER(") === 0 ||
    strpos($line,"ZEND_VM_HOT_SEND_HANDLER(") === 0 ||
    strpos($line,"ZEND_VM_HOT_OBJ_HANDLER(") === 0 ||
    strpos($line,"ZEND_VM_COLD_HANDLER(") === 0 ||
    strpos($line,"ZEND_VM_COLD_CONST_HANDLER(") === 0 ||
    strpos($line,"ZEND_VM_COLD_CONSTCONST_HANDLER(") === 0) {
    // Parsing opcode handler's definition
    if (preg_match(
            "/^ZEND_VM_(HOT_|INLINE_|HOT_OBJ_|HOT_SEND_|HOT_NOCONST_|HOT_NOCONSTCONST_|COLD_|COLD_CONST_|COLD_CONSTCONST_)?HANDLER\(\s*([0-9]+)\s*,\s*([A-Z_]+)\s*,\s*([A-Z_|]+)\s*,\s*([A-Z_|]+)\s*(,\s*([A-Z_|]+)\s*)?(,\s*SPEC\(([A-Z_|=,]+)\)\s*)?\)/",
            $line,
            $m) == 0) {
        die("ERROR ($def:$lineno): Invalid ZEND_VM_HANDLER definition.\n");
    }
    $hot = !empty($m[1]) ? $m[1] : false;
    $code = (int)$m[2];
    $op   = $m[3];
    $len  = strlen($op);
    $op1  = parse_operand_spec($def, $lineno, $m[4], $flags1);
    $op2  = parse_operand_spec($def, $lineno, $m[5], $flags2);
    $flags = $flags1 | ($flags2 << 8);
    if (!empty($m[7])) {
        $flags |= parse_ext_spec($def, $lineno, $m[7]);
    }

    if ($len > $max_opcode_len) {
        $max_opcode_len = $len;
    }
    if ($code > $max_opcode) {
        $max_opcode = $code;
    }
    if (isset($opcodes[$code])) {
        die("ERROR ($def:$lineno): Opcode with code '$code' is already defined.\n");
    }
    if (isset($opnames[$op])) {
        die("ERROR ($def:$lineno): Opcode with name '$op' is already defined.\n");
    }
    $opcodes[$code] = array("op"=>$op,"op1"=>$op1,"op2"=>$op2,"code"=>"","flags"=>$flags,"hot"=>$hot);
    if (isset($m[9])) {
        $opcodes[$code]["spec"] = parse_spec_rules($def, $lineno, $m[9]);
        if (isset($opcodes[$code]["spec"]["NO_CONST_CONST"])) {
            $opcodes[$code]["flags"] |= $vm_op_flags["ZEND_VM_NO_CONST_CONST"];
        }
        if (isset($opcodes[$code]["spec"]["COMMUTATIVE"])) {
            $opcodes[$code]["flags"] |= $vm_op_flags["ZEND_VM_COMMUTATIVE"];
        }
    }
    $opnames[$op] = $code;
    $handler = $code;
    $helper = null;
    $list[$lineno] = array("handler"=>$handler);
    }
```

这段代码具体的细节我们不去深究，总结起来就是去正则匹配`zend_vm_def.h`里面当前行的`ZEND_VM_HANDLER`，然后把相关的信息存在全局变量`$opcodes`里面。例如：

```bash
ZEND_VM_HOT_NOCONSTCONST_HANDLER(1, ZEND_ADD, CONST|TMPVARCV, CONST|TMPVARCV)
=>
[
    1 =>
    [
        "op" => "ZEND_ADD",
        "op1" => [
            "CONST" => 0,
            "TMPVARCV" => 1
        ],
        "op2" => [
            "CONST" => 0,
            "TMPVARCV" => 1
        ],
        "code" => "",
        "flags" => 2827,
        "hot" => "HOT_NOCONSTCONST_"
    ]
]
```

其中

```bash
1 => [
    "op" => "ZEND_ADD"
]
```

实际上就是`ZEND_VM_HOT_NOCONSTCONST_HANDLER(1, ZEND_ADD, CONST|TMPVARCV, CONST|TMPVARCV)`里面的`1`和`ZEND_ADD`，这会用来定义`opcode`，对应`zend_vm_opcodes.h`文件里面的：

```cpp
#define ZEND_ADD 1
```

```bash
"CONST" => 0,
"TMPVARCV" => 1
```

代表`CONST|TMPVARCV`的序号。实际上就是：

```php
array_flip(explode("|", CONST|TMPVARCV))
```

之后的结果。

```bash
"flags" => 2827
```

计算方法是`(CONST|TMPVARCV) | ((CONST|TMPVARCV) << 8)`。至于`CONST`和`TMPVARCV`的值，我们可以在文件`zend_vm_gen.php`的变量`$vm_op_decode`里面找到。

接着，对于`ZEND_VM_HANDLER`就会执行`$opcodes[$handler]["code"] .= $line;`了，和`ZEND_VM_HELPER`的类似。

```php
// Generate opcode #defines (zend_vm_opcodes.h)
$code_len = strlen((string)$max_opcode);
$f = fopen(__DIR__ . "/zend_vm_opcodes.h", "w+") or die("ERROR: Cannot create zend_vm_opcodes.h\n");

// Insert header
out($f, HEADER_TEXT);
fputs($f, "#ifndef ZEND_VM_OPCODES_H\n#define ZEND_VM_OPCODES_H\n\n");
fputs($f, "#define ZEND_VM_SPEC\t\t" . ZEND_VM_SPEC . "\n");
fputs($f, "#define ZEND_VM_LINES\t\t" . ZEND_VM_LINES . "\n");
fputs($f, "#define ZEND_VM_KIND_CALL\t" . ZEND_VM_KIND_CALL . "\n");
fputs($f, "#define ZEND_VM_KIND_SWITCH\t" . ZEND_VM_KIND_SWITCH . "\n");
fputs($f, "#define ZEND_VM_KIND_GOTO\t" . ZEND_VM_KIND_GOTO . "\n");
fputs($f, "#define ZEND_VM_KIND_HYBRID\t" . ZEND_VM_KIND_HYBRID . "\n");
if ($GLOBALS["vm_kind_name"][ZEND_VM_KIND] === "ZEND_VM_KIND_HYBRID") {
    fputs($f, "/* HYBRID requires support for computed GOTO and global register variables*/\n");
    fputs($f, "#if (defined(__GNUC__) && defined(HAVE_GCC_GLOBAL_REGS))\n");
    fputs($f, "# define ZEND_VM_KIND\t\tZEND_VM_KIND_HYBRID\n");
    fputs($f, "#else\n");
    fputs($f, "# define ZEND_VM_KIND\t\tZEND_VM_KIND_CALL\n");
    fputs($f, "#endif\n");
} else {
    fputs($f, "#define ZEND_VM_KIND\t\t" . $GLOBALS["vm_kind_name"][ZEND_VM_KIND] . "\n");
}
fputs($f, "\n");
```

这段代码就很简单了，直接往`zend_vm_opcodes.h`文件里面写这些内容。

```php
foreach($vm_op_flags as $name => $val) {
    fprintf($f, "#define %-24s 0x%08x\n", $name, $val);
}
```

这段代码是把`zend_vm_gen.php`文件里面的`$vm_op_flags`内容以`16`进制的格式写在`zend_vm_opcodes.h`文件里面：

```cpp
$vm_op_flags = array(
    "ZEND_VM_OP_SPEC"         => 1<<0,
    "ZEND_VM_OP_CONST"        => 1<<1,
    // 省略其他的
);

=>

#define ZEND_VM_OP_SPEC          0x00000001
#define ZEND_VM_OP_CONST         0x00000002
// 省略其他的
```

接着

```php
foreach ($opcodes as $code => $dsc) {
    $code = str_pad((string)$code,$code_len," ",STR_PAD_LEFT);
    $op = str_pad($dsc["op"],$max_opcode_len);
    if ($code <= $max_opcode) {
        fputs($f,"#define $op $code\n");
    }
}
```

会去用我们上面搜集好的`$opcodes`来定义我们的`opcode`，例如：

```cpp
#define ZEND_NOP                          0
#define ZEND_ADD                          1
// 省略其他的
```

接着

```php
$code = str_pad((string)$max_opcode,$code_len," ",STR_PAD_LEFT);
$op = str_pad("ZEND_VM_LAST_OPCODE",$max_opcode_len);
fputs($f,"\n#define $op $code\n");

fputs($f, "\n#endif\n");
```

会去定义`PHP`内核一共有多少个`opcode`，例如：

```cpp
#define ZEND_VM_LAST_OPCODE             199
```

至此，我们的`zend_vm_opcodes.h`文件生成完毕了。接着，开始生成`zend_vm_opcodes.c`文件。

其中：

```php
fputs($f,"static const char *zend_vm_opcodes_names[".($max_opcode + 1)."] = {\n");
for ($i = 0; $i <= $max_opcode; $i++) {
    fputs($f,"\t".(isset($opcodes[$i]["op"])?'"'.$opcodes[$i]["op"].'"':"NULL").",\n");
}
fputs($f, "};\n\n");
```

用来定义我们所有`opcode`对应的名字，例如：

```cpp
static const char *zend_vm_opcodes_names[200] = {
    "ZEND_NOP",
    "ZEND_ADD",
    // 省略其他的
};
```

这个`zend_vm_opcodes_names`数组的索引实际上就是`opcode`对应的`id`。所以，如果我们要得到一个`opcode`的名字，那么可以通过以下方式拿到：

```cpp
zend_vm_opcodes_names[ZEND_ADD]
=>
"ZEND_ADD"
```

接着

```php
fputs($f,"static uint32_t zend_vm_opcodes_flags[".($max_opcode + 1)."] = {\n");
for ($i = 0; $i <= $max_opcode; $i++) {
    fprintf($f, "\t0x%08x,\n", isset($opcodes[$i]["flags"]) ? $opcodes[$i]["flags"] : 0);
}
fputs($f, "};\n\n");
```

用来定义`opcode`对应的`flags`。例如：

```cpp
static uint32_t zend_vm_opcodes_flags[200] = {
    0x00000000,
    0x00000b0b,
    // 省略其他的
};
```

`flags`的值的算法我们已经在上面介绍过了，这里再总结下：

```php
$flags = $flags1 | ($flags2 << 8);
```

接着：

```php
fputs($f, "ZEND_API const char* ZEND_FASTCALL zend_get_opcode_name(zend_uchar opcode) {\n");
fputs($f, "\tif (UNEXPECTED(opcode > ZEND_VM_LAST_OPCODE)) {\n");
fputs($f, "\t\treturn NULL;\n");
fputs($f, "\t}\n");
fputs($f, "\treturn zend_vm_opcodes_names[opcode];\n");
fputs($f, "}\n");
```

定义一个获取`opcode name`的函数。生成的结果如下：

```cpp
ZEND_API const char* ZEND_FASTCALL zend_get_opcode_name(zend_uchar opcode) {
    if (UNEXPECTED(opcode > ZEND_VM_LAST_OPCODE)) {
        return NULL;
    }
    return zend_vm_opcodes_names[opcode];
}
```

首先是判断一下是否有这个`opcode`，有的话返回它的`name`，没有的话返回`NULL`。

接着：

```php
puts($f, "ZEND_API uint32_t ZEND_FASTCALL zend_get_opcode_flags(zend_uchar opcode) {\n");
fputs($f, "\tif (UNEXPECTED(opcode > ZEND_VM_LAST_OPCODE)) {\n");
fputs($f, "\t\topcode = ZEND_NOP;\n");
fputs($f, "\t}\n");
fputs($f, "\treturn zend_vm_opcodes_flags[opcode];\n");
fputs($f, "}\n");
```

定义一个获取`opcode flags`的函数。生成的结果如下：

```cpp
ZEND_API uint32_t ZEND_FASTCALL zend_get_opcode_flags(zend_uchar opcode) {
    if (UNEXPECTED(opcode > ZEND_VM_LAST_OPCODE)) {
        opcode = ZEND_NOP;
    }
    return zend_vm_opcodes_flags[opcode];
}
```

首先是判断一下是否有这个`opcode`，有的话返回它的`flags`，没有的话返回`ZEND_NOP`的`flags`（也就是`0`）。

至此，我们的`zend_vm_opcodes.c`文件生成完毕了。接着，开始生成`zend_vm_execute.h`文件。

```php
// Support for ZEND_USER_OPCODE
out($f, "static user_opcode_handler_t zend_user_opcode_handlers[256] = {\n");
for ($i = 0; $i < 255; ++$i) {
    out($f, "\t(user_opcode_handler_t)NULL,\n");
}
out($f, "\t(user_opcode_handler_t)NULL\n};\n\n");
```

用来定义一个`zend_user_opcode_handlers`数组，这个数组初始的时候全都是`NULL`。生成结果如下：

```cpp
static user_opcode_handler_t zend_user_opcode_handlers[256] = {
    (user_opcode_handler_t)NULL,
    (user_opcode_handler_t)NULL,
    (user_opcode_handler_t)NULL,
    // 省略其他的
    (user_opcode_handler_t)NULL
};
```

接着：

```php
out($f, "static zend_uchar zend_user_opcodes[256] = {");
for ($i = 0; $i < 255; ++$i) {
    if ($i % 16 == 1) out($f, "\n\t");
    out($f, "$i,");
}
out($f, "255\n};\n\n");
```

用来定义我们的`zend_user_opcodes`，生成结果如下：

```cpp
static zend_uchar zend_user_opcodes[256] = {0,
    1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,
    17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,
    33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,
    49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,
    65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,
    81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,
    97,98,99,100,101,102,103,104,105,106,107,108,109,110,111,112,
    113,114,115,116,117,118,119,120,121,122,123,124,125,126,127,128,
    129,130,131,132,133,134,135,136,137,138,139,140,141,142,143,144,
    145,146,147,148,149,150,151,152,153,154,155,156,157,158,159,160,
    161,162,163,164,165,166,167,168,169,170,171,172,173,174,175,176,
    177,178,179,180,181,182,183,184,185,186,187,188,189,190,191,192,
    193,194,195,196,197,198,199,200,201,202,203,204,205,206,207,208,
    209,210,211,212,213,214,215,216,217,218,219,220,221,222,223,224,
    225,226,227,228,229,230,231,232,233,234,235,236,237,238,239,240,
    241,242,243,244,245,246,247,248,249,250,251,252,253,254,255
};
```

说明一共支持`256`个`zend_user_opcodes`。

接着，开始调用`gen_executor`来按照模板文件`Zend/zend_vm_execute.skl`生成代码。这个函数也是逐行扫描`zend_vm_execute.skl`文件。

其中`zend_vm_execute.skl`文件的第一行是：

```txt
{%DEFINES%}
```

意味着我们在`zend_vm_execute.h`里面需要生成一些定义。具体的生成过程如下：

```php
out($f,"#define SPEC_START_MASK        0x0000ffff\n");
out($f,"#define SPEC_EXTRA_MASK        0xfffc0000\n");
out($f,"#define SPEC_RULE_OP1          0x00010000\n");
out($f,"#define SPEC_RULE_OP2          0x00020000\n");
out($f,"#define SPEC_RULE_OP_DATA      0x00040000\n");
out($f,"#define SPEC_RULE_RETVAL       0x00080000\n");
out($f,"#define SPEC_RULE_QUICK_ARG    0x00100000\n");
out($f,"#define SPEC_RULE_SMART_BRANCH 0x00200000\n");
out($f,"#define SPEC_RULE_COMMUTATIVE  0x00800000\n");
out($f,"#define SPEC_RULE_ISSET        0x01000000\n");
out($f,"#define SPEC_RULE_OBSERVER     0x02000000\n");
```

这是一些`opcode`对应的操作数的规则，例如`SPEC_RULE_OP1`意味着需要用到操作数`1`，并且支持的类型至少是`2`种。对应的代码如下：

```php
if (isset($dsc["op1"]) && !isset($dsc["op1"]["ANY"])) {
    $count = 0;
    foreach ($op_types_ex as $t) {
        if (isset($dsc["op1"][$t])) {
            $def_op1_type = $t;
            $count++;
        }
    }
    if ($count > 1) {
        $spec_op1 = true;
        $specs[$num] .= " | SPEC_RULE_OP1";
        $def_op1_type = "ANY";
    }
}
```

接着：

```php
out($f,"static const uint32_t *zend_spec_handlers;\n");
out($f,"static const void * const *zend_opcode_handlers;\n");
out($f,"static int zend_handlers_count;\n");
if ($kind == ZEND_VM_KIND_HYBRID) {
    out($f,"#if (ZEND_VM_KIND == ZEND_VM_KIND_HYBRID)\n");
    out($f,"static const void * const * zend_opcode_handler_funcs;\n");
    out($f,"static zend_op hybrid_halt_op;\n");
    out($f,"#endif\n");
}
out($f,"#if (ZEND_VM_KIND != ZEND_VM_KIND_HYBRID) || !ZEND_VM_SPEC\n");
out($f,"static const void *zend_vm_get_opcode_handler(zend_uchar opcode, const zend_op* op);\n");
out($f,"#endif\n\n");
if ($kind == ZEND_VM_KIND_HYBRID) {
    out($f,"#if (ZEND_VM_KIND == ZEND_VM_KIND_HYBRID)\n");
    out($f,"static const void *zend_vm_get_opcode_handler_func(zend_uchar opcode, const zend_op* op);\n");
    out($f,"#else\n");
    out($f,"# define zend_vm_get_opcode_handler_func zend_vm_get_opcode_handler\n");
    out($f,"#endif\n\n");
}
```

这个是根据`ZEND_VM_KIND`来定义一些变量和函数，生成结果如下：

```cpp
static const uint32_t *zend_spec_handlers;
static const void * const *zend_opcode_handlers;
static int zend_handlers_count;
#if (ZEND_VM_KIND == ZEND_VM_KIND_HYBRID)
static const void * const * zend_opcode_handler_funcs;
static zend_op hybrid_halt_op;
#endif
#if (ZEND_VM_KIND != ZEND_VM_KIND_HYBRID) || !ZEND_VM_SPEC
static const void *zend_vm_get_opcode_handler(zend_uchar opcode, const zend_op* op);
#endif

#if (ZEND_VM_KIND == ZEND_VM_KIND_HYBRID)
static const void *zend_vm_get_opcode_handler_func(zend_uchar opcode, const zend_op* op);
#else
# define zend_vm_get_opcode_handler_func zend_vm_get_opcode_handler
#endif
```

`zend_vm_gen.php`默认是`ZEND_VM_KIND_HYBRID`模式。

接着，会有一大段的代码来定义一些如下宏：

```cpp
HYBRID_NEXT()
HYBRID_SWITCH()
HYBRID_CASE(op)
HYBRID_BREAK()
HYBRID_DEFAULT
```

接着，会调用`gen_executor_code`来生成`opcode`的详细`handler`。例如，我们的操作数有如下类型：

```php
$op_types_ex = array(
    "ANY",
    "CONST",
    "TMPVARCV",
    "TMPVAR",
    "TMP",
    "VAR",
    "UNUSED",
    "CV",
);
```

那么，就最大就会有`op1_type * op1_type`个`handler`。所以，就会有如下代码：

```php
// Produce specialized executor
$op1t = $op_types_ex;
// for each op1.op_type
foreach($op1t as $op1) {
    $op2t = $op_types_ex;
    // for each op2.op_type
    foreach($op2t as $op2) {
        // for each handlers in helpers in original order
        foreach ($list as $lineno => $dsc) {
            if (isset($dsc["handler"])) {
                $num = $dsc["handler"];
                foreach (extra_spec_handler($opcodes[$num]) as $extra_spec) {
                    // Check if handler accepts such types of operands (op1 and op2)
                    if (isset($opcodes[$num]["op1"][$op1]) &&
                        isset($opcodes[$num]["op2"][$op2])) {
                        // Generate handler code
                        gen_handler($f, 1, $kind, $opcodes[$num]["op"], $op1, $op2, isset($opcodes[$num]["use"]), $opcodes[$num]["code"], $lineno, $opcodes[$num], $extra_spec, $switch_labels);
                    }
                }
            } else if (isset($dsc["helper"])) {
                $num = $dsc["helper"];
                foreach (extra_spec_handler($helpers[$num]) as $extra_spec) {
                    // Check if handler accepts such types of operands (op1 and op2)
                    if (isset($helpers[$num]["op1"][$op1]) &&
                        isset($helpers[$num]["op2"][$op2])) {
                        // Generate helper code
                        gen_helper($f, 1, $kind, $num, $op1, $op2, $helpers[$num]["param"], $helpers[$num]["code"], $lineno, $helpers[$num]["inline"], $helpers[$num]["cold"], $helpers[$num]["hot"], $extra_spec);
                    }
                }
            } else {
                var_dump($dsc);
                die("??? $kind:$num\n");
            }
        }
    }
}
```

对于这段代码，`$list`里面存放了所有的`helper`的名字和`opcode`的值，例如：

```php
"helper" => "zend_add_helper",
"handler" => 1,
"helper" => "zend_sub_helper",
"handler" => 2,
// 省略其他的内容
```

如果是`helper`，那么我们从`$helpers`里面获取到这个`helper`函数的信息。

如果是`handler`，那么我们从`$opcodes`里面获取到这个`opcode`的信息。

其中：

```php
$opcodes[$num]["op1"]
$opcodes[$num]["op2"]
```

里面存放的就是这个`opcode`对应的操作数`1`和操作数`2`支持的所有类型，我们在前面解析的时候就拿到了这些信息。

无论是是`helper`还是`opcode`类型的`handler`，都会调用`extra_spec_handler`来生成`spec`函数。在生成`spec`的时候，会将`zend_vm_def.h`里面对应的`handler`的`code`进行替换，替换的规则在函数`gen_code`里面。

生成了`handler`对应的`specs`之后，就完成了模板文件里面`{%DEFINES%}`的替换了。

接着，开始替换模板文件里面的`{%EXECUTOR_NAME%}`，也就是开始生成我们的`zend_execute`函数了：

```php
case "EXECUTOR_NAME":
    out($f, $m[1].$executor_name.$m[3]."\n");
    break;
```

这里是名字是`execute`。

接着替换模板文件的`{%HELPER_VARS%}`：

```php
case "HELPER_VARS":
    // 省略代码
    break;
```

生成结果如下：

```cpp
#ifdef ZEND_VM_IP_GLOBAL_REG
    const zend_op *orig_opline = opline;
#endif
#ifdef ZEND_VM_FP_GLOBAL_REG
    zend_execute_data *orig_execute_data = execute_data;
    execute_data = ex;
#else
    zend_execute_data *execute_data = ex;
#endif
```

接着替换模板文件的`{%INTERNAL_LABELS%}`：

```php
out($f,$prolog."if (UNEXPECTED(execute_data == NULL)) {\n");
out($f,$prolog."\tstatic const void * const labels[] = {\n");
gen_labels($f, $spec, ($kind == ZEND_VM_KIND_HYBRID) ? ZEND_VM_KIND_GOTO : $kind, $prolog."\t\t", $specs);
out($f,$prolog."\t};\n");
```

这里定义了一个名字叫做`labels`的静态变量，也就意味着每次调用`zend_execute`是共享的。生成的代码如下：

```cpp
if (UNEXPECTED(execute_data == NULL)) {
    static const void * const labels[] = {
        (void*)&&ZEND_NOP_SPEC_LABEL,
        (void*)&&ZEND_ADD_SPEC_CONST_CONST_LABEL,
        (void*)&&ZEND_ADD_SPEC_CONST_TMPVARCV_LABEL,
        (void*)&&ZEND_ADD_SPEC_CONST_TMPVARCV_LABEL,
        (void*)&&ZEND_NULL_LABEL,
        (void*)&&ZEND_ADD_SPEC_CONST_TMPVARCV_LABEL,
        (void*)&&ZEND_ADD_SPEC_TMPVARCV_CONST_LABEL,
        (void*)&&ZEND_ADD_SPEC_TMPVARCV_TMPVARCV_LABEL,
        (void*)&&ZEND_ADD_SPEC_TMPVARCV_TMPVARCV_LABEL,
        (void*)&&ZEND_NULL_LABEL,
        (void*)&&ZEND_ADD_SPEC_TMPVARCV_TMPVARCV_LABEL,
        (void*)&&ZEND_ADD_SPEC_TMPVARCV_CONST_LABEL,
        (void*)&&ZEND_ADD_SPEC_TMPVARCV_TMPVARCV_LABEL,
        (void*)&&ZEND_ADD_SPEC_TMPVARCV_TMPVARCV_LABEL,
        (void*)&&ZEND_NULL_LABEL,
        (void*)&&ZEND_ADD_SPEC_TMPVARCV_TMPVARCV_LABEL,
        (void*)&&ZEND_NULL_LABEL,
        (void*)&&ZEND_NULL_LABEL,
        (void*)&&ZEND_NULL_LABEL,
        (void*)&&ZEND_NULL_LABEL,
        (void*)&&ZEND_NULL_LABEL,
        (void*)&&ZEND_ADD_SPEC_TMPVARCV_CONST_LABEL,
        (void*)&&ZEND_ADD_SPEC_TMPVARCV_TMPVARCV_LABEL,
        (void*)&&ZEND_ADD_SPEC_TMPVARCV_TMPVARCV_LABEL,
        (void*)&&ZEND_NULL_LABEL,
        (void*)&&ZEND_ADD_SPEC_TMPVARCV_TMPVARCV_LABEL,
        (void*)&&ZEND_NULL_LABEL
        // 省略其他的内容
    };
```

也就意味着，当第一次调用`zend_execute`的时候，会初始化这个`labels`变量。

接着，我们会生成一堆的`HYBRID_SWITCH`和`HYBRID_CASE`。这个和`labels`变量里面的指针是对应的，并且和我们生成的`handler`是对应的。我们后面会写一个小`demo`来解释下这个`switch ... case`的原理。

接着，会生成`$specs`：

```cpp
static const uint32_t specs[] = {
    0,
    1 | SPEC_RULE_OP1 | SPEC_RULE_OP2,
    26 | SPEC_RULE_OP1 | SPEC_RULE_OP2,
    51 | SPEC_RULE_OP1 | SPEC_RULE_OP2 | SPEC_RULE_COMMUTATIVE,
    // 省略其他的
};
```

其中，`SPEC_RULE_OP1`和`SPEC_RULE_OP2`解释过了。那么它们前面的数字是什么呢？实际上，前面的数字是第一个当前`opcode`的第一个`spec handler`在`labels`变量的索引。这么说比较抽象，我用下面的图来解释一下：

```bash
  +-------------+                                   +-------------+          
  |    specs    |                                   |   labels    |          
  +-------------+                                   +-------------+          
                                                                             
                                                                             
+------+----+------+                       +--------------------------------+
|      | 0  |      |---------------------->|      ZEND_NOP_SPEC_LABEL       |
+------+----+------+                       +--------------------------------+
|      | 1  |      |---------------------->|ZEND_ADD_SPEC_CONST_CONST_LABEL |
+------+----+------+                       +--------------------------------+
|      | 26 |      |-----------+           |ZEND_ADD_SPEC_CONST_TMPVARCV_LAB|
+------+----+------+           |           +--------------------------------+
|      | 51 |      |           |           |              ...               |
+------+----+------+           |           +--------------------------------+
|                  |           +---------->|ZEND_SUB_SPEC_CONST_CONST_LABEL |
|                  |                       +--------------------------------+
|       ...        |                       |ZEND_SUB_SPEC_CONST_TMPVARCV_LAB|
|                  |                       +--------------------------------+
|                  |                       |              ...               |
|                  |                       |                                |
+------------------+                       +--------------------------------+
```

至此，`zend_vm_gen.php`生成代码的过程结束了。
