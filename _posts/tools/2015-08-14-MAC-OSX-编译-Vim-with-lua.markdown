---
layout: post
title: MAC OSX 编译 Vim with lua
category: 工具
tags: Vim Lua 
keywords:
description:
---

## 前言 
最近想学习 Go develop，所以想在 MAC 上搭一个 Go 开发环境。万万没想到啊，MAC 上的 Vim 还是 7.3，所以，首先第一件事就是升级到 7.4，因为目测很多插件需要神马 Lua，比方说 neocomplete.

---

## 使用 Homebrew ?

额，理论上讲这是最简单的。

```
brew install vim --with-lua
``` 

整个世界本来就因此而清净了。可是万万没想到啊，我的 MAC OSX 10.10 如此调皮

```
qmhysadeiMac:~ Bob$ brew install vim --with-lua
==> Downloading https://mirrors.kernel.org/debian/pool/main/v/vim/vim_7.4.712.orig.tar.gz
Already downloaded: /Library/Caches/Homebrew/vim-7.4.712.tar.gz
==> ./configure --prefix=/usr/local --mandir=/usr/local/Cellar/vim/7.4.712/share/man --enable-multibyte --with-tlib=ncurses --enable-cscope --with-features=huge --with-compiledby=Homebrew --enable-luainterp    --enable-perlinterp --enable-p
==> make
      _luaV_debug in if_lua.o
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
make[1]: *** [vim] Error 1
make: *** [first] Error 2

READ THIS: https://git.io/brew-troubleshooting

These open issues may also help:
vim  7.4.488 build fails if Homebrew ruby is installed but /usr/bin/ruby comes first in PATH (https://github.com/Homebrew/homebrew/issues/33705)
```

它就这样报错了，无缘无故的报错了。而且，`我不知道怎么去修改`，这尼玛才是重点。

---

## 编译安装 Vim with Lua

在不能使用 Homebrew 的时候也不要迷茫。毕竟我们都是用 *nix 的人。那就自己动手编译咯。

#### 1. 下载 Vim 源码
官网下载就好了嘛。
找个地方解压一下。
在终端进入解压的目录。

`注意：`
首先要修改一下源码，在 src 目录下找到 `os_unix.h`, 然后在适当的位置加上

```
#include <AvailabilityMacros.h>  
```

#### 2.安装 Lua

使用 brew 安装就好了嘛。

```
brew install lua
```

不能安装？ 自己下载源码编译一个嘛。

`[option]`
根据需要是否安装 LuaJit, 其实我不知道这个是干嘛用的。
首先去下载 http://luajit.org/download/LuaJIT-2.0.4.tar.gz , 然后解压

```
tar -xzvf LuaJIT-2.0.4.tar.gz
cd LuaJIT-2.0.4
// 使用默认安装路径
sudo make
sudo make install
```

#### 3.编译 Vim

> * 终端进入 Vim 的解压目录
> * 终端执行：
```
 ./configure --with-features=huge --enable-pythoninterp=yes  --enable-cscope --enable-fontset --enable-perlinterp --enable-rubyinterp --with-python-config-dir=/usr/lib/python2.7/config --with-luajit --enable-luainterp --with-lua-prefix=/usr/local  
```
> * 终端: `sudo make`
> * 终端: `sudo make install`

#### 4. 额外的收尾工作

添加vim命令的别名，如果是bash（默认shell）
在.bash_profile中添加一行

```
alias vim=‘/usr/local/bin/vim'
```

然后在终端中执行
`source ~/.bash_profile`

> **一些无关的说明：**
> .bash_profile 里面可以设置 GOPATH 或者一些其他设置信息，我的 .bash_profile 里面有 Maven 和 GOPATH,
> 还有一段简单的 `Visual Studio Code` 的快捷启动命令

#### 5. 查看最终成果
如果幸运的话，现在 Vim with lua 已经安装成功
终端 `:vim --version`

```
qmhysadeiMac:go Bob$ vim --version
VIM - Vi IMproved 7.4 (2013 Aug 10, compiled Aug 14 2015 12:32:05)
MacOS X (unix) version
Included patches: 1-712
Compiled by Bob@qmhysadeiMac.local
Huge version without GUI.  Features included (+) or not (-):
+acl             +farsi           +mouse_netterm   +syntax
+arabic          +file_in_path    +mouse_sgr       +tag_binary
+autocmd         +find_in_path    -mouse_sysmouse  +tag_old_static
-balloon_eval    +float           +mouse_urxvt     -tag_any_white
-browse          +folding         +mouse_xterm     -tcl
++builtin_terms  -footer          +multi_byte      +terminfo
+byte_offset     +fork()          +multi_lang      +termresponse
+cindent         -gettext         -mzscheme        +textobjects
-clientserver    -hangul_input    +netbeans_intg   +title
+clipboard       +iconv           +path_extra      -toolbar
+cmdline_compl   +insert_expand   +perl            +user_commands
+cmdline_hist    +jumplist        +persistent_undo +vertsplit
+cmdline_info    +keymap          +postscript      +virtualedit
+comments        +langmap         +printer         +visual
+conceal         +libcall         +profile         +visualextra
+cryptv          +linebreak       +python          +viminfo
+cscope          +lispindent      -python3         +vreplace
+cursorbind      +listcmds        +quickfix        +wildignore
+cursorshape     +localmap        +reltime         +wildmenu
+dialog_con      +lua             +rightleft       +windows
+diff            +menu            +ruby            +writebackup
+digraphs        +mksession       +scrollbind      -X11
-dnd             +modify_fname    +signs           -xfontset
-ebcdic          +mouse           +smartindent     -xim
+emacs_tags      -mouseshape      -sniff           -xsmp
+eval            +mouse_dec       +startuptime     -xterm_clipboard
+ex_extra        -mouse_gpm       +statusline      -xterm_save
+extra_search    -mouse_jsbterm   -sun_workshop    -xpm
   system vimrc file: "$VIM/vimrc"
     user vimrc file: "$HOME/.vimrc"
 2nd user vimrc file: "~/.vim/vimrc"
      user exrc file: "$HOME/.exrc"
  fall-back for $VIM: "/usr/local/share/vim"
Compilation: gcc -c -I. -Iproto -DHAVE_CONFIG_H   -DMACOS_X_UNIX  -g -O2 -U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=1      
Linking: gcc   -L. -L/usr/local/lib  -L/usr/local/lib -o vim        -lm -lncurses  -liconv -framework Cocoa  -pagezero_size 10000 -image_base 100000000 -L/usr/local/lib -lluajit-5.1 -fstack-protector  -L/System/Library/Perl/5.18/darwin-thread-multi-2level/CORE -lperl -framework Python   -lruby.2.0.0 -lobjc   
```

注意到 lua 前面是 (+) 说明 lua 安装成功。

---

### 参考

[1]  <http://blog.csdn.net/u011542994/article/details/39058779>                             
[2]  <http://wuxu92.github.io/z-compile-vim-with-lua-support-in-centos-7/>                         
[3]  <http://www.cnblogs.com/spch2008/p/4593370.html>   









