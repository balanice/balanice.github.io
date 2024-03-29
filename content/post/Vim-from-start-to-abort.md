---
title: "Vim 从入门到放弃"
date: 2021-01-17T08:12:37+08:00
draft: true
tags: Linux
---

## Vim 是什么

Vim - Vi improved, 是 Vi 这个命令行编辑器的优化版本. 它的操作可以不依赖鼠标, 实现文本的快速编辑, 打开大文件毫无压力.

## Vim 怎么用

### Vim 基础教程

Vim 自带教程, 在命令行中输入 `vimtutor` 即可进入, 完成这个教程以后就能掌握 Vim 的基本操作.

### vimrc 配置

学习完成 Vim 的基础操作之后, 就必须学习一下 Vim 的配置, 通过对 Vim 的配置, 可以增强 Vim 的使用体验.

Vim 的配置文件为 `.vimrc`, Linux 系统位于 `~/.vimrc`, Windows 路径为 `C:\Users\username\_vimrc`, 如果该路径下没有可以手动创建, 常用配置如下:

```vim
"显示行号
set number

"代码高亮
syntax on

"启用鼠标
set mouse-=a

"自动缩进
set expandtab
set tabstop=4
set softtabstop=4
set shiftwidth=4
set backspace=2
set textwidth=79
set autoindent
set smartindent		

set nocompatible "使用vim自己的键盘

"自动补全括号
:inoremap ( ()<esc>i
:inoremap { {}<esc>i
:inoremap [ []<esc>i
:inoremap " ""<esc>i
:inoremap ' ''<esc>i

"支持中文 
set fileencodings=utf-8,gbk    
set ambiwidth=double  

"补全函数 ctrl + x ctrl + o  

"使用pydiction补全python代码
filetype on
filetype plugin on
filetype indent on
autocmd FileType python set omnifunc=pythoncomplete#Complete    

let g:pydiction_location="~/.vim/tools/pydiction/complete-dict"

" 使用C样式的缩进
set cindent

"设定默认编码
set encoding=utf-8
set fileencoding=chinese
set fenc=utf-8
set fencs=utf-8,gbk,cp936,gb2312,latin1

"颜色配置
colorscheme slate 

"设置高亮搜索
set hlsearch

set showcmd

"添加Pyhon和C的首行标注
if filereadable("/home/force/.vim/template/py")
    autocmd BufNewFile *.py 0 read /home/force/.vim/template/py
endif

if filereadable("/home/force/.vim/template/c")
    autocmd BufNewFile *.c 0 read /home/force/.vim/template/c
endif

"添加c++模板"
if filereadable("/home/force/.vim/template/cpp")
    autocmd BufNewFile *.cpp 0 read /home/force/.vim/template/cpp
endif

"required
filetype off        

set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()

"let Vundle manage Vundle, required
Plugin 'VundleVim/Vundle.vim'

"YouCompleteMe
Plugin 'Valloric/YouCompleteMe'

"Color
Plugin 'tomasr/molokai'
Plugin 'flazz/vim-colorschemes'

call vundle#end()
filetype plugin indent on
```

### Vim 插件

Vim 使用 [Vundle](https://github.com/VundleVim/Vundle.vim) 管理插件, 通过它可以给 Vim 安装各种插件, 如代码补全插件 [YouCompleteMe](https://github.com/ycm-core/YouCompleteMe)

## 为什么说放弃 Vim

Vim 作为编辑器异常强大, 尤其是打开大文件的时候, 会导致 sublime 和 notepad++ 崩溃的大文件 Vim 应付起来毫无压力, 纯键盘操作的理念也很好, 但是它毕竟不是 IDE, 在开发项目需要浏览多个文件的时候往往力不从心, 学习曲线陡峭对新手不友好, 插件安装繁琐, 所以我最后还是投奔了 VSCode, 而 Vim 只用来编辑配置文件或查看 log.