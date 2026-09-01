---
title: 使用Swoole作为动态连接库编写代码
date: 2019-07-25 18:53:38
tags:
- PHP
- Swoole
---

这篇文章是讲解如何使用`Swoole`作为动态链接库，引入到我们自己的项目中。

编译`Swoole`为动态链接库的过程很简单，过程如下。

先下载一份`Swoole`源码，然后进入`Swoole`源码的根目录，然后开始进行编译过程。

```shell
~/codeDir/cppCode/swoole-src # phpize 
Configuring for:
PHP Api Version:         20180731
Zend Module Api No:      20180731
Zend Extension Api No:   320180731
~/codeDir/cppCode/swoole-src # 
```

```shell
~/codeDir/cppCode/swoole-src # ./configure
checking for grep that handles long lines and -e... /bin/grep
checking for egrep... /bin/grep -E
checking for a sed that does not truncate output... /bin/sed
checking for cc... cc
checking whether the C compiler works... yes
checking for C compiler default output file name... a.out

# 省略其他的输出内容

(cached) (cached) checking how to hardcode library paths into programs... immediate
configure: creating ./config.status
config.status: creating config.h
~/codeDir/cppCode/swoole-src # 
```

```shell
~/codeDir/cppCode/swoole-src # cmake .
-- The C compiler identification is GNU 8.3.0
-- The CXX compiler identification is GNU 8.3.0
-- Check for working C compiler: /usr/bin/cc
-- Check for working C compiler: /usr/bin/cc -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Check for working CXX compiler: /usr/bin/c++
-- Check for working CXX compiler: /usr/bin/c++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- The ASM compiler identification is GNU
-- Found assembler: /usr/bin/cc
-- Configuring done
-- Generating done
-- Build files have been written to: /root/codeDir/cppCode/swoole-src
~/codeDir/cppCode/swoole-src # 
```

然后安装`Swoole`为动态链接库：

```shell
~/codeDir/cppCode/swoole-src # make install
Scanning dependencies of target shared
[  1%] Building C object CMakeFiles/shared.dir/src/core/array.c.o
[  2%] Building C object CMakeFiles/shared.dir/src/core/base.c.o
[  3%] Building C object CMakeFiles/shared.dir/src/core/channel.c.o

# 省略其他的输出内容

[ 91%] Building CXX object CMakeFiles/shared.dir/src/wrapper/timer.cc.o
[ 92%] Linking CXX shared library lib/libswoole.so
[ 97%] Built target shared
Scanning dependencies of target test_server
[ 98%] Building C object CMakeFiles/test_server.dir/examples/test_server.c.o
[100%] Linking C executable bin/test_server
[100%] Built target test_server
Install the project...
-- Install configuration: "Debug"
Are you run command using root user?
-- Installing: /usr/local/lib/libswoole.so.4.4.1
-- Up-to-date: /usr/local/lib/libswoole.so
-- Up-to-date: /usr/local/include/swoole/array.h
-- Up-to-date: /usr/local/include/swoole/asm_context.h
-- Up-to-date: /usr/local/include/swoole/async.h

# 省略其他的输出内容

-- Up-to-date: /usr/local/include/swoole/wrapper/base.hpp
-- Up-to-date: /usr/local/include/swoole/wrapper/client.hpp
-- Up-to-date: /usr/local/include/swoole/wrapper/server.hpp
-- Up-to-date: /usr/local/include/swoole/swoole_config.h
-- Installing: /usr/local/include/swoole/config.h
~/codeDir/cppCode/swoole-src # 
```

我们发现，我们编译出了一个动态链接库`libswoole.so`。现在，我们就可以使用`Swoole`实现的函数了并且里面有许多的数据结构可以被我们使用。

我们来进行测试：

```c++
#include <swoole/swoole.h>
#include <swoole/hashmap.h>

#include <iostream>

using namespace std;

int main(int argc, char const *argv[])
{
    char *ret;
    swHashMap *hm = swHashMap_new(16, NULL);
    swHashMap_add(hm, (char *) SW_STRL("key1"), (void *)"value1");
    swHashMap_add(hm, (char *) SW_STRL("key2"), (void *)"value2");
    swHashMap_add(hm, (char *) SW_STRL("key3"), (void *)"value3");
    swHashMap_add(hm, (char *) SW_STRL("key4"), (void *)"value4");
    swHashMap_add(hm, (char *) SW_STRL("key5"), (void *)"value5");

    ret = (char *)swHashMap_find(hm, (char *) SW_STRL("key1"));
    cout << ret << endl;

    ret = (char *)swHashMap_find(hm, (char *) SW_STRL("key2"));
    cout << ret << endl;

    ret = (char *)swHashMap_find(hm, (char *) SW_STRL("key3"));
    cout << ret << endl;

    ret = (char *)swHashMap_find(hm, (char *) SW_STRL("key4"));
    cout << ret << endl;

    ret = (char *)swHashMap_find(hm, (char *) SW_STRL("key5"));
    cout << ret << endl;

    swHashMap_update(hm, (char *) SW_STRL("key1"), (void *)"newvalue1");
    ret = (char *)swHashMap_find(hm, (char *) SW_STRL("key1"));
    cout << ret << endl;

    swHashMap_free(hm);
}
```

编译、执行：

```shell
~/codeDir/cppCode # g++ -D HAVE_CONFIG_H main.cpp -lswoole
~/codeDir/cppCode # ./a.out 
value1
value2
value3
value4
value5
newvalue1
```

成功。

有了这个基础，我们可以在我们的`PHP`协程扩展里面使用`Swoole`的基础数据结构，实现网络模块。

注意，我们这里不可以直接使用`Swoole`的定时器。因为`Swoole`的定时器用了全局变量`SwooleG`，所以我们需要自己去实现定时器。