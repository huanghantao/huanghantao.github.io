---
title: 使用nvim来调试phpactor
date: 2021-06-17 17:59:27
tags:
- PHP
---

一、启动服务器

首先安装`phpactor`，安装完之后，执行启动命令：

```bash
phpactor language-server --address=127.0.0.1:9901 -vvv
```

二、配置`init.vim`如下：

```vim
call plug#begin('~/.vim/plugged')

Plug 'neoclide/coc.nvim', {'branch': 'release'}

call plug#end()
```

三、然后安装插件

在`nvim`里面输入：

```vim
:PlugInstall
```

四、配置CocConfig

在`nvim`里面输入：

```vim
:CocConfig
```

然后配置如下：

```json
{
"languageserver": {
    "socketserver": {
    "filetypes": ["php"],

    "host": "127.0.0.1",
    "port": 9901
    }
}
}
```

然后，打开一个`PHP`文件，即可调试服务器了。
