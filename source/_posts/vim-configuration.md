---
title: CentOS7 vim个人配置
date: 2020-09-22 16:54:00
tags: vim
categories: vim
---

因为最终想要安装YCM插件，而最新的YCM插件只支持 python3.6 和 vim8.1 以上的版本，所以需要更新系统自带的 python 和 vim（如果以安装最新的环境，则可忽略）。
<!-- more -->

全文最底下添加了vimrc配置的信息可自行下载。

## 安装python3

安装相关编译环境：

```text
yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gcc make
```

下载 python 安装包，此处我下载的是 3.7.4 版本的 python，可自行选择最新的版本下载安装：

```text
mkdir ~/download
cd ~/download
wget https://www.python.org/ftp/python/3.7.4/Python-3.7.4.tgz
```

解压：

```text
tar -vxzf Python-3.7.4.tgz
```

编译安装：

```text
cd Python-3.7.4

./configure --prefix=/usr/local/python3 --with-ssl --enable-shared --enable-optimizations

make && make install
```

此处的添加 --enable-shared 参数是因为YCM插件需要。
安装完毕后，添加软链接：

```text
ln -sf /usr/local/python3/bin/python3 /usr/bin/python3
ln -sf /usr/local/python3/bin/pip3 /usr/bin/pip3
```

执行命令验证 python 版本：

```text
python --version
Python 2.7.12

python3 --version
Python 3.7.4
```

---

## 安装vim8

下载vim：

```text
cd ~/download
git clone https://github.com/vim/vim.git
```

编译安装支持 python3 的 vim8：

```text
cd vim

./configure \
--with-features=huge \
--enable-multibyte \
--enable-rubyinterp=yes \
--enable-python3interp=yes \
--with-python-config-dir=/usr/local/python3/lib/python3.7/config-3.7m-x86_64-linux-gnu \
--enable-perlinterp=yes \
--enable-luainterp=yes \
--enable-gui=gtk2 \
--enable-cscope \
--prefix=/usr/local/vim8

make && make install
```

安装完毕后，添加软链接：

```text
ln -sf /usr/local/vim8/bin/vim /usr/bin/vim # 覆盖旧版本的vim
```

验证vim：

```text
vim --version
```

查看结果：
![vim-version](vim-version.png)

如果红框中 python3 前面显示 `+` ，则表示 vim 成功支持 python3。
至此以安装完毕python3 + vim8。
接下来对 vim 进行相关配置。

## vim配置

### vim基础配置

首先对 vim 进行一个基础配置

```text
set number          " 显示行号
syntax on           " 语法高亮
set showcmd         " 输入的命令显示出来，看的清楚些
set cursorline      " 当前行显示
set autoindent      " 自动缩进
set cindent
set tabstop=4       " tab缩进4个空格
set softtabstop=4
set shiftwidth=4
set expandtab
set incsearch       " 开启实时搜索功能
set ignorecase      " 搜索时大小写不敏感
set hlsearch        " 高亮显示搜索结果
set foldenable      " 允许折叠
set foldmethod=manual   " 手动折叠
set nocompatible    " 去掉讨厌的有关vi一致性模式，避免以前版本的一些bug和局限
set clipboard+=unnamed  " 共享剪贴板
set autowrite       " 自动保存
set confirm         " 在处理未保存或只读文件的时候，弹出确认
set ruler           " 打开状态栏标尺
set langmenu=zh_CN.UTF-8    " 语言设置
set helplang=cn
set laststatus=2    " 总是显示状态行
set linespace=0     " 字符间插入的像素行数目
set backspace=2     " 使回格键（backspace）正常处理indent, eol, start等
set showmatch       " 高亮显示匹配的括号
set scrolloff=3     " 光标移动到buffer的顶部和底部时保持3行距离
set completeopt=menu
" 相关颜色替换
hi Search term=standout cterm=bold ctermfg=7 ctermbg=1
hi SpellBad term=reverse ctermfg=15 ctermbg=9 guifg=White guibg=Red
let mapleader=','
```

### vim插件安装

#### 安装vim插件管理器(vim-plug)

