---
title: 现代化PHP-生产环境下优化Composer加载的原理
date: 2019-08-23 16:30:06
tags:
- PHP
- Composer
---

# Composer自动加载类型

```shell
1、files
2、classmap
3、psr-0
4、psr-4
```

建议：项目代码用`psr-4`自动加载，`helper`用`files`自动加载，生产环境用`classmap`自动加载。`psr-0`已经被抛弃了，历史遗留代码有部分使用。

我们现在来测试一下，我们先创建一个测试根目录`autoloads`，然后进入：

```shell
~/codeDir/phpCode # mkdir autoloads ; cd autoloads
~/codeDir/phpCode/autoloads # 
```

## files

我们创建目录`libraries`，然后进入：

```shell
~/codeDir/phpCode/autoloads # mkdir libraries ; cd libraries
~/codeDir/phpCode/autoloads/libraries # 
```

然后创建文件`functions.php`：

```shell
~/codeDir/phpCode/autoloads/libraries # touch functions.php
~/codeDir/phpCode/autoloads/libraries # 
```

内容如下：

```shell
<?php

function func1 () {
    return 'in func1.';
}
```

然后，我们在目录`autoloads`下面创建一个`composer.json`文件：

```shell
~/codeDir/phpCode/autoloads/libraries # cd .. ; touch composer.json
```

内容如下：

```json
{
    "autoload": {
        "files": ["libraries/functions.php"]
    }
}
```

这里，`autoload`我们填写的是`files`，说明我们打算基于`files`来完成自动加载。

然后执行命令`composer dump`：

```shell
~/codeDir/phpCode/autoloads # composer dump
Do not run Composer as root/super user! See https://getcomposer.org/root for details
Generated autoload files containing 0 classes
~/codeDir/phpCode/autoloads # 
```

我们会发现多了一个文件夹`vendor`，目录结构如下：

```shell
~/codeDir/phpCode/autoloads # tree -L 3
.
├── composer.json
├── libraries
│   └── functions.php
└── vendor
    ├── autoload.php
    └── composer
        ├── ClassLoader.php
        ├── LICENSE
        ├── autoload_classmap.php
        ├── autoload_files.php
        ├── autoload_namespaces.php
        ├── autoload_psr4.php
        ├── autoload_real.php
        └── autoload_static.php

3 directories, 11 files
~/codeDir/phpCode/autoloads # 
```

我们会发现，`composer`文件夹里面的这几个`autoload_*.php`文件刚好和我们介绍的自动加载类型对应。并且，只有`autoload_files.php`文件里面有自动加载的有效信息：

```php
return array(
    '4f84e339e19580763acd6b29b090e23c' => $baseDir . '/libraries/functions.php',
);
```

然后，我们创建一个测试文件`test.php`来测试是否可以找到文件`functions.php`中的`func1`函数：

```shell
~/codeDir/phpCode/autoloads # touch test.php
~/codeDir/phpCode/autoloads # cat > test.php <<EOF
<?php

require 'vendor/autoload.php';

echo func1();
EOF
~/codeDir/phpCode/autoloads # 
```

为了方便大家复制粘贴，我直接通过`heredoc`语法来编辑文件，大家直接在命令行里面粘贴进去即可。

然后，我们执行脚本：

```shell
~/codeDir/phpCode/autoloads # php test.php 
in func1.~/codeDir/phpCode/autoloads # 
```

成功调用了函数`func1`。

这是基于`files`的自动加载。

## classmap

现在，我们通过`classmap`来完成自动加载。这比`files`类型的加载好，因为我们不必指明具体的文件，只需指明文件所在的目录即可。所以相对于`files`类型的加载又更加的灵活一点了。

我们创建目录`classmap`：

```shell
~/codeDir/phpCode/autoloads # mkdir classmap ; cd classmap
```

然后创建文件`functions.php`：

```shell
~/codeDir/phpCode/autoloads/classmap # touch functions.php
~/codeDir/phpCode/autoloads/classmap # 
```

编辑文件内容：

