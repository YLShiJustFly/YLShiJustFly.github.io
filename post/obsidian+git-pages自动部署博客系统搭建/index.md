
&emsp;&emsp;&nbsp;我之前使用 neovim 写 markdown，hugo 管理博客静态站点，并写了一个编译 hugo，并发布到 github pages 的脚本。   
&emsp;&emsp;&nbsp;最近开始迁到 obsidan 做笔记，完备的插件生态，颇有俯拾即是、左右逢源之感。  
&emsp;&emsp;&nbsp;于是就想着编辑 markdown 都用 obsidian，故研究了一下使用 obsidian+git 插件+github Actions 替代原有工作流，在 obsidian 内完成从编写博文到部署到 github pages 的所有流程。  
&emsp;&emsp;下面是配置流程：

1. github 侧的配置

	1. 需要两个 repo，一个对应 hugo 项目目录，一个是 github pages 的 repo。
![image.png](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/20230302104302.png)
	2. 在 hugo 项目目录里，设置 Actions 的 workflow
		1. 新建 workflow
![image.png](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/20230302104620.png)
		2. 搜索 hugo
![image.png](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/20230302104929.png)
		3. 点击configure，配置hugo，并在.github/workflow目录下生成hugo.yml配置文件。

![image.png](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/20230302105105.png)
        内容如下：

        ```
        name: pages build and deploy
        
        on:
        
        workflow_dispatch:
        
        push:
        
        branches:
        
        - main
        
        jobs:
        
        build-and-deploy:
        
        runs-on: ubuntu-latest
        
        steps:
        
        - uses: actions/checkout@v3
        
        with:
        
        submodules: recursive
        
        - name: Setup Hugo
        
        uses: peaceiris/actions-hugo@v2
        
        with:
        
        hugo-version: '0.100.2'
        
        extended: true
        
        - name: Build with Hugo
        
        env:
        
        HUGO_ENVIRONMENT: production
        
        HUGO_ENV: production
        
        run: hugo --theme=even --baseUrl="http://YLShiJustFly.github.io/"
        
        - name: Deploy
        
        uses: peaceiris/actions-gh-pages@v3
        
        with:
        
        PUBLISH_BRANCH: main
        
        EXTERNAL_REPOSITORY: YLShiJustFly/YLShiJustFly.github.io
        
        PERSONAL_TOKEN: ${{ secrets.PERSONAL_TOKEN }}
        
        PUBLISH_DIR: ./public
        
        commit_message: ${{ github.event.head_commit.message }}
        ```
        
&emsp;&emsp;其中，`YLShiJustFly/YLShiJustFly.github.io` 为需要托管的 git pages 的 git repo 地址。`PERSONAL_TOKEN` 在 GitHub 账户下 `Setting -> Developer setting -> Personal access tokens` 下创建。注意生成的 `token` 只会显示一次，可以自己保存下来。  
![image.png](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/20230302110308.png)

2. obsidian 侧的配置

	1. 图床
		1. 新建一个 picturebed repo，`Setting -> General -> Danger Zone -> visibility`需要设置成 public
         ![image.png](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/20230302110859.png)
		2. 本机安装 picgo 命令行工具。
        ```
        brew install picgo
        ```
		3. 执行 picgo set upload，选择 github，并依次配置 picturebed 相关内容。
      ![image.png](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/20230302111159.png)
      ![image.png](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/20230302111938.png)
		4. 执行 picgo use upload，选择 github
      ![image.png](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/20230302112801.png)

	2. 安装 git 插件
   &emsp;&emsp;&nbsp;插件市场搜索 Obsidian Git，安装即可。

3. 使用
&emsp;&emsp;&nbsp;在Obsidian编译好文章，输入cmd+p呼出命令窗口，输入git，可以看到Obsidian支持的git命令。
![image.png](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/20230302113431.png)
&emsp;&emsp;&nbsp;依次执行git pull，git commit，git push，即可把博文推送至hugo仓库。Actions会自动编译并发布到git pages。
![image.png](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/20230302113750.png)
