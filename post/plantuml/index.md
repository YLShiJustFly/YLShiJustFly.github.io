
## 传统画流程图的痛点
&nbsp;&emsp;&emsp;我们经常需要画流程图来表示代码逻辑或者基本框架等。但我们在绘画流程图的时候，经常会在对齐连接线这些和流程图表达的意义无关的环节上浪费大量时间。
而流程图这一表达方式本身天然适合代码描述，因此有人设计了plantuml这种软件语言来专门处理流程图逻辑，把图片渲染的工作交给graphviz来做。  
&nbsp;&emsp;&emsp;本文旨在基于neovim和浏览器（一般是chrome，实际上支持各种浏览器），使用plantuml描述语言，搭建使用代码画流程图，并实时预览的绘图环境。

## 具体流程
### 1、依赖的第三方工具：graphviz
### 2、vim配置plantuml实时预览用到的插件

#### a）packer.nvim安装方式：
- use {'tyru/open-browser.vim', ft = {'plantuml'}, event = 'BufEnter'}
- use {'sheerun/vim-polyglot', ft = {'plantuml', 'markdown'}}
- use {'weirongxu/plantuml-previewer.vim', ft = {'plantuml'}, event = 'BufEnter'}

#### b）vim-plug安装方式：
- Plug 'sheerun/vim-polyglot
- Plug 'tyru/open-browser.vim'
- Plug 'weirongxu/plantuml-previewer.vim'

&nbsp;&emsp;&emsp;其中，vim-polyglot用来做语法高亮，plantuml-previewer.vim调用graphviz进行图片的渲染，open-browser.vim用来通过浏览器预览渲染出来的图。  
&nbsp;&emsp;&emsp;最近plantuml支持了不同theme进行图片的渲染，更多选择，总有一款适合你。

### 3、安装完插件后需要做的配置
1. 写一个bash脚本，文件名叫makeprg，内容如下：
```
#!/bin/bash
java -jar ~/.local/share/nvim/site/pack/packer/opt/plantuml-previewer.vim/lib/plantuml.jar -tsvg $@
```
2. makeprg放入系统`$PATH`路径下，供plantuml-previewer调用。

### 4、开始画我们第一个plantuml流程图
1. 先看已经安装了哪些theme  
  
&emsp;&emsp;a） nvim新建一个puml文件
```
nivm theme.puml
```
&emsp;&emsp;b） 输入如下内容：
```
@startuml
help themes
@enduml
```
&emsp;&emsp;c） 命令模式输入:PlantumlOpen，得到已经安装的theme列表，后续要选择其中一种。  
![theme](http://youseeicanfly.gitee.io/picturebed/plantuml/theme.png)

2. 写我们的第一个流程图代码，指定要用的theme为bluegray。   
```
@startuml
!theme bluegray
start
if ("你想练葵花宝典") then (true)
floating note right:修炼葵花宝典注意事项
    if ("你能接受自宫吗") then (no)
        :"很遗憾，你无法修炼";
    else (yes)
        if ("你有葵花宝典秘籍吗") then (yes)
            :"恭喜你，可以修炼";
        else (no)
            :"很遗憾，你无法修炼";
        endif
    endif
else (false)
    :"不修炼";
endif
stop
@enduml
```
3. 执行:PlantumlOpen生成流程图。  
![test](http://youseeicanfly.gitee.io/picturebed/plantuml/test_bluegray.png)

3. 换成sandstone主题。  
```
@startuml
!theme sandstone
start
if ("你想练葵花宝典") then (yes)
floating note right:修炼葵花宝典注意事项
    if ("你能接受自宫吗") then (no)
        :"很遗憾，你无法修炼";
    else (yes)
        if ("你有葵花宝典秘籍吗") then (yes)
            :"恭喜你，可以修炼";
        else (no)
            :"很遗憾，你无法修炼";
        endif
    endif
else (no)
    :"不修炼";
endif
stop
@enduml
```
![test](http://youseeicanfly.gitee.io/picturebed/plantuml/test_sandstone.png)
