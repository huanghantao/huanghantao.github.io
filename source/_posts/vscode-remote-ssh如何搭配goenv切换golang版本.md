---
title: vscode remote ssh如何搭配goenv切换golang版本
date: 2022-03-20 17:04:29
tags:
---

问题大概就是，当我在远程服务器使用`goenv`安装了多个`golang`版本的时候，使用`vscode`的`remote ssh`方式远程连接服务器，`vscode`的`shell`无法自动切换到对应的`golang`版本。

发现`vscode shell`里面的环境变量用的是老的，似乎没有加载`~/.bash_profile`里面的环境变量。于是找到了`remote-ssh`的这个[issue](https://github.com/microsoft/vscode-remote-release/issues/1671)。争论了挺久的，有兴趣可以看完。

解决方法是在`settings.json`文件里面配置指定要使用`login shell`：

```json
{
    "terminal.integrated.defaultProfile.linux": "bash",
    "terminal.integrated.profiles.linux": {
        "bash": {
            "path": "bash",
            "args": [
                "-l"
            ]
        }
    }
}
```

注意，这个配置只能够解决`vscode shell`中`golang`版本没切换的问题，但是`vscode golang`插件此时用的`golang`版本可能也是错误的。我们还需要给`vscode golang`插件配置一下，例如：

```json
"go.goroot": "/root/.goenv/versions/1.17.8",
"go.gopath": "/root/go/1.17.8"
```

所以，最终的配置如下：

```json
{
    "terminal.integrated.defaultProfile.linux": "bash",
    "terminal.integrated.profiles.linux": {
        "bash": {
            "path": "bash",
            "args": [
                "-l"
            ]
        }
    },
    "go.goroot": "/root/.goenv/versions/1.17.8",
    "go.gopath": "/root/go/1.17.8"
}
```
