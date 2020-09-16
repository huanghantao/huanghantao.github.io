---
title: fsw协程库
date: 2019-09-02 17:09:30
tags:
- 协程
- C++
---

为了让我创作出来的`PHP`协程扩展教程更加的好，所以我把扩展里`src`的代码抽离了出来，并且可以编译出一个动态链接库，[仓库地址](https://github.com/huanghantao/fsw)。

这里简单的演示一下，同学们可以下载下来玩一玩。

首先，我们需要把`fsw`编译为动态链接库：

```shell
~/codeDir/cppCode/fsw # cmake .
~/codeDir/cppCode/fsw # make
~/codeDir/cppCode/fsw # make install
```

然后，我们先写一个创建协程的代码：

```cpp
#include <iostream>
#include "fsw/coroutine.h"

using namespace Fsw;
using namespace std;

int main(int argc, char const *argv[])
{
    Coroutine::create([](void *arg)
    {
        cout << "co1" << endl;
    });

    Coroutine::create([](void *arg)
    {
        cout << "co2" << endl;
    });

    return 0;
}
```

然后进行编译、运行：

```shell
~/codeDir/cppCode/fsw/examples # g++ create.cc -lfsw
~/codeDir/cppCode/fsw/examples # ./a.out 
co1
co2
~/codeDir/cppCode/fsw/examples # 
```

然后，我们再写一段协程`sleep`的代码：

```cpp
#include <iostream>
#include "fsw/coroutine.h"

using namespace Fsw;
using namespace std;

int main(int argc, char const *argv[])
{
    Coroutine::create([](void *arg)
    {
        cout << "co1" << endl;
        Coroutine::sleep(0.3);
        cout << "co1" << endl;
    });

    Coroutine::create([](void *arg)
    {
        cout << "co2" << endl;
        Coroutine::sleep(1);
        cout << "co2" << endl;
    });

    Coroutine::scheduler();
    
    return 0;
}
```

然后进行编译、运行：

```shell
~/codeDir/cppCode/fsw/examples # g++ sleep.cc -lfsw
~/codeDir/cppCode/fsw/examples # ./a.out 
co1
co2
co1
co2
~/codeDir/cppCode/fsw/examples # 
```

