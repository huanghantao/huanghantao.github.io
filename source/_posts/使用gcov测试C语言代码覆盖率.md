---
title: 使用gcov测试C语言代码覆盖率
date: 2020-06-29 20:09:08
tags:
- C语言
---

最近在给`Swoole`的内核代码做覆盖率测试，我们测试的是`Swoole`的`core-tests`对`libswoole`的代码覆盖率，这个过程中遇到了一些问题，所以总结下。

## 基本流程

我通过一个简单的例子来进行讲解。

首先，定义一些函数（可以理解为`libswoole`）：

```cpp
// lib.c
void func1(int a) {
    if (a > 0) {
        a--;
    } else {
        a++;
    }
}

void func2() {
    int b = 0;
    b++;
}
```

然后在`main`函数去调用这些函数（可以理解为`core-tests`）：

```cpp
// gcov.c
extern void func1(int a);
extern void func2();

int main(int argc, char const *argv[])
{
    func1(1);
    func2();

    return 0;
}
```

然后我们开始测试覆盖率。首先是编译我们需要测试的库，也就是`lib.c`这个文件：

```bash
[root@a896c4eb1fc4 gcov]# ls
gcov.c  lib.c

[root@a896c4eb1fc4 gcov]# gcc --coverage -c lib.c
[root@a896c4eb1fc4 gcov]# ls
gcov.c  lib.c  lib.gcno  lib.o
```

我们发现，如果我们在编译文件的时候加上了`--coverage`，那么就会为这个文件产生一个对应的`.gcno`文件。

接着编译出可执行文件：

```bash
[root@a896c4eb1fc4 gcov]# gcc lib.o gcov.c -lgcov
[root@a896c4eb1fc4 gcov]# ls
a.out  gcov.c  lib.c  lib.gcno  lib.o
[root@a896c4eb1fc4 gcov]#
```

现在，我们执行这个可执行文件：

```bash
[root@a896c4eb1fc4 gcov]# ./a.out
[root@a896c4eb1fc4 gcov]# ls
a.out  gcov.c  lib.c  lib.gcda  lib.gcno  lib.o
[root@a896c4eb1fc4 gcov]#
```

我们发现，当执行完可执行文件之后，会为我们代测试的文件产生一个`.gcda`文件（我们需要记住的一点就是，一定要生成了`.gcda`文件之后，才可以看到覆盖率）。

然后，我们就可以测试`lib.c`的覆盖率了：

```bash
[root@a896c4eb1fc4 gcov]# gcov lib.c
File 'lib.c'
Lines executed:88.89% of 9
Creating 'lib.c.gcov'

[root@a896c4eb1fc4 gcov]#
```

可以看出，`lib.c`的测试覆盖率是`88.89%`。

但是，光看这一点信息是看不出到底是没有覆盖到哪些代码。

此时，我们注意到，多了一个`.gcov`文件：

```bash
[root@a896c4eb1fc4 gcov]# ls
a.out  gcov.c  lib.c  lib.c.gcov  lib.gcda  lib.gcno  lib.o
[root@a896c4eb1fc4 gcov]#
```

我们可以看一下这个文件内容：

```bash
[root@a896c4eb1fc4 gcov]# cat lib.c.gcov
        -:    0:Source:lib.c
        -:    0:Graph:lib.gcno
        -:    0:Data:lib.gcda
        -:    0:Runs:1
        -:    0:Programs:1
        1:    1:void func1(int a) {
        1:    2:    if (a > 0) {
        1:    3:        a--;
        -:    4:    } else {
    #####:    5:        a++;
        -:    6:    }
        1:    7:}
        -:    8:
        1:    9:void func2() {
        1:   10:    int b = 0;
        1:   11:    b++;
        1:   12:}
[root@a896c4eb1fc4 gcov]#
```

其中，标记为`1`的代表覆盖到了，标记为`#####`代表没有覆盖到。

