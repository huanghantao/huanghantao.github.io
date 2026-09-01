---
title: 安装了Swoole扩展之后，查看opcode，出现许多swoole的opcode的解决方法
date: 2019-07-26 18:57:23
tags:
- PHP
- Swoole
---

如果你在安装了`Swoole`扩展之后，查看`PHP`脚本对应的`opcode`的时候，可能会发现出现很多的`opcode`。我举个例子吧。`PHP`脚本如下：

```php
<?php

function test()
{
    echo "test";
}

echo "main";

```

这是一个很简单的脚本，按理来说只会生成两组`opcode`。但是，如果我们查看它的`opcode`的话，就会有如下情况：

```shell
function name: (null)
L1-9 {main}() /root/codeDir/phpCode/test/swoole/test.php - 0x7f82eae77840 + 5 ops
 L3    #0     EXT_STMT                                                                              
 L3    #1     NOP                                                                                   
 L8    #2     EXT_STMT                                                                              
 L8    #3     ECHO                    "main"                                                        
 L9    #4     RETURN<-1>              1                                                             

# 省略了其他的opcode

function name: _array
L16-19 _array() @swoole-src/library/functions.php - 0x7f82eae6c480 + 8 ops
 L16   #0     RECV_INIT               1                    array(0)             $array              
 L18   #1     NEW<1>                  "Swoole\\ArrayObje"+                      @0                  
 L18   #2     SEND_VAR_EX             $array               1                                        
 L18   #3     DO_FCALL                                                                              
 L18   #4     VERIFY_RETURN_TYPE      @0                                                            
 L18   #5     RETURN                  @0                                                            
 L19   #6     VERIFY_RETURN_TYPE                                                                    
 L19   #7     RETURN<-1>              null                                                          

function name: scheduler
L24-31 scheduler() @swoole-src/library/functions.php - 0x7f82eae6c700 + 8 ops
 L26   #0     BIND_STATIC<1>          $scheduler           "scheduler"                              
 L27   #1     BOOL_NOT                $scheduler                                ~0                  
 L27   #2     JMPZ                    ~0                   J6                                       
 L28   #3     NEW                     "Swoole\\Coroutine"+                      @1                  
 L28   #4     DO_FCALL                                                                              
 L28   #5     ASSIGN                  $scheduler           @1                                       
 L30   #6     RETURN                  $scheduler                                                    
 L31   #7     RETURN<-1>              null                                                          

function name: test
L3-6 test() /root/codeDir/phpCode/test/swoole/test.php - 0x7f82eae77780 + 5 ops
 L3    #0     EXT_NOP                                                                               
 L5    #1     EXT_STMT                                                                              
 L5    #2     ECHO                    "test"                                                        
 L6    #3     EXT_STMT                                                                              
 L6    #4     RETURN<-1>              null                                                          

# 省略其他的opcode
```

我们发现，除了`main`和`test`的两组`opcode`，还生成了其他大量的`opcode`。

为什么呢？因为`Swoole4`引入了一个实现扩展的技巧：通过在`C/C++`扩展来直接执行`PHP`代码，从而实现扩展的部分功能。

核心函数是`php_swoole_library.h`：

