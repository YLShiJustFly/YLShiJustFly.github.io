
## vim配置plantuml实现流程图实时绘制并预览
### 1、依赖的第三方工具：graphviz
### 2、vim配置plantuml实时预览用到的插件

#### packer.nvim安装方式：
- use {'tyru/open-browser.vim', ft = {'plantuml'}, event = 'BufEnter'}
- use {'sheerun/vim-polyglot', ft = {'plantuml', 'markdown'}}
- use {'weirongxu/plantuml-previewer.vim', ft = {'plantuml'}, event = 'BufEnter'}

#### vim-plug安装方式：
- Plug 'sheerun/vim-polyglot
- Plug 'tyru/open-browser.vim'
- Plug 'weirongxu/plantuml-previewer.vim'

其中，vim-polyglot用来做语法高亮，plantuml-previewer.vim调用graphviz进行图片的渲染，open-browser.vim用来通过浏览器预览渲染出来的图。  
最近plantuml支持了不同theme进行图片的渲染，更多选择，总有一款适合你。

### 3、安装完后需要做的配置
1) 写一个bash脚本，文件名叫makeprg，内容如下：
```
#!/bin/bash
java -jar ~/.local/share/nvim/site/pack/packer/opt/plantuml-previewer.vim/lib/plantuml.jar -tsvg $@
```
2) makeprg放入系统`$PATH`路径下，供plantuml-previewer调用。

### 4、开始画我们第一个plantuml流程图
#### 先看已经安装了哪些theme
1) nvim新建一个puml文件
nivm theme.puml

2) 输入如下内容：
```
@startuml
help themes
@enduml
```
3) 命令模式输入:PlantumlOpen，得到已经安装的theme列表，后续要选择其中一种。  
![theme](http://youseeicanfly.gitee.io/picturebed/plantuml/theme.png)

4) 写我们的第一个流程图代码，指定要用的theme为bluegray。   
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
5) 执行:PlantumlOpen生成流程图。  
![test](http://youseeicanfly.gitee.io/picturebed/plantuml/test_bluegray.png)

6) 换成sandstone主题。  
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