所以，我们需要修改一下我们的测试代码，来覆盖到这一行：

```cpp
extern void func1(int a);
extern void func2();

int main(int argc, char const *argv[])
{
    func1(1);
    func1(-1);
    func2();

    return 0;
}
```

然后重新编译可执行文件：

```bash
[root@a896c4eb1fc4 gcov]# gcc lib.o gcov.c -lgcov
```

然后重新执行可执行文件：

```bash
[root@a896c4eb1fc4 gcov]# ./a.out
```

然后重新测试覆盖率：

```bash
[root@a896c4eb1fc4 gcov]# gcov lib.c
File 'lib.c'
Lines executed:100.00% of 9
Creating 'lib.c.gcov'

[root@a896c4eb1fc4 gcov]#
```

可以发现，测试覆盖率达到了`100.00%`。

## 通过lcov可视化结果

除了用`gcov`来查看覆盖率之外，我们还可以用`lcov`来生成`html`页面来看覆盖率：

```bash
[root@a896c4eb1fc4 gcov]# lcov --directory . --capture --output-file coverage.info
Found gcov version: 4.8.5
Scanning . for .gcda files ...
Found 1 data files in .
Processing lib.gcda
Finished .info-file creation

[root@a896c4eb1fc4 gcov]# ls
a.out  coverage.info  gcov.c  lib.c  lib.gcda  lib.gcno  lib.o
[root@a896c4eb1fc4 gcov]#
```

此时生成了文件`coverage.info`。

其中，`--directory`用来指定`.gcda`文件的目录。现在我们的`.gcda`是在当前目录。假设我们不知道`.gcda`文件的路径，我们可以通过如下方法来查看：

```bash
[root@a896c4eb1fc4 gcov]# strings a.out | grep gcda
/root/codeDir/cCode/test/gcov/lib.gcda
[root@a896c4eb1fc4 gcov]#
```

然后我们可以通过如下命令来看覆盖率：

```bash
[root@a896c4eb1fc4 gcov]# lcov --list coverage.info
Reading tracefile coverage.info
            |Lines       |Functions  |Branches
Filename    |Rate     Num|Rate    Num|Rate     Num
==================================================
[/root/codeDir/cCode/test/gcov/]
lib.c       | 100%      9| 100%     2|    -      0
==================================================
      Total:| 100%      9| 100%     2|    -      0
[root@a896c4eb1fc4 gcov]#
```

然后，我们可以去生成`html`文件：

```bash
[root@a896c4eb1fc4 gcov]# genhtml -o report_dir coverage.info
Reading data file coverage.info
Found 1 entries.
Found common filename prefix "/root/codeDir/cCode/test"
Writing .css and .png files.
Generating output.
Processing file gcov/lib.c
Writing directory view page.
Overall coverage rate:
  lines......: 100.0% (9 of 9 lines)
  functions..: 100.0% (2 of 2 functions)

[root@a896c4eb1fc4 gcov]# ls report_dir/
amber.png  emerald.png  gcov  gcov.css  glass.png  index-sort-f.html  index-sort-l.html  index.html  ruby.png  snow.png  updown.png
[root@a896c4eb1fc4 gcov]#
```

（我们用浏览器打开`index.html`文件就可以看到覆盖率信息了）

## gcov实现原理

`gcc`编译的时候，如果加上了`--coverage`覆盖率测试选项后，`gcc`会作如下处理：

1. 在输出目标文件中留出一段存储区保存统计数据
2. 在源代码中每行可执行语句生成的代码之后附加一段更新覆盖率统计结果的代码。（若用户进程并非调用`exit`正常退出，覆盖率统计数据就无法输出，也就无从生成报告了）
3. 在可执行文件进入`main`函数之前调用`gcov_init`内部函数初始化统计数据区，并将`gcov_exit`内部函数注册为`exit handlers`
4. 可执行文件调用`exit`正常结束时，`gcov_exit`函数得到调用，其继续调用`__gcov_flush`函数输出统计数据到`*.gcda`文件中