```shell
~/codeDir/phpCode/autoloads/classmap # cat > functions.php << EOF
<?php
Class Test {
    public function func1 () {
        return 'in Test->func1';
    }
}
EOF
~/codeDir/phpCode/autoloads/classmap # 
```

然后修改`composer.json`文件：

```shell
~/codeDir/phpCode/autoloads/classmap # cd ..
~/codeDir/phpCode/autoloads # cat > composer.json << EOF
{
    "autoload": {
        "files": ["libraries/functions.php"],
        "classmap": ["classmap"]
    }
}
EOF
```



然后，我们编写测试文件：

```shell
~/codeDir/phpCode/autoloads # cat > test.php << EOF
<?php

require 'vendor/autoload.php';

echo func1();

\$t = new Test();
echo \$t->func1();
EOF
~/codeDir/phpCode/autoloads # 
```

然后执行命令：

```shell
~/codeDir/phpCode/autoloads # composer dump
Do not run Composer as root/super user! See https://getcomposer.org/root for details
Generated autoload files containing 1 classes
```

现在，除了`vendor/composer/autoload_files.php`里面有有效的内容外，`vendor/composer/autoload_classmap.php`里面也有了：

```php
return array(
    'Test' => $baseDir . '/classmap/functions.php',
);
```

接着测试脚本：

```shell
~/codeDir/phpCode/autoloads # php test.php 
in func1.in Test->func1~/codeDir/phpCode/autoloads # 
```

我们发现成功的调用了`Test->func1`。

## psr-0

`psr-0`用的就比较少了，这里不演示了。

## psr-4

我们创建`src`目录：

```shell
~/codeDir/phpCode/autoloads # mkdir src ; cd src
~/codeDir/phpCode/autoloads/src # 
```

然后创建文件`Test.php`：

```shell
~/codeDir/phpCode/autoloads/src # touch Test.php
~/codeDir/phpCode/autoloads/src # 
```

文件内容如下：

```shell
~/codeDir/phpCode/autoloads/src # cat > Test.php << EOF
<?php

namespace App;

Class Test {
    public function func1 () {
        return 'in Test->func1';
    }
}
EOF
```

然后修改`composer.json`文件：

```shell
~/codeDir/phpCode/autoloads/src # cd ..
~/codeDir/phpCode/autoloads # cat > composer.json << EOF
{
    "autoload": {
        "files": ["libraries/functions.php"],
        "classmap": ["classmap"],
        "psr-4": {
            "App\\": "src"
        }
    }
}
EOF
```

注意`App`后面不要漏了`\\`，否则会报错：

```shell
A non-empty PSR-4 prefix must end with a namespace separator.
```

翻译过来就是：

```
非空PSR-4前缀必须以命名空间分隔符结尾。
```

编写测试文件：

```shell
~/codeDir/phpCode/autoloads # cat > test.php << EOF
<?php

require 'vendor/autoload.php';

echo func1();

\$t = new Test();
echo \$t->func1();

\$t1 = new App\Test();
echo \$t1->func1();
EOF
```

然后执行命令：

```shell
~/codeDir/phpCode/autoloads # composer dump
Do not run Composer as root/super user! See https://getcomposer.org/root for details
Generated autoload files containing 1 classes
~/codeDir/phpCode/autoloads # 
```

此时，文件`autoload_psr4.php`里面有了有效内容：

```php
return array(
    'App\\' => array($baseDir . '/src'),
);
```

执行脚本：

```shell
~/codeDir/phpCode/autoloads # php test.php 
in func1.in Test->func1in Test->func1~/codeDir/phpCode/autoloads # 
```

调用成功。

# 全部使用classmap自动加载

执行命令：

```shell
~/codeDir/phpCode/autoloads # composer dumpautoload -o
Do not run Composer as root/super user! See https://getcomposer.org/root for details
Generated optimized autoload files containing 2 classes
~/codeDir/phpCode/autoloads # 
```

我们会发现，在文件`autoload_classmap.php`里面，内容更新为了：

