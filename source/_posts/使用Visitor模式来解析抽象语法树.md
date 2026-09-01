---
title: 使用Visitor模式来解析抽象语法树
date: 2020-08-30 17:19:47
tags:
- 编译原理
- PHP
---

我们在生成了`AST`之后，要做的事情就是去解析它。我们可以对它做很多的操作，比如修改`AST`的某些节点；对`AST`执行我们的语义操作，比如碰到`+`号，表示我们要做加法运算了，这样，我们就可以实现我们自己的语言了。

而`visitor`模式的思想就是，当我们遍历`AST`上的每一个节点的时候，都去执行我们注册的所有`visitor`。这样，我们可以让代码更加的优雅，我们只需要专注于实现当前`visitor`的功能即可，让`AST`的结构和对`AST`的操作解耦。

我以`PHP`为例，大致介绍下如何通过`visitor`模式来用`PHP`实现`PHP`。我们给我们的这门语言命名为`yaphp`（实际上，这正是我现在开发的语言，还未开源）。

我们有如下`yaphp`的代码：

```php
<?php

$a = 1 + 2 + 3 - 4;
$b = 2 * 3;
$c = $a + $b;
var_dump($c);
```

我们可以得到它的抽象语法树：

```php
array(
    0: Stmt_Expression(
        expr: Expr_Assign(
            var: Expr_Variable(
                name: a
            )
            expr: Expr_BinaryOp_Minus(
                left: Expr_BinaryOp_Plus(
                    left: Expr_BinaryOp_Plus(
                        left: Scalar_LNumber(
                            value: 1
                        )
                        right: Scalar_LNumber(
                            value: 2
                        )
                    )
                    right: Scalar_LNumber(
                        value: 3
                    )
                )
                right: Scalar_LNumber(
                    value: 4
                )
            )
        )
    )
    1: Stmt_Expression(
        expr: Expr_Assign(
            var: Expr_Variable(
                name: b
            )
            expr: Expr_BinaryOp_Mul(
                left: Scalar_LNumber(
                    value: 2
                )
                right: Scalar_LNumber(
                    value: 3
                )
            )
        )
    )
    2: Stmt_Expression(
        expr: Expr_Assign(
            var: Expr_Variable(
                name: c
            )
            expr: Expr_BinaryOp_Plus(
                left: Expr_Variable(
                    name: a
                )
                right: Expr_Variable(
                    name: b
                )
            )
        )
    )
    3: Stmt_Expression(
        expr: Expr_FuncCall(
            name: Name(
                parts: array(
                    0: var_dump
                )
            )
            args: array(
                0: Arg(
                    name: null
                    value: Expr_Variable(
                        name: c
                    )
                    byRef: false
                    unpack: false
                )
            )
        )
    )
)
```

这个抽象语法树还是比较简单的。我们大致可以看到，有`4`条表达式语句。我们逐个来看。

第一条表达式语句：

```bash
0: Stmt_Expression(
    expr: Expr_Assign(
        var: Expr_Variable(
            name: a
        )
        expr: Expr_BinaryOp_Minus(
            left: Expr_BinaryOp_Plus(
                left: Expr_BinaryOp_Plus(
                    left: Scalar_LNumber(
                        value: 1
                    )
                    right: Scalar_LNumber(
                        value: 2
                    )
                )
                right: Scalar_LNumber(
                    value: 3
                )
            )
            right: Scalar_LNumber(
                value: 4
            )
        )
    )
)
```

这是一个赋值语句，我们通过这个结构得出，这个`AST`是右结合的。也就意味着，我们的赋值语句是右结合的。`Expr_Assign`它的左子树是一个变量的名字，`Expr_Assign`它的右子树是一个算数表达式。通过这个算数表达式的`AST`结构，我们可以看成，它是左结合的。

`OK`，我们可以根据这些节点的类型，写出对应的`visitor`。

首先是`AdditiveExpressionVisitor`：

```php
<?php

declare(strict_types=1);
/**
 * This file is part of Yaphp.
 *
 * @contact  codinghuang@qq.com
 */
namespace Yaphp\NodeVisitor;

use PhpParser\Node;
use PhpParser\Node\Expr;
use PhpParser\Node\Expr\BinaryOp;
use PhpParser\Node\Expr\BinaryOp\Minus;
use PhpParser\Node\Expr\BinaryOp\Plus;
use PhpParser\Node\Expr\Variable;
use PhpParser\Node\Scalar\LNumber;
use PhpParser\NodeVisitorAbstract;
use Yaphp\HandWritten\CompilerGlobals;

class AdditiveExpressionVisitor extends NodeVisitorAbstract
{
    public function enterNode(Node $node)
    {
        if (! ($node instanceof Plus) && ! ($node instanceof Minus)) {
            return;
        }
        switch (get_class($node)) {
            case Plus::class:
            case Minus::class:
                $result = $this->additiveExpression($node);
                $resultNode = new Node\Scalar\LNumber($result);
                break;
            default:
                break;
        }

        return $resultNode;
    }

    protected function additiveExpression(Expr $expr): int
    {
        if (isset($expr->left)) {
            $leftValue = $this->additiveExpression($expr->left);
        }
        if (isset($expr->right)) {
            $rightValue = $this->additiveExpression($expr->right);
        }

        switch (get_class($expr)) {
            case Plus::class:
                return $leftValue + $rightValue;
                break;
            case Minus::class:
                return $leftValue - $rightValue;
                break;
            case Variable::class:
                return CompilerGlobals::getSymbol($expr->name);
                break;
            case LNumber::class:
                return $expr->value;
                break;
            default:
                echo sprintf("Don't support expression %s", $expr->getType());
                exit;
                break;
        }

        return $leftValue + $rightValue;
    }
}
```

