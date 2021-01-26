# vim 编辑器之神

Vim是一个类似于Vi的著名的功能强大、高度可定制的文本编辑器，在Vi的基础上改进和增加了很多特性。

---

vimrc的存放位置： 
系统 vimrc 文件: "$VIM/vimrc" 
用户 vimrc 文件: "$HOME/.vimrc" 
用户 exrc 文件: "$HOME/.exrc" 
系统 gvimrc 文件: "$VIM/gvimrc" 
用户 gvimrc 文件: "$HOME/.gvimrc" 
系统菜单文件: "$VIMRUNTIME/menu.vim" 
$VIM 预设值: "/usr/share/vim" 

"是否兼容VI，compatible为兼容，nocompatible为不完全兼容 
"如果设置为compatible，则tab将不会变成空格 
set nocompatible

"执行 Vim 缺省提供的 .vimrc 文件的示例，包含了打开语法加亮显示等最常用的功能
source $VIMRUNTIME/vimrc_example.vim

"设置鼠标运行模式为WINDOWS模式 
source $VIMRUNTIME/mswin.vim
behave mswin

"gVim下生效，个人觉得不必这么设置
if has("gui_running")
set guioptions-=m " 隐藏菜单栏
set guioptions-=T " 隐藏工具栏
set guioptions-=L " 隐藏左侧滚动条
set guioptions-=r " 隐藏右侧滚动条
set guioptions-=b " 隐藏底部滚动条
set showtabline=0 " 隐藏Tab栏
endif

"语法高亮 
syntax on 
 
"自动缩进 
set autoindent 

"不产生备份文件
set nobackup

"不产生undo文件.un~
set noundofile
"不产生备份的文件
set noswapfile


---

## 通过vundle管理vim
vundle是用于管理vim插件的一个工具，使用它是可以方便的从github上下载相应的插件，所以在使用前需安装好git
`sudo apt-get install vim git`

同步vundle.vim到~/.vim/bundle/目录下,将配置vundle和插件的语句添加到.vimrc文件的最顶部，确保vim在启动的时候优先加载

---

## 我的win下gVim配置

