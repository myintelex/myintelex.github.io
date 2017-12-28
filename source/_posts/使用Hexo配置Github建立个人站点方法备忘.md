---
title: 使用Hexo配置Github建立个人站点方法备忘
date: 2017-12-25 16:21:25
categories: 开发经验
tags: [Hexo, Blog, Github]
---
使用 Hexo 来建立自己的个人博客已经一年多了，虽说没有什么人来看，但是作为自己的一个知识整理库。还是有一定的效果的，但是当有时候更换电脑重装环境后，总是需要重新去查看如何安装 Hexo 本地环境。因此，还是在这里自己记录一下。
<!-- more -->

## 一、安装
安装 Hexo 之前，必须先安装 Node 环境。同时，因为我们的博客是托管在 Github 上的，需要安装 Git 工具，申请 Github 账号，并建立对应的 Rep。
### 1、安装 Node
首先，点击进入 [`node.js`](https://nodejs.org/en/)官网。

选择想要的版本，下载。

随后一路按照正常的安装方法安装即可。

### 2、安装 Git
同样，点击进入 [`Git`](https://git-scm.com/downloads)官网。

选择想要的版本，下载。

随后一路按照正常的安装方法安装即可。

### 3、GitHub 注册和建立 Rep 
注册这一点就不再详细描述，进入 [`GitHub`](https://github.com) 官网注册即可。

注册完成后，建立与用户名对应的仓库，譬如我的即为 `myintelex.github.io`

### 4、安装 Hexo
进入主角环节，创建书写 Blog 的文件夹，进入文件夹后，执行以下命令：
````
npm install -g hexo
````
即可安装 Hexo。

随后执行命令进行初始化：
````
hexo init
````
这时候 Hexo 就已经安装好了。

### 5、配置测试
首先，需要与远程仓库绑定，打开博客目录下的 `_config.yml` 文件，将下列内容
````
deploy:

     type: git

     repo: https://github.com/myintelex/myintelex.github.io.git

     branch: master
````
中的 `repo` 更改为自己的 repo 地址。

这时候可以执行`hexo g`命令，生成静态页面。

完成后，执行 `hexo server` 启动本地服务。

浏览器打开 [http://localhost:4000](http://localhost:4000)，即可测试本地效果。


## 二、Hexo 常用命令：
|命令|含义|
|--|--|
|hexo new"postName" |#新建文章
|hexo new page"pageName" |#新建页面
|hexo generate |#生成静态页面至 public 目录
|hexo server |#开启预览访问端口（默认端口 4000，'ctrl + c'关闭 server）
|hexo deploy |#将. deploy 目录部署到 GitHub
|hexo help |# 查看帮助
|hexo version |#查看 Hexo 的版本

## 三、报错总结（此项持续更新）

* `ERROR Deployer not found: git 或者 ERROR Deployer not found: github`

  解决方法： `npm install hexo-deployer-git --save`     
  如发生报错： `ERROR Process failed: layout/.DS_Store `, 那么进入主题里面 `layout` 和 `_partial` 目录下，删除 `.DS_Store` 

* `ERROR Plugin load failed: hexo-server`

  解决方法，执行命令：`sudo npm install hexo-server`
