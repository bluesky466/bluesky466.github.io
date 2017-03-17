title: 使用hexo搭建个人博客
date: 2016-01-19 21:10:24
tags: 
    - 技术相关
    - hexo
---

## 一、安装hexo

**1.安装Node.js**
hexo是一款基于Node.js的静态博客框架，所以要使用它必须先安装Node.js。网上很多教程讲的都是如何编译源码安装Node.js。但我认为应该用最简单的方式取获取和使用软件或者框架，不是每个初学者都需要从一个软件的源代码编译开始学习。当然，在日后想要更深入理解它的时候，挖出它的源代码进行分析也是一种十分有效的途径。
下面是ubuntu下使用apt-get安装Node.js的方法（源自博客[如何在Ubuntu上安装最新版本的Node.js ](http://blog.csdn.net/chszs/article/details/37521463)）
1. apt-get update
2. apt-get install -y python-software-properties software-properties-common
3. add-apt-repository ppa:chris-lea/node.js
4. apt-get update
5. apt-get install nodejs

**2.安装hexo**
之后安装hexo就更简单了，只需要这一行代码

    npm install hexo -g

**3.更新hexo**

    npm update hexo -g 
    
<br>

## 二、创建和配置hexo项目

**1.初始化hexo项目**

    hexo init [folder]

如果指定 _folder_，便会在目前的资料夹建立一个名为 _folder_ 的新资料夹，否则会在当前文件夹初始化。
执行完这条命令，会出现如下提示:

> INFO  You are almost done! Don't forget to run 'npm install' before you start blogging with Hexo!

所以记得执行npm install


**2.创建新的文章**

    hexo new "文章标题"

执行完创建命令后会生成以下文件:

> source/_posts/文章标题.md

之后只需要在这个markdown文件里面编写自己的博客文章就可以了

**3.添加主题**

有很多人为hexo编写了很多漂亮的主题，可以自己去[主题列表](https://github.com/hexojs/hexo/wiki/Themes)选择
安装的方法也很简单，这些主题都是托管在github上的，只要把它们克隆到项目文件夹的themes目录下面就可以了。当然还需要去__config.yml修改配置，选择使用哪个主题：

    theme: 主题名

**4.启动服务器**

编写完文章之后只需要运行下面命令就可以在浏览器地址栏输入 http://0.0.0.0:4000 查看自己的博客了

    hexo server

<br>

## 三、配置博客信息

可以在项目根目录下的_config.yml文件配置博客的标题，作者，语言等相关信息

<br>

## 四、部署到Github

github提供了一个名叫Github Pages的服务，我们可以免费的用它来搭建自己的博客。
做法很简单，首先在github建立与你用户名对应的仓库，仓库名必须为 “你的github用户名.github.io” 

接着执行以下命令在本机安装hexo-deployer-git

    npm install hexo-deployer-git --save

然后在_config.yml文件，找到下面的内容

    # Deployment
    ## Docs: http://hexo.io/docs/deployment.html
    deploy:
      type:

将它们修改为

    # Deployment
    ## Docs: http://hexo.io/docs/deployment.html
    deploy:
      type: git
      repository: git@github.com:你的github用户名/你的github用户名.github.io
      branch: master

最后执行以下三条命令即可：

    hexo clean
    hexo generate
    hexo deploy

（当然你必须为你的电脑添加github ssh key才能正常执行hexo deploy命令上传代码）

<br>

## 五、绑定域名

按照github pages的文档，是在github项目根目录下创建CNAME文件。但因为每次使用hexo更新博客再次上传，都会清除掉之前创建的CNAME文件。所以我们把CNAME放在source目录下（想上传的文件都放在该目录下）

完整步骤如下:

1. 在source目录下添加一个CNAM文件，没有后缀名，里面内容为你的域名(如我的是:blog.islinjw.cn)。

2. ping username.github.io记录下IP地址

3. 购买域名，配置域名解析username.github.io的ip地址
