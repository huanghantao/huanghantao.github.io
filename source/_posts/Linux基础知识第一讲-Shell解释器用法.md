---
title: Linux基础知识第一讲-Shell解释器用法
date: 2019-08-23 10:44:36
tags:
- Linux
- Shell
---

# 作业(job) 控制

## 现代的shell解释器中存在作业控制功能

1、按`ctrl+ Z`将向当前进程发送`SIGSTOP`信号，如果当前进程没有实现`SIGSTOP`的特定逻辑，默认的行为是当前进程暂停并置入后台

2、使用命令`fg`来使恢复后台进程的前台运行

3、使用命令`bg`来使暂停的后台进程后台继续运行

4、使用命令`jobs`来显示后台作业

5、作业号不同于进程号

举个例子，比如我要查看`Docker`容器的输出日志：

```shell
hantaohuang@~ docker logs -f php
log1
log2
log3
^Z
[1]+  Stopped                 docker logs -f php
```

这个时候，我按下了`ctrl + z`。容器就会停止打印日志。

这里

```shell
[1]+  Stopped                 docker logs -f php
```

中的`1`就是作业号。如果我们想要恢复作业的执行，可以输入命令：

```shell
hantaohuang@~ bg %1
log4
log5
```

虽然还在打印日志，但是我们可以在这个时候执行命令，因为这个作业是跑在后台的（尽管它的输出是`STDOUT`）

```shell
hantaohuang@~ ls
Applications		Downloads		Music			Public			data			dockerfileDir		huanghantao.github.io
Desktop			Library			Pictures		cert			db			eclipse			tmp
Documents		Movies			Postman			codeDir			dev			eclipse-workspace	var
```

再举个例子，假设我在编辑一个文件：

```shell
hantaohuang@~ vim test.txt

~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
"test.txt" [New File]
```

这个时候，我会进入`vim`编辑器。此时，我们写`hello world`：

```shell
hello world
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
-- INSERT --
```

接着，我想写一下`open()`这个函数的用法，但是我忘了。这个时候，我们可以不需要另起一个终端，我们先按`esc`退出`vim`的编辑模式，然后按下`ctrl + z`：

```shell
hantaohuang@~ vim test.txt

[1]+  Stopped                 vim test.txt
hantaohuang@~ 
```

这个时候，我们再查看文档：

```shell
hantaohuang@~ man open
NAME
     open -- open files and directories
```

然后，我们再通过命令`fg`把作业拉回前台：

```shell
hantaohuang@~ fg %1
hello world
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~ 
```

然后写下`open()`函数的用法：

```shell
hello world
open -- open files and directories
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~                                                                                                                                                                                                           
~ 
-- INSERT --
```

然后再关闭文件。

（如果你是通过`bg`命令来恢复`vim`的话，是不可以编辑的，小伙伴们可以尝试一下）

# 通过分号来分隔多条命令

之前，我一直是通过`&&`来分隔多条命令的：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # echo 1 && echo 2
1
2
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

其实也可以通过`;`来分隔多条命令：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # echo 1 ; echo 2
1
2
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

# 用&后台执行条命令

我们上面举了一个在后台输出`Docker`容器日志的例子。其实，`ctrl + z`和`bg %1`可以用命令后面加`&`来完成。来测试一下：

```shell
hantaohuang@~ docker logs -f php &
log1
log2
log3
hantaohuang@~ ls
Applications		Downloads		Music			Public			data			dockerfileDir		huanghantao.github.io
Desktop			Library			Pictures		cert			db			eclipse			tmp
Documents		Movies			Postman			codeDir			dev			eclipse-workspace	var
log4
log5
```

实际上，通过命令加`&`也会生成一个作业号。当然，我们就可以去控制这个作业了。

# pipeline

```shell
|
>
<
>>
```

## |

`|`会把前一个命令的输出作为后一条命令的输入。

例如：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # echo hello | base64
aGVsbG8K
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

