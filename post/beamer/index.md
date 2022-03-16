
# beamer和powerpoint的不同
    我们经常需要用ppt这一形式来展示我们的工作成果，但众所周知，微软的powerpoint是收费软件，且价格不菲，wps的画图功能能用，但需要保存成其他格式，比如pdf时。  
    是需要收费的。而基于latex的beamer宏包，我们可以使用编辑器写latex代码，用latex的编译工具编译成适合展示的ppt。当然ppt是pdf形式的，并且不太适合做  
    有复杂效果的ppt。
    本文旨在基于neovim和latex，使用latex的beamer宏包，搭建使用代码写ppt，并用skim实时预览的ppt制作环境，同时支持双向搜索跳转（即可以在vim中按快捷键跳转到  
    光标所在代码编译生成的ppt的对应页，同时支持在ppt中点击鼠标，定位到vim中对应代码）。

# 具体流程
## 1、依赖的第三方工具：开源的pdf阅读、编辑软件skim，latex编译及中文环境。
### a. 首先安装skim。
    这个软件支持brew方式安装，但现在的release版本有bug，正向跳转有问题，所以建议到官网下载补丁版。  
    安装好skim后，需要如下设置：  
    打开偏好设置->同步，选择自定义，命令写nvim，参数写
    ```
    --headless -c "VimtexInverseSearch %l '%f'" 
    ```
### b. 然后安装latex及中文环境。
    这个软件使用brew install mactex --cask命令安装即可，装好后在~目录下新建配置文件.latexmkrc，内容为   
 ```
$pdf_mode=5;
$xelatex = "xelatex -synctex=1 -interaction=nonstopmode -file-line-error %O %S";
 ```
    这个配置文件主要作用是指定编译使用xeletex，并支持持续编译模式。
    latex的beamer宏包有一些默认的主题，如果不满意，可以到网上下载第三方主题，拷贝到latex目录下，执行sudo texhash将主题等内容更新到latex库里。

## 2、vim配置ppt实时编译及预览用到的插件只有一个，但需要搭配snippets软件使用，方便插入大段固定代码。

### a.packer.nvim安装方式：
- use {'lervag/vimtex', event = 'BufEnter'}
- use {'honza/vim-snippets', event = 'BufEnter'}
- use {'SirVer/ultisnips', ft = {'tex'}, event = 'BufEnter'}

### b.vim-plug安装方式：
- Plug 'lervag/vimtex'
- Plug 'honza/vim-snippets'
- Plug 'SirVer/ultisnips'

## 3、安装完插件后需要做的配置
    注意：其他配置按vimtex github主页的说明即可，但要支持双向搜索，需要加如下设置：
    ```
        let g:vimtex_view_general_viewer = '/Applications/Skim.app/Contents/SharedSupport/displayline'
        let g:vimtex_view_general_options = '-g -r @line @pdf @tex'
        augroup vimtex_mac
            autocmd!
            autocmd User VimtexEventCompileSuccess call UpdateSkim()
        augroup END
        function! UpdateSkim() abort
            let l:out = b:vimtex.out()
            let l:src_file_path = expand('%:p')
            let l:cmd = [g:vimtex_view_general_viewer, '-r']
            if !empty(system('pgrep Skim'))
                call extend(l:cmd, ['-g'])
              endif
            call jobstart(l:cmd + [line('.'), l:out, l:src_file_path])
        endfunction
    ```
    这段代码调用了skim.app里的一个bash脚本，这个脚本实际调用了一段applescript脚本来支持定位到skim中的位置。
## 4、开始写我们的ppt
```
\documentclass[aspectratio=169,fontset=windows,UTF-8,10pt,xcolor={usenames,dvipsnames,svgnames,x11names}]{beamer}
\usepackage[space]{ctex}
\usetheme{cleancode}
\usetikzlibrary{shadows}
\usepackage{tikz}
\usepackage{bookmark}
\usepackage{diagbox}
\usepackage[absolute,overlay,showboxes]{textpos}
\usepackage{indentfirst} 

\begin{document}
%首页
{
    \usebackgroundtemplate{\tikz[overlay,remember picture]\node[opacity=1]at (current page.center){\includegraphics[height=\paperheight,width=\paperwidth]{"/Users/shiyaoliang/Documents/a.管控资料文件/c.ata/a.nvim/c.png/a.首页.png"}};}
    \begin{frame}[fragile]
        \begin{center}
        \Large 
            基于lsp和neovim的半隔离网络环境\\
                     c++开发环境搭建
            \\[4\baselineskip]
        \small
             钟吕---块存储
        \end{center}
    \end{frame}
}

%目录
\section[Contents]{}
\frame 
{
    \frametitle{\secname}
    %\tableofcontents[pausesections]
    \tableofcontents
}

\AtBeginSection[] 
{
    \frame<handout:0> 
    {
        \frametitle{Contents}
        \tableofcontents[current,currentsection]
    }
}

\usebackgroundtemplate{\tikz[overlay,remember picture]\node[anchor=45]at ([xshift=10ex,yshift=-5ex] current page.45) {\includegraphics[height=0.1\paperheight,width=0.2\paperwidth]{"/Users/shiyaoliang/Documents/a.管控资料文件/c.ata/a.nvim/c.png/c.角标.png"}};}
\section{lsp简介}
\begin{frame}[fragile]
    \frametitle{lsp简介}
    \setlength{\parindent}{2em}
    近两年基于language server protocol（后文简称lsp）的language server和language client都在持续发展，已经到了可以使用的程度。
    这不仅使vscode的功能更加强大，也造福了其他功能可扩展的编辑器。
    对于vim而言，受益于lsp，可以拥有和vscode完全一样的语法检测和定义跳转等的体验，并且更加轻量。
    \\[1\baselineskip]
    现在ata上关于lsp介绍的文章还很少，本文旨在基于vim的lsp，在开发机的半网络隔离环境下，搭建C++开发环境。
\end{frame}

%尾页
{
    \usebackgroundtemplate{\tikz[overlay,remember picture]\node[opacity=1]at (current page.center){\includegraphics[height=\paperheight,width=\paperwidth]{"/Users/shiyaoliang/Documents/a.管控资料文件/c.ata/a.nvim/c.png/b.尾页.png"}};}
    \begin{frame}
    \end{frame}
}
\end{document}
```
## 5、按vimtex默认快捷键\<leader\>ll，开启编译并实时预览
效果：
![test](http://youseeicanfly.gitee.io/picturebed/beamer/ppt_up.png)
![test](http://youseeicanfly.gitee.io/picturebed/beamer/ppt_content.png)
![test](http://youseeicanfly.gitee.io/picturebed/beamer/ppt_down.png)

## 6、双向搜索定位方法
- 正向定位：在vim中输入快捷键：\<leader\>lv，ppt中的位置会定位到vim中光标所在位置表示的页数。
- 反向定位：在skim中按住shift+command，并切鼠标点击ppt内容部分，vim中的光标跳转到该页ppt对应的代码位置。