```php
return array(
    'App\\Test' => $baseDir . '/src/Test.php',
    'Test' => $baseDir . '/classmap/functions.php',
);
```

多了`'App\\Test' => $baseDir . '/src/Test.php'`。它把以`psr-4`方式加载的类也变成了以`classmap`方式加载了。

我们看看`-o`参数的解释：

```shell
~/codeDir/phpCode/autoloads # composer dumpautoload --help
Options:
  -o, --optimize                 Optimizes PSR0 and PSR4 packages to be loaded with classmaps too, good for production.
```

也就是说，会自动把我们配置好的`psr-0`以及`psr-4`加载方式转化为`classmap`的方式加载。

如果我们要取消这种优化，那么我们可以执行命令：

```shell
~/codeDir/phpCode/autoloads # composer dumpautoload 
Do not run Composer as root/super user! See https://getcomposer.org/root for details
Generated autoload files containing 1 classes
~/codeDir/phpCode/autoloads # 
```

此时，文件`autoload_classmap.php`里面的内容变成了原来的：

```php
return array(
    'Test' => $baseDir . '/classmap/functions.php',
);
```

阅读`vendor/composer/ClassLoader.php`：

```php
public function findFile($class)
{
  // class map lookup
  if (isset($this->classMap[$class])) {
    return $this->classMap[$class];
  }
  
  // 省略了其他的代码

  $file = $this->findFileWithExtension($class, '.php');

  // 省略了其他的代码
}

private function findFileWithExtension($class, $ext)
{
  // PSR-4 lookup
  $logicalPathPsr4 = strtr($class, '\\', DIRECTORY_SEPARATOR) . $ext;

  //省略了其他的代码
  
  // PSR-0 lookup
  if (false !== $pos = strrpos($class, '\\')) {
    // namespaced class name
    $logicalPathPsr0 = substr($logicalPathPsr4, 0, $pos + 1)
      . strtr(substr($logicalPathPsr4, $pos + 1), '_', DIRECTORY_SEPARATOR);
  } else {
    // PEAR-like class name
    $logicalPathPsr0 = strtr($class, '_', DIRECTORY_SEPARATOR) . $ext;
  }
  
  // 省略了其他的代码
}
```

我们会发现，`findFile`函数会先去`classmap`里面查找类的文件路径然后加载类（即`include`类），如果没找到，再通过`psr-4`的方式加载类，如果`psr-4`方式没找到，再通过`psr-0`的方式加载。

因为`classmap`的方式是`key - value`，可以直接找到类文件的位置，而不需要向`psr-4`那样需要用到一些拼接的操作，所以通过`classmap`的方式找类文件会快一点。

如果，我们的类都是通过`classmap`的方式加载的，并且在`classmap`里面找不到类的时候，不再通过其他加载方式查找类（即隐含的认为`classmap`中就是所有合法的类，这样当类不存在的时候，就可以避免没必要的查找了），那么，我们可以通过如下的命令来优化：

```shell
~/codeDir/phpCode/autoloads # composer dumpautoload -a
Do not run Composer as root/super user! See https://getcomposer.org/root for details
Generated optimized autoload files (authoritative) containing 2 classes
~/codeDir/phpCode/autoloads # 
```

然后，我们看看`findFile`的代码：

```php
    public function findFile($class)
    {
        // class map lookup
        if (isset($this->classMap[$class])) {
            return $this->classMap[$class];
        }
        if ($this->classMapAuthoritative || isset($this->missingClasses[$class])) {
            return false;
        }
      // 省略其他的代码
```

此时`$this->classMapAuthoritative`的值会变成`true`。所以，一旦：

```php
if (isset($this->classMap[$class])) {
  return $this->classMap[$class];
}
```

这个`classmap`里面没有我们的类，那么就会直接返回`false`。这样就避免了没必要的查找了。

但是**不推荐在生产环境下使用`-a`的优化**，因为我们无法确保`classmap`里面真的就是包含了所有我们需要的类。我们**推荐使用`-o`进行优化**。