因为需要安装的 vim 插件很多，所以需要有一个管理工具来对这些插件进行统一管理。目前主流使用的是 [vim-plug](https://github.com/junegunn/vim-plug) 插件管理器，该插件能够异步并行进行快速安装、更新和卸载插件。

下载 plug.vim 文件到 `~/.vim/autoload/` 文件夹中：

```text
curl -fLo ~/.vim/autoload/plug.vim --create-dirs https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

执行命令的时候，发现 raw.githubusercontent.com 地址可能访问不了，需要配置 hosts 才能进行访问，配置的方法网上很多，可自行查询解决。
下载完成后，在 vim 的配置文件( `~/.vimrc` )中添加以下代码：

```text
" Specify a directory for plugins
" - For Neovim: stdpath('data') . '/plugged'
" - Avoid using standard Vim directory names like 'plugin'
call plug#begin('~/.vim/plugged')
" 以下添加需要安装的插件
" 可在github上查找需要安装的插件，以 Plug 'xxx/xxx' 的形式来添加，如下：

" YCM插件
Plug 'Valloric/YouCompleteMe'

" Initialize plugin system
call plug#end()
```

配置完成后，打开vim，输入`:PlugInstall`进行安装。

vim-plug操作命令：

|命令|描述|
|---|---|
|PlugInstall [name ...] [#threads]|安装插件|
|PlugUpdate [name ...] [#threads]|更新插件|
|PlugClean[!]|卸载插件|
|PlugUpgrade|更新vim-plug|
|PlugStatus|查看插件状态|
|PlugDiff|审查插件|
|PlugSnapshot[!] [output path]|生成用于恢复插件当前快照的脚本|

下面安装vim实用的插件。

---

#### NERDTree文件树目录

[NERDTree](https://github.com/preservim/nerdtree)是一款可以提供树形目录的插件，方便浏览目录结构和进行文件跳转等。

```text
Plug 'preservim/nerdtree'
```

相关配置如下：

```text
" nerdtree
autocmd vimenter * NERDTree  "自动开启Nerdtree
" 关闭所有文本窗口时自动退出vim,否则需要两次退出才可
autocmd BufEnter * if 0 == len(filter(range(1, winnr('$')), 'empty(getbufvar(winbufnr(v:val), "&bt"))')) | qa! | endif
let NERDTreeShowHidden=1    " 是否显示隐藏文件
let NERDTreeIgnore=['\.pyc','\~$','\.swp']  " 忽略文件的显示

" 设置NerdTree 按f2打开或关闭NERDTree
map <F2> :NERDTreeMirror<CR>
map <F2> :NERDTreeToggle<CR>
```

#### nerdtree-git-plugin目录树git文件状态

[nerdtree-git-plugin](https://github.com/Xuyuanp/nerdtree-git-plugin)能显示git管理的项目文件的变更状态。

```text
Plug 'Xuyuanp/nerdtree-git-plugin'
```

相关配置如下：

```text
let g:NERDTreeIndicatorMapCustom = {
            \ "Modified"  : "✹",
            \ "Staged"    : "✚",
            \ "Untracked" : "✭",
            \ "Renamed"   : "➜",
            \ "Unmerged"  : "═",
            \ "Deleted"   : "✖",
            \ "Dirty"     : "✗",
            \ "Clean"     : "✔︎",
            \ 'Ignored'   : '☒',
            \ "Unknown"   : "?"
            \ }
```

#### tagbar函数列表

[tagbar](https://github.com/preservim/tagbar)能显示出所写代码的函数、变量、类等等。
安装 taglist 之前，需要先安装 ctags。
ctags 的功能：扫描指定的源文件，找出其中所包含的语法元素，并将找到的相关内容记录下来。
安装ctags：
>到 http://ctags.sourceforge.net/ 下载最新的ctags源码(当前最新版本为 ctags-5.8.tar.gz)
>解压并安装：
>tar vxzf ctags-5.8.tar.gz
>cd ctags-5.8
>./configure && make && make install

ctags的使用方法可以自行百度查询。

接下来安装tagbar：

```text
Plug 'preservim/tagbar'
```

相关配置如下：

```text
" [tagbar]
" 设置tagbar使用的ctags的插件,必须要设置对
let g:tagbar_ctags_bin='/usr/bin/ctags'
" 设置tagbar的窗口宽度
let g:tagbar_width=35
" 设置tagbar的窗口显示的位置,默认右边
let g:tagbar_right=1
" 打开文件自动 打开tagbar
" autocmd BufReadPost *.cpp,*.c,*.h,*.hpp,*.cc,*.cxx call tagbar#autoopen()
" 这是tagbar一打开，光标即在tagbar页面内，默认在vim打开的文件内
let g:tagbar_autofocus = 1
"设置标签不排序，默认排序
let g:tagbar_sort = 0
" 映射tagbar的快捷键
nnoremap <silent> <F3> :TagbarToggle<CR>
```

#### vim-airline状态栏美化

[vim-airline](https://github.com/vim-airline/vim-airline) 和 [vim-airline-themes](https://github.com/vim-airline/vim-airline-themes) 搭配能将vim的底部状态增强/美化。

```text
Plug 'vim-airline/vim-airline'
Plug 'vim-airline/vim-airline-themes'
```

相关配置如下：

```text
set t_Co=256      "在windows中用xshell连接打开vim可以显示色彩
let g:airline#extensions#tabline#enabled = 1   " 是否打开tabline
"这个是安装字体后 必须设置此项
let g:airline_powerline_fonts = 1
set laststatus=2  "永远显示状态栏
let g:airline_theme='bubblegum' "选择主题
let g:airline#extensions#tabline#enabled=1    "Smarter tab line:显示窗口tab和buffer
"let g:airline#extensions#tabline#left_sep = ' '  "separater
"let g:airline#extensions#tabline#left_alt_sep = '|'  "separater
"let g:airline#extensions#tabline#formatter = 'default'  "formater
let g:airline_left_sep = '▶'
let g:airline_left_alt_sep = '❯'
let g:airline_right_sep = '◀'
let g:airline_right_alt_sep = '❮'
```

#### nerdcommenter批量注释

[nerdcommenter](https://github.com/preservim/nerdcommenter)是一款对代码进行批量注释的插件。

```text
Plug 'preservim/nerdcommenter'
```

相关配置如下：

```text
let g:NERDSpaceDelims = 1   " 注释中间加一个空格  
let g:NERDDefaultAlign = 'left' " 多行注释向左对齐
```

使用方法：
在 Normal 和 Visual 模式下：

* \<leader>ca在可选的注释方式之间切换，比如C/C++ 的块注释/* */和行注释//  
* \<leader>cc注释当前行
* \<leader>c\<space> 切换注释/非注释状态
* \<leader>cs 以”性感”的方式注释
* \<leader>cA 在当前行尾添加注释符，并进入Insert模式
* \<leader>cu 取消注释
* \<leader>c$ 从光标开始到行尾注释  ，这个要说说因为c$也是从光标到行尾的快捷键，这个按过逗号（，）要快一点按c$
* 2\<leader>cc 光标以下count行添加注释
* 2\<leader>cu 光标以下count行取消注释
* 2\<leader>cm:光标以下count行添加块注释(2,cm)
* Normal模式下，几乎所有命令前面都可以指定行数
* Visual模式下执行命令，会对选中的特定区块进行注释/反注释

#### golang相关插件

[vim-go](https://github.com/fatih/vim-go) 是一款go代码高亮和语法检查的插件。

```text
Plug 'fatih/vim-go', { 'do': ':GoUpdateBinaries' }
```

[gocode](https://github.com/nsf/gocode) 是一款go的代码提示插件。
首先要正确设置GOROOT、GOPATH、GOBIN等几个环境变量。
下载gocode：

```text
mkdir -p $GOPATH/src/github.com/nsf

cd $GOPATH/src/github.com/nsf

git clone https://github.com/nsf/gocode.git
```

编译安装gocode：

```text
cd gocode

go build

go install
```

编译完成后生成一个 gocode 的可执行文件，并被放到 $GOBIN 目录下。
接下来安装vim-gocode插件:

```text
Plug 'nsf/gocode', { 'rtp': 'vim', 'do': '~/.vim/plugged/gocode/vim/symlink.sh' }
```

#### YouCompleteMe代码补全插件

[YouCompleteMe](https://github.com/ycm-core/YouCompleteMe) 是一款非常强大的异步代码自动补全，支持非常多的编程语言，如：C/C++, Golang, Python, Rust, Java等。

准备依赖环境：

```text
yum install cmake

yum install python-devel libffi-devel graphviz-devel elfutils-libelf-devel readline-devel libedit-devel libxml2-devel protobuf-devel gtext-devel doxygen swig
```

下载安装：

```text
git clone https://github.com/ycm-core/YouCompleteMe.git ~/.vim/plugged/YouCompleteMe

cd ~/.vim/plugged/YouCompleteMe

python3 install.py --all
```

相关配置如下：

```text
let g:ycm_confirm_extra_conf=0      " 关闭加载.ycm_extra_conf.py提示
let g:ycm_complete_in_comments = 1  "在注释输入中也能补全
let g:ycm_complete_in_strings = 1   "在字符串输入中也能补全
let g:ycm_collect_identifiers_from_tags_files=1                 " 开启 YCM 基于标签引擎
let g:ycm_collect_identifiers_from_comments_and_strings = 1   "注释和字符串中的文字也会被收入补全
let g:ycm_seed_identifiers_with_syntax=1   "语言关键字补全, 不过python关键字都很短，所以，需要的自己打开
let g:ycm_min_num_of_chars_for_completion=2                     " 从第2个键入字符就开始罗列匹配项
" 引入，可以补全系统，以及python的第三方包 针对新老版本YCM做了兼容
if !empty(glob("~/..vim/plugged/YouCompleteMe/third_party/ycmd/.ycm_extra_conf.py"))
    let g:ycm_global_ycm_extra_conf = "~/..vim/plugged/YouCompleteMe/third_party/ycmd/.ycm_extra_conf.py"
endif
"mapping
nmap <leader>gd :YcmDiags<CR>
nnoremap <leader>gl :YcmCompleter GoToDeclaration<CR>           " 跳转到申明处
nnoremap <leader>gf :YcmCompleter GoToDefinition<CR>            " 跳转到定义处
nnoremap <leader>gg :YcmCompleter GoToDefinitionElseDeclaration<CR>
" 直接触发自动补全
let g:ycm_key_invoke_completion = '<C-Space>'
" 黑名单,不启用
      \ 'tagbar' : 1,
      \ 'gitcommit' : 1,
      \}
```

---

此处还没有结束，未来如果碰到好用的插件，会继续更新。。。
以上所有的配置我将它上传到了github上，有需要的可自行下载 [vimrc](https://github.com/Grizzly1127/vim-config)。