```cpp
/**
 * 执行PHP代码：
 * 1、constants.php 定义常量SWOOLE_LIBRARY为true
 * 2、array.php 定义swoole_array_walk和swoole_array_walk_recursive函数
 * 3、exec.php 定义swoole_exec和swoole_shell_exec函数
 * 4、curl.php 定义了类swoole_http_status_code，里面包含了status code和对应的reason；定义了swoole_curl_handler类等等
 * 5、WaitGroup.php 定义了WaitGroup类，类似于golang的WaitGroup
 * 6、ObjectPool.php 定义了ObjectPool类，即对象池
 * 7、StringObject.php 定义了StringObject类，封装了对字符串常用的一些操作
 * 8、ArrayObject.php 定义了ArrayObject类，封装了对数组常用的一些操作
 * 9、Server.php 定义了一个协程化的Swoole\Coroutine\Server类，实际上是对Swoole\Coroutine\Socket的封装
 * 10、Connection.php 定义了一个Swoole\Coroutine\Server\Connection类
 * 11、functions.php 定义了一些函数
 * 12、alias.php 定义了Swoole\Coroutine\WaitGroup类别名Co\WaitGroup::class以及Swoole\Coroutine\Server::class类别名Co\Server::class
 * 13、alias_ns.php 定义了一些在非根命名空间下的函数，例如Swoole\Coroutine\run()以及Co\run()，实际上是对Scheduler类add和start的封装
 * 
 * 我们会发现，随着Swoole的壮大，可以用已经实现的Swoole功能去实现实现其他的功能，从而封装成一个库，提供给PHP用户空间使用。
 * 而且，这种实现方式对Swoole性能的影响还是比较小的，因为这些库在Swoole启动的时候就已经编译为了对应opcode。
 * 但是个人认为，这种实现方式比较浪费内存。例如一个int的PHP变量，需要消耗16字节，而C定义一个int变量一般只需要4字节，
 * 所以能够用C/C++实现的功能，尽可能的用C/C++实现。目前，Swoole主要是用这种方式实现内置库。
 */
static void php_swoole_load_library()
{
    zend::eval(swoole_library_source_constants, "@swoole-src/library/constants.php");
    zend::eval(swoole_library_source_std_array, "@swoole-src/library/std/array.php");
    zend::eval(swoole_library_source_std_exec, "@swoole-src/library/std/exec.php");
    zend::eval(swoole_library_source_ext_curl, "@swoole-src/library/ext/curl.php");
    zend::eval(swoole_library_source_core_coroutine_wait_group, "@swoole-src/library/core/Coroutine/WaitGroup.php");
    zend::eval(swoole_library_source_core_coroutine_object_pool, "@swoole-src/library/core/Coroutine/ObjectPool.php");
    zend::eval(swoole_library_source_core_string_object, "@swoole-src/library/core/StringObject.php");
    zend::eval(swoole_library_source_core_array_object, "@swoole-src/library/core/ArrayObject.php");
    zend::eval(swoole_library_source_core_coroutine_server, "@swoole-src/library/core/Coroutine/Server.php");
    zend::eval(swoole_library_source_core_coroutine_server_connection, "@swoole-src/library/core/Coroutine/Server/Connection.php");
    zend::eval(swoole_library_source_functions, "@swoole-src/library/functions.php");
    zend::eval(swoole_library_source_alias, "@swoole-src/library/alias.php");
    zend::eval(swoole_library_source_alias_ns, "@swoole-src/library/alias_ns.php");
}
```

我已经给出了这个函数的注释了。其中`zend::eval`直接执行`PHP`代码。通过这种方式，`Swoole`让`curl`协程化了。

所以，这就是为什么我们只是查看一个简单脚本，就会打印出众多的`opcode`。因为在加载`Swoole`扩展的时候，默认会把`swoole-src/library`里面的`PHP`代码编译为`opcode`。

所以，我们这个时候可以这样来查看我们这个脚本的`opcode`：

```shell
~/codeDir/phpCode/test/swoole # phpdbg -np* test.php
function name: (null)
L1-9 {main}() /root/codeDir/phpCode/test/swoole/test.php - 0x7f517c681000 + 3 ops
 L3    #0     NOP                                                                                   
 L8    #1     ECHO                    "main"                                                        
 L9    #2     RETURN<-1>              1                                                             

function name: test
L3-6 test() /root/codeDir/phpCode/test/swoole/test.php - 0x7f517c67e060 + 2 ops
 L5    #0     ECHO                    "test"                                                        
 L6    #1     RETURN<-1>              null                                                          
[Script ended normally]
~/codeDir/phpCode/test/swoole # 
```

多了一个`-n`参数，禁用默认的`php.ini`配置。