```
source $VIMRUNTIME/vimrc_example.vim
source $VIMRUNTIME/mswin.vim
behave mswin

set diffexpr=MyDiff()
function MyDiff()
  let opt = '-a --binary '
  if &diffopt =~ 'icase' | let opt = opt . '-i ' | endif
  if &diffopt =~ 'iwhite' | let opt = opt . '-b ' | endif
  let arg1 = v:fname_in
  if arg1 =~ ' ' | let arg1 = '"' . arg1 . '"' | endif
  let arg2 = v:fname_new
  if arg2 =~ ' ' | let arg2 = '"' . arg2 . '"' | endif
  let arg3 = v:fname_out
  if arg3 =~ ' ' | let arg3 = '"' . arg3 . '"' | endif
  if $VIMRUNTIME =~ ' '
    if &sh =~ '\<cmd'
      if empty(&shellxquote)
        let l:shxq_sav = ''
        set shellxquote&
      endif
      let cmd = '"' . $VIMRUNTIME . '\diff"'
    else
      let cmd = substitute($VIMRUNTIME, ' ', '" ', '') . '\diff"'
    endif
  else
    let cmd = $VIMRUNTIME . '\diff'
  endif
  silent execute '!' . cmd . ' ' . opt . arg1 . ' ' . arg2 . ' > ' . arg3
  if exists('l:shxq_sav')
    let &shellxquote=l:shxq_sav
  endif
endfunction

"启动后最大化 
au GUIEnter * simalt ~x 
"行高亮 
set cursorline

"文件编码
set fileencoding=gb2312
"内部编码
set encoding=cp936
"设定字符集
set fileencodings=utf-8,cp936,latin-1
"设置tab代表4个空格
set tabstop=4
"编辑模式的时候按退格键的时候退回缩进的长度
set softtabstop=4
"表示每一级缩进的长度
set shiftwidth=4
"设置成 expandtab 时，缩进用空格来表示
set expandtab

"不产生备份文件
set nobackup
set noundofile

"打开文件类型检测
filetype plugin indent on

"打开折叠功能
set fdm=syntax

syntax enable
colorscheme solarized
set background=light

"F2切换背景
function! UpdateSolarizedColor()
	let temp = &background
	if temp=="light"
		set background=dark
	else
		set background=light
	endif
endfunction
nmap <F2> :call UpdateSolarizedColor()<CR>

"F3打开配置文件
function! OpenVimrc()
	gvim $VIM\_vimrc
endfunction
nmap <F3> :call OpenVimrc()<CR>

"让vim首先在当前目录里寻找tags文件，所以要加分号，如果没有找到tags文件，或者没有找到对应的目标，就到父目录中查找，一直向上递归
set tags=D:\vimproject\ctags;tags;
"因为tags文件中记录的路径总是相对于tags文件所在的路径，所以要使用第二个设置项来改变vim的当前目录。如果不加入这两个语句，那么有的宏定义，还有一些就找不到了
set autochdir


"F12生成/更新tags文件 
function! UpdateTagsFile() 
    silent !ctags -R --languages=c++ --langmap=c++:+.c:+.ft:+.ic -h +.ft:+.ic --c++-kinds=+px --fields=+aiKSz --extras=+q -f D:\vimproject\ctags D:\EBSsvn\EBS\ebs\Branches\EBS\ZXCCPEBSV3.05_Branch\csd\src\*
endfunction 
nmap <F12> :call UpdateTagsFile()<CR> 
 
"Ctrl + F12删除tags文件 
function! DeleteTagsFile() 
    "Linux下的删除方法 
    "silent !rm tags 
    "Windows下的删除方法 
    silent !del /F /Q D:\vimproject\ctags 
endfunction 
nmap <C-F12> :call DeleteTagsFile()<CR> 
"退出VIM之前删除tags文件 
"au VimLeavePre * call DeleteTagsFile()

"F11生成/更新cscope文件 
function! UpdateCscopeFile()
	cs kill cscope.out 
	silent !del /F /Q D:\vimproject\cscope.out
    silent !d: & cd D:\vimproject & dir /s /b *.c *.cc *.h *cpp *.ft *.ic D:\EBSsvn\EBS\ebs\Branches\EBS\ZXCCPEBSV3.05_Branch\csd\src > cscope.files &  cscope -Rbk -i cscope.files
	cs add D:\vimproject\cscope.out
endfunction 
nmap <F11> :call UpdateCscopeFile()<CR> 

"Ctrl + F11删除cscope文件
function! DeleteCscopeFile() 
	cs kill cscope.out
    silent !del /F /Q D:\vimproject\cscope.out 
endfunction 
nmap <C-F11> :call DeleteCscopeFile()<CR>

cs a D:\vimproject\cscope.out

"ctags.exe放在$VIMRUNTIME目录下的ctags目录下
let Tlist_Ctags_Cmd = '"' . $VIMRUNTIME . '\ctags\ctags"'
let Tlist_Use_Right_Window=1
let Tlist_Inc_Winwidth=0
let Tlist_Auto_Open = 1
let Tlist_File_Fold_Auto_Close=1
let Tlist_Exit_OnlyWindow = 1
nnoremap <silent> <F8> :TlistToggle<CR>

"F5执行编译脚本 
function! SvnRunBat() 
    silent !D:\putty\svnrun.bat & PAUSE
endfunction 
nmap <F5> :call SvnRunBat()<CR> 

"F7加载NERDTree
nmap <F7> :NERDTreeToggle<CR>

"CtrlP的起始目录
let g:ctrlp_working_path_mode = 'ra'
let g:ctrlp_root_markers = ['*.spec']

```

---

## 我的最新Vim配置

```
"不产生备份文件
set nobackup
set noundofile

"启动后最大化
au GUIEnter * simalt ~x

"设置行高亮
set cursorline

"文件编码
set fileencoding=utf-8
"内部编码,linux下显示中文设置utf-8
set encoding=cp936
"设定字符集
set fileencodings=utf-8,cp936,latin-1

"设置文件格式为unix
set fileformats=unix

"设置tab代表4个空格
set tabstop=4
"编辑模式时退格键退回缩进的长度
set softtabstop=4
"每一级缩进的长度
set shiftwidth=4
"设置成expandtab时，缩进用空格表示
set expandtab

"打开文件类型检测
filetype plugin indent on

"打开折叠功能
set fdm=syntax

"设置是用solarized8配色方案
colorscheme solarized8
set background=light

"F2切换背景
function! UpdateSolarizedColor()
	let temp = &background
	if temp=="light"
		set background=dark
	else
		set background=light
	endif
endfunction
nmap <F2> :call UpdateSolarizedColor()<CR>

"F3打开配置文件
function! OpenVimrc()
	gvim $VIM\_vimrc
endfunction
nmap <F3> :call OpenVimrc()<CR>

"Ctrl + F4 删除文件换行
function! DeleteFileEnd()
    %s/\r//g
endfunction
nmap <C-F4> :call DeleteFileEnd()<CR>

" 显示状态行当前设置
set statusline

" 设置状态行显示常用信息
" %F 完整文件路径名
" %m 当前缓冲被修改标记
" %m 当前缓冲只读标记
" %h 帮助缓冲标记
" %w 预览缓冲标记
" %Y 文件类型
" %b ASCII值
" %B 十六进制值
" %l 行数
" %v 列数
" %p 当前行数占总行数的的百分比
" %L 总行数
" %{...} 评估表达式的值，并用值代替
" %{"[fenc=".(&fenc==""?&enc:&fenc).((exists("+bomb") && &bomb)?"+":"")."]"} 显示文件编码
" %{&ff} 显示文件类型
set statusline=%F%m%r%h%w%=\ [ft=%Y]\ %{\"[fenc=\".(&fenc==\"\"?&enc:&fenc).((exists(\"+bomb\")\ &&\ &bomb)?\"+\":\"\").\"]\"}\ [ff=%{&ff}]\ [asc=%03.3b]\ [hex=%02.2B]\ [pos=%04l,%04v][%p%%]\ [LN=%L]

" 设置 laststatus = 0 ，不显式状态行
" 设置 laststatus = 1 ，仅当窗口多于一个时，显示状态行
" 设置 laststatus = 2 ，总是显式状态行
set laststatus=2

"设置F7快捷键功能，使得在VIM右侧打开文件浏览,linux 下为20Vexplore!
nmap <F7> :20Lexplore!<CR>

"设置Netrw的list的格式
let g:netrw_liststyle = 3

"因为tags文件中记录的路径总是相对于tags文件所在的路径，所以要使用第二个设置项来改变vim的当前目录。如果不加入这两个语句，那么有的宏定义，还有一些就找不到了
set autochdir

"ctags.exe放在$VIMRUNTIME目录下的ctags目录下.linux下为"/usr/bin/ctags"
let Tlist_Ctags_Cmd = '"' . $VIMRUNTIME . '\ctags\ctags"' 
let Tlist_Inc_Winwidth=0
let Tlist_Exit_OnlyWindow = 1
nnoremap <silent> <F8> :TlistToggle<CR>

"设置项目路径,该路径后续可通过source导入文件修改
let project_dir = "D:\\vimproject"
"设置项目代码路径
let g:project_src = ''.project_dir.'\*'
"让vim首先在当前目录里寻找tags文件，所以要加分号，如果没有找到tags文件，或者没有找到对应的目标，就到父目录中查找，一直向上递归
let g:project_tags = '' . project_dir . '\_vim\ctags'
"tags需要使用set操作，但因为需要让其等于变量，所以使用let操作并加上&符号
let &tags = ''.project_tags.';tags;'

"F12生成/更新tags文件 
function! UpdateTagsFile(f_dir)
    let g:project_tags = '' . a:f_dir . '\_vim\ctags'
    let &tags = ''.g:project_tags.';tags;'
    let g:project_src = ''.a:f_dir.'\*'
    let func_cmd = '!ctags -R --languages=c++ --langmap=c++:+.c -h --c++-kinds=+px --fields=+aiKSz --extras=+q -f '.g:project_tags.' '.g:project_src
    execute func_cmd
endfunction 
nmap <F12> :call UpdateTagsFile(project_dir)<CR>

"F11 生成/更新cscope文件
function! UpdateCscopeFile(f_dir)
	cs kill cscope.out 
    "linux 下使用rm -f
    let func_cmd = '!del /F /Q '.a:f_dir.'\_vim\cscope.out'
    execute func_cmd
    "linux 下使用rm -f
    let func_cmd = '!del /F /Q '.a:f_dir.'\_vim\cscope.files'
    execute func_cmd
    "linux下使用'!find '.a:f_dir.' -name "*.[hc]*" >>
    "'.a:f_dir.'/.vim/cscope.files'
    let func_cmd = '!for /r '.a:f_dir.' \%i in (*.c *.cc *.h *.hh) do @echo \%i >> '.a:f_dir.'\_vim\cscope.files'
    execute func_cmd
    "linux下命令之间使用&&
    let func_cmd = '!cd '.a:f_dir.'\_vim\ & cscope -Rbk -i '.a:f_dir.'\_vim\cscope.files'
    execute func_cmd
    let func_cmd = 'cs add '.a:f_dir.'\_vim\cscope.out'
	execute func_cmd
endfunction 
nmap <F11> :call UpdateCscopeFile(project_dir)<CR> 

"cscope的相关查找快捷键设置
nmap <C-\>s :cs find s <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>g :cs find g <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>c :cs find c <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>t :cs find t <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>e :cs find e <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>f :cs find f <C-R>=expand("<cfile>")<CR><CR>
nmap <C-\>i :cs find i ^<C-R>=expand("<cfile>")<CR>$<CR>
nmap <C-\>d :cs find d <C-R>=expand("<cword>")<CR><CR>

```