因为，加法和减法我们认为是同一类操作，所以，我们可以把加法和减法写在一个`visitor`里面。

接着就是`AssignVistor`：

```php
<?php

declare(strict_types=1);
/**
 * This file is part of Yaphp.
 *
 * @contact  codinghuang@qq.com
 */
namespace Yaphp\NodeVisitor;

use PhpParser\Node;
use PhpParser\Node\Expr\Assign;
use PhpParser\Node\Expr\BinaryOp;
use PhpParser\Node\Scalar\LNumber;
use PhpParser\NodeVisitorAbstract;
use Yaphp\HandWritten\CompilerGlobals;

class AssignVistor extends NodeVisitorAbstract
{
    public function leaveNode(Node $node)
    {
        $value = -INF;
        if (! ($node instanceof Assign)) {
            return;
        }
        if ($node->expr instanceof LNumber) {
            $value = $node->expr->value;
        } else if ($node->expr instanceof BinaryOp) {
            $return = (new AdditiveExpressionVisitor)->enterNode($node->expr);
            $value = $return->value;
        }
        CompilerGlobals::setSymbol($node->var->name, $value);
    }
}
```

可以看到，实现变量就是一个字典，把变量的名字和对应的值存起来即可。目前我们这篇文章不考虑变量的作用域，所以我们把所有的变量通通存在了一个全局变量里面。

第二条表达式语句：

```php
1: Stmt_Expression(
    expr: Expr_Assign(
        var: Expr_Variable(
            name: b
        )
        expr: Expr_BinaryOp_Mul(
            left: Scalar_LNumber(
                value: 2
            )
            right: Scalar_LNumber(
                value: 3
            )
        )
    )
)
```

可以看到，这个结构和加法的算数表达式几乎是一样的。但是，我们认为加法和乘法还是有一定的区别的，所以我们单独给乘法写一个`visitor`：

```php
<?php

declare(strict_types=1);
/**
 * This file is part of Yaphp.
 *
 * @contact  codinghuang@qq.com
 */
namespace Yaphp\NodeVisitor;

use PhpParser\Node;
use PhpParser\Node\Expr;
use PhpParser\Node\Expr\BinaryOp\Div;
use PhpParser\Node\Expr\BinaryOp\Mul;
use PhpParser\NodeVisitorAbstract;

class MultiplicativeExpressionVisitor extends NodeVisitorAbstract
{
    public function enterNode(Node $node)
    {
        if (! ($node instanceof Mul) && ! ($node instanceof Div)) {
            return;
        }
        switch (get_class($node)) {
            case Mul::class:
            case Div::class:
                $result = $this->multiplicativeExpression($node);
                $resultNode = new Node\Scalar\LNumber($result);
                break;
            default:
                break;
        }

        return $resultNode;
    }

    protected function multiplicativeExpression(Expr $expr): int
    {
        if (isset($expr->left)) {
            $leftValue = $this->multiplicativeExpression($expr->left);
        }
        if (isset($expr->right)) {
            $rightValue = $this->multiplicativeExpression($expr->right);
        }

        switch (get_class($expr)) {
            case Mul::class:
                return $leftValue * $rightValue;
                break;
            case Div::class:
                return $leftValue / $rightValue;
                break;
            default:
                return $expr->value;
                break;
        }

        return $leftValue * $rightValue;
    }
}
```

第三条表达式语句：

```php
2: Stmt_Expression(
    expr: Expr_Assign(
        var: Expr_Variable(
            name: c
        )
        expr: Expr_BinaryOp_Plus(
            left: Expr_Variable(
                name: a
            )
            right: Expr_Variable(
                name: b
            )
        )
    )
)
```

这是两个变量相加，然后把表达式的结果赋值给一个新的变量。因为，我们把两个变量相加，也放在了`AdditiveExpressionVisitor`，所以，这里我们无须再实现一个新的`visitor`了。

我们来看最后一个表达式语句：

```php
3: Stmt_Expression(
    expr: Expr_FuncCall(
        name: Name(
            parts: array(
                0: var_dump
            )
        )
        args: array(
            0: Arg(
                name: null
                value: Expr_Variable(
                    name: c
                )
                byRef: false
                unpack: false
            )
        )
    )
)
```

这是一个函数调用了，所以，我们需要实现一个新的`FuncCallExpressionVisitor`。因为，函数也是可以求值的，所以我们把函数调用`visitor`归类为`expression`：

```php
<?php

declare(strict_types=1);
/**
 * This file is part of Yaphp.
 *
 * @contact  codinghuang@qq.com
 */
namespace Yaphp\NodeVisitor;

use PhpParser\Node;
use PhpParser\Node\Expr\FuncCall;
use PhpParser\NodeVisitorAbstract;
use Yaphp\HandWritten\CompilerGlobals;

class FuncCallExpressionVisitor extends NodeVisitorAbstract
{
    public function leaveNode(Node $node)
    {
        if (! ($node instanceof FuncCall)) {
            return;
        }

        $functionName = $node->name->parts[0];
        $symbol = $node->args[0]->value->name;

        if (! CompilerGlobals::hasSymbol($symbol)) {
            throw new \Exception(sprintf('not define symbol %s', $symbol), 1);
        }

        if ($functionName === 'var_dump') {
            $this->varDumpHandler(CompilerGlobals::getSymbol($symbol));
        }
    }

    protected function varDumpHandler($symbol)
    {
        var_dump($symbol);
    }
}
```

这里，我们实现`var_dump`的方式就非常的简单了。当我们发现，这是一个`var_dump`函数的时候，我们直接调用`var_dump`函数即可。

至此，我们就算是实现了我们所有的`visitor`了。只要我们对`AST`使用上我们的`visitor`，我们就可以很愉快的去解析它了。