我们把`echo`命令的输出作为`base64`命令的输入，所以这里会把`hello`字符串进行`base64`编码。

如果我们需要把`aGVsbG8K`解码，可以做类似的操作：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # echo aGVsbG8K | base64 -d
hello
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

成功解码。

## >

把输出内容写入文件里面，举个例子：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # echo hello > test.txt
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

然后查看内容：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # cat test.txt 
hello
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

## >>

以追加模式打开文件内容，然后把输出内容追加到文件末尾，举个例子：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # echo world >> test.txt 
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

然后查看内容：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # cat test.txt 
hello
world
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

## <

可以把文件里面的内容作为左边命令的标准输入，举个例子：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # base64 < test.txt 
aGVsbG8Kd29ybGQK
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

我们解码一下：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # echo aGVsbG8Kd29ybGQK | base64 -d
hello
world
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

# Heredoc

在命令后使用`<<`来使用`heredoc`功能。

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # base64 <<EOF
> hello
> EOF
aGVsbG8K
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

我们发现`Heredoc`可以作为标准输入。

我们在`<<`的后面输入一个终结符`EOF`，但是终结符可以用其他任意的字符串，例如：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # base64 <<EO
> hello
> EO
aGVsbG8K
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

但是，开头和结尾必须要匹配：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # base64 <<EO
> hello
> Eo
> EO
aGVsbG8KRW8K
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

我们发现，大小写要一致。

那么，我们能否直接用`<`呢？小伙伴们发现，`<<`和`<`很类似对吧，我们来测试一下：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # base64 < hello
sh: can't open hello: no such file
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

我们发现不行，因为`<`的右边得是一个文件名字而不是字符串。

我们再举一个比较高级一点的例子，`cat`命令可以打开一个文件、读取里面的内容，然后输出到`stdout`。

举个例子：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # echo hello > test.txt 
~/codeDir/phpCode/swoole-src-4.4.4 # cat test.txt 
hello
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

我们先往文件`test.txt`里面写入字符串`hello`，然后通过命令`cat`读取到`test.txt`文件的内容输出到`stdout`。但是，如果`cat`命令后面没有接文件名会怎么样？

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # cat

```

我们发现，这个进程会阻塞起来。然后，如果我们输入一些字符串按下回车：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # cat
hello
hello

```

我们发现，会打印出我们敲入的字符串`hello`。这说明了什么问题？说明`cat`后面不接文件名，那么就会读取`stdin`的内容，然后输出到`stdout`。

我们继续来看这个例子：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # echo in > in.txt
~/codeDir/phpCode/swoole-src-4.4.4 # cat in.txt > out.txt
~/codeDir/phpCode/swoole-src-4.4.4 # cat out.txt 
in
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

我们把文件`in.txt`里面的内容通过`cat`命令输出后通过`>`重定向到了文件`out.txt`里面，所以我们可以在文件`out.txt`里面看到内容`in`。

我们再来看一个例子：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # cat > test.txt

```

使用了`>`意味着我们想把第一个命令的`stdout`重定向到文件`test.txt`里面对吧。我们输入内容：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # cat > test.txt 
hello
^C
~/codeDir/phpCode/swoole-src-4.4.4 # cat test.txt 
hello
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

我们输入`hello`之后按下回车键，然后按下`ctrl + c`关闭进程。可以看到，文件`test.txt`里面有了内容`hello`。

最后，我们来完成一个高级的例子：

```shell
~/codeDir/phpCode/swoole-src-4.4.4 # cat > test.txt << EOF
> world
> EOF
~/codeDir/phpCode/swoole-src-4.4.4 # cat test.txt 
world
~/codeDir/phpCode/swoole-src-4.4.4 # 
```

通过上面的例子，我相信大家可以很好的例子这条命令了。我们通过`heredoc`语法把`world`字符串作为`stdin`，然后因为`cat`命令后面没有根文件名，所以会去读取`stdin`里面的内容，即`world`，然后输出到`stdout`，最后通过`>`重定向到文件`test.txt`里面。

