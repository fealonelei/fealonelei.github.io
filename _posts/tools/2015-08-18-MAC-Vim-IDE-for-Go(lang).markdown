---
layout: post
title: MAC Vim IDE for Go(lang)
category: 工具
tags: Vim Golang 
keywords: Vim for Go, Vim IDE
description:
---

## 前言
Vim 是极具生产力的工具。我个人选择 Vim 作为 Go dev 的工具。
本文不讨论 Which one is 世界上最好的编辑工具或者 ~~PHP 是不是世界上最好的语言~~ 。
我主要是对自己配置 Vim for Go 的一个总结(**OS: MAC OSX 10.10**)。

如果对 Vim 操作一无所知，请先去看一下左耳朵耗子叔写的[简明 Vim 练级攻略](http://coolshell.cn/articles/5426.html)，并且及时使用一些 ~~404~~ 搜索引擎补充知识。
在阅读本文之前，请确定：

> * 有基本 Vim 知识
> * 懂得基本 Terminal 操作
> * 有一颗往死了折腾自己的心
> * 有一颗往死了折腾自己的心
> * 有一颗往死了折腾自己的心

---

## 配置 basic vim
Well，we need Vim with lua. If Using OSX, here is a [artical](http://fealonelei.github.io/2015/08/14/MAC-OSX-%E7%BC%96%E8%AF%91-Vim-with-lua.html) for consideration.

---

## Install vim-plugin package manager
There are a several of choise, such as [Pathogen](https://github.com/tpope/vim-pathogen), [Vundle](http://www.vim.org/scripts/script.php?script_id=3458).
I am using Pathogen.
Well, installation of Pathogen is super easy, so is Vundle.
Just using terminal(copy both lines once) :

```
mkdir -p ~/.vim/autoload ~/.vim/bundle && \
curl -LSso ~/.vim/autoload/pathogen.vim https://tpo.pe/pathogen.vim
```

Well, don't forget to add to **the first line** of .vimrc ( [what is .vimrc](https://www.google.com/webhp?sourceid=chrome-instant&ion=1&espv=2&es_th=1&ie=UTF-8#newwindow=1&safe=strict&q=what+is+vimrc+file) ):

```
execute pathogen#infect() 
```

---
`一些不相关的说明:` <br />
**如果用 Vim 来编辑 .vimrc , 非常诡异的事情是不能正常将浏览器或者其他人的 .vimrc 复制粘贴进去,这尼玛也算奇葩一景**

---

## Install the Vim-Go plugin 
Super easy.
Using terminal:

```
git clone https://github.com/fatih/vim-go.git ~/.vim/bundle/vim-go
```

add below to .vimrc:

```
syntax enable              
filetype plugin indent on  
set number                 
let g:go_disable_autoinstall=0
```

If you wanna config Vim-Go more flexible, find more info on [GitHub Vim-Go](https://github.com/fatih/vim-go).

---


## Install gotags && tagbar
Since I am using mac, Homebrew will offer great convenience. 

#### 1. install ctags
Using terminal:

```
brew install ctags
```

#### 2. install gotags
Using terminal:

```
go get -u github.com/jstemmer/gotags
```

#### 3. configure .vimrc
add 

```
let g:go_gotags_bin='~/go/bin/gotags'
let ctagsbin = 'gotags'
" check if vim-go is available and has the binary
if !executable('gotags') && !exists("g:go_gotags_bin")
    let ctagsbin = expand(g:go_gotags_bin),
endif

let g:tagbar_type_go = {
    \ 'ctagstype' : 'go',
    \ 'kinds'     : [
        \ 'p:package',
        \ 'i:imports:1',
        \ 'c:constants',
        \ 'v:variables',
        \ 't:types',
        \ 'n:interfaces',
        \ 'w:fields',
        \ 'e:embedded',
        \ 'm:methods',
        \ 'r:constructor',
        \ 'f:functions'
    \ ],
    \ 'sro' : '.',
    \ 'kind2scope' : {
        \ 't' : 'ctype',
        \ 'n' : 'ntype'
    \ },
    \ 'scope2kind' : {
        \ 'ctype' : 't',
        \ 'ntype' : 'n'
    \ },
    \ 'ctagsbin'  : expand(g:go_gotags_bin), 
     \'ctagsargs' : '-sort -silent'
\ }
```

#### 4. Install tagbar 
Using terminal:

```
git clone https://github.com/fatih/vim-go.git ~/.vim/bundle/tagbar
```

then, configure .vimrc:

```
nmap <F8> :TagbarToggle<CR>
```

---

## Install neocomplete 实时补全

#### 1.Using terminal:

```
git clone https://github.com/fatih/vim-go.git ~/.vim/bundle/neocomplete.vim
```

#### 2. **编辑 .vimrc:**
the author has a great configuration example on [github](https://github.com/Shougo/neocomplete.vim). 
Using it is perfect.

#### 3. how to use:
点击 tab 键.

---

## extra sugar

why not install more helpful and productive plugin, such as Nerdtree, Nerdcommenter, snippet, etc ?   <br />
explore more. have fun.

For 天朝程序员 <br />
<https://github.com/yangyangwithgnu/use_vim_as_ide> 