服务器程序一般启动后就很少主动退出，用`kill`杀死进程强制退出时就不会调用`exit`，因此没有覆盖率统计结果产生。为了解决这个问题，我们可以给待测程序增加一个`signal handler`，拦截`SIGHUP`、`SIGINT`、`SIGQUIT`、`SIGTERM`等常见强制退出信号，并在`signal handler`中主动调用`exit`或`__gcov_flush`函数输出统计结果即可。

这种方案会修改我们的待测程序，所以，我们可以通过动态库预加载技术和`gcc`扩展的`constructor`属性，然后将`signalhandler`和其注册过程都封装到一个独立的动态库中，并在预加载动态库时实现信号拦截注册。

我们来举个例子，修改一下我们的测试库：

```cpp
// lib.c
void func1(int a) {
    if (a > 0) {
        a--;
    } else {
        a++;
    }
}

void func2() {
    int b = 0;
    b++;
    sleep(-1);
}
```

然后重复上面的过程：

```bash
[root@a896c4eb1fc4 gcov]# yes | rm -r a.out \
coverage.info \
lib.gcda \
lib.gcno \
lib.o \
report_dir
[root@a896c4eb1fc4 gcov]# gcc --coverage -c lib.c
[root@a896c4eb1fc4 gcov]# gcc lib.o gcov.c -lgcov
[root@a896c4eb1fc4 gcov]# ./a.out

```

我们会发现，我们的程序阻塞了，不会退出。此时，我们按`CTRL + C`来退出进程：

```bash
[root@a896c4eb1fc4 gcov]# ./a.out
^C
[root@a896c4eb1fc4 gcov]# ls
a.out  gcov.c  lib.c  lib.gcno  lib.o
[root@a896c4eb1fc4 gcov]#
```

我们发现，不会生成`.gcda`文件，所以我们需要去捕获退出信号。我们来编写一下我们的预加载动态库：

```cpp
// preload.c
#include <stdlib.h>
#include <signal.h>

extern void __gcov_flush();

void sighandler(int signo)
{
    __gcov_flush();
}

__attribute__ ((constructor)) void ctor()
{
    int sigs[] = { SIGILL, SIGFPE, SIGABRT, SIGBUS, SIGSEGV, SIGHUP, SIGINT, SIGQUIT, SIGTERM };
    int i;
    struct sigaction sa;

    sa.sa_handler = sighandler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESETHAND;
    for(i = 0; i < sizeof(sigs) / sizeof(sigs[0]); ++i) {
        sigaction(sigs[i], &sa, NULL);
    }
}
```

然后，我们编译出动态库：

```bash
[root@a896c4eb1fc4 gcov]# gcc -shared -fPIC preload.c -o libpreload.so -lgcov
[root@a896c4eb1fc4 gcov]# ls
a.out  gcov.c  lib.c  lib.gcno  lib.o  preload.c  libpreload.so
[root@a896c4eb1fc4 gcov]#
```

此时，我们在编译可执行文件的时候，链接一下这个库：

```bash
[root@a896c4eb1fc4 gcov]# gcc lib.o gcov.c -L. -lgcov -lpreload
```

执行可执行文件后终止它：

```bash
[root@a896c4eb1fc4 gcov]# ./a.out
^C
[root@a896c4eb1fc4 gcov]# ls
a.out  gcov.c  lib.c  lib.gcda  lib.gcno  lib.o  libpreload.so  preload.c
[root@a896c4eb1fc4 gcov]#
```

可以发现，会生成`.gcda`文件。我们现在就可以来查看覆盖率了：

```bash
[root@a896c4eb1fc4 gcov]# gcov lib.c
File 'lib.c'
Lines executed:100.00% of 10
Creating 'lib.c.gcov'

[root@a896c4eb1fc4 gcov]#
```
