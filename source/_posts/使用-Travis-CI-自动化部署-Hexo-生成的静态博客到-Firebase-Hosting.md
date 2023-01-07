---
title: 使用 Travis CI 自动化部署 Hexo 生成的静态博客到 Firebase Hosting
date: 2018-03-08
typora-root-url: ../../source
tags:
- CI
---
先说下前因吧，原本博客是托管在 Github Pages 上的，但是一直觉得速度不理想、打开得比较慢，但是又找不到合适的替代品故放任不理。无意中发现 Firebase 提供了静态资源托管服务，而且居然是国内可用的，经过测试速度很可观（反正感觉是比 Github Pages 要快）。于是就开始着手博客迁移的工作，本文便是这次折腾经过的总结。
在正式开始之前，先介绍一遍涉及到的东西：
0. Hexo：静态博客生成工具
0. Github：托管博客源码
0. Travis CI：持续集成工具
0. Firebase Hosting：静态资源托管服务

# 搭建博客
市面上有很多搭建博客的工具，不过我独爱 Hexo Next 主题，故选择 Hexo 作为搭建工具。Hexo 主要是通过预设的主题样式和编写好的 MarkDown 生成静态的网站。而 Github 也提供了静态网站的托管服务，所以也造就了 Hexo 这类生成静态博客的工具有众多的使用人数。
安装 Hexo 步骤很简单，官方也提供了中文文档，需要注意的是 Hexo 依赖 Git 和 Node.js。核心步骤：
``` bash
# 安装 hexo
npm install hexo-cli -g

# 初始化名为 blog 的文件夹用于存放生成静态网站用的源码
hexo init blog

# 使用 npm 自动安装依赖
cd blog
npm install

# 安装本地服务器，用于在本地预览博客效果
npm install hexo-server --save

# 启动本地服务器，即可在浏览器中输入 http://localhost:4000 预览效果
hexo server
```

# 博客网站源码托管
因为博客网站是通过上文中 **blog** 文件夹中的文件生成的，意味着只要源码在，随时随地都能重新生成一个一摸一样的静态网站并可以重新部署到任意的地方，所以需要好好保管这些文件。和代码一样，这里选择使用 Git 管理代码版本，方便进行回滚等一系列操作，并选择 Github 作为远程仓库。通过 Git 将源码保存到 Github 中操作简单，并且不是本文重点故略过。

# 自动化部署
在之前把博客托管在 Github Pages 时并没有使用 Travis CI 来完成自动化部署任务，因为从本地把生成好的博客发布到 Github Pages 中十分的方便，只需要一行命令即可搞定。但是使用 Firebase Hosting 就麻烦了许多，需要使用 Firebase CLI 来完成发布，而且使用 Firebase CLI 是需要科学上网的，对于在国内来说非常的不方便，所以不如直接通过 CI 来完成发布要方便许多。选择 Travis CI 的主要原因是用于 Github 中的开源仓库是免费使用的。
首先在 [Travis CI](https://travis-ci.org/) 中使用托管了源码的 Github 账号登陆，勾选对应仓库名左边的选择框并开启服务，如图所示：

![](/images/20180308231147.png)

然后就是需要在源码文件夹的根目录（上文中也就是 blog 文件夹）添加 **.travis.yml** 文件，该文件中主要描述 CI 的任务。示例如下：
``` yml
# 执行命令需要 sudo 命令
sudo: required

# Travis CI 预置了多种常用语言的配置，这里 Hexo 需要使用的是 Node.js
language: node_js

# 设置需要安装的 Node.js 版本号
node_js: "8"

# 执行命令生成静态博客网站
script:
  - hexo g

# 接下来是部署到 Firebase Hosting 的配置信息
deploy:
  # 默认在部署前会把整个目录恢复到刚拷贝时的状态，但是我们需要上面生成好的文件，故跳过该阶段
  skip_cleanup: true

  # Travis CI 预置了发布到 firebase 的配置
  provider: firebase
  
  # 配置 Firebase 的用户验证和工程信息，下文会提到
  token:
    secure: $FIREBASE_TOKEN
  project: $FIREBASE_PROJECT
```
有了 .travis.yml 文件后，我们只需要通过 
``` bash
git push
```
命令把代码推送到 Github 上，Travis CI 便会检测到有变更且自动执行上面的任务。现在我们还差的就是把上面用的环境变量  `FIREBASE_TOKEN` 和 `FIREBASE_PROJECT` 补充上。环境变量在 Travis CI 控制台中对应项目的设置页面中添加，添加后如图所示：
![](/images/20180308235926.png)

而他们的值通过一下步骤获取：
先到 [Firebase](firebase.google.com) 申请一个工程，环境变量 FIREBASE_PROJECT 的值便是 **项目 ID**。
接下来需要在能科学上网的地方安装 Firebase CLI：
``` bash
npm install -g firebase-tools
```
通过 `firebase login:ci` 命令登陆，成功登陆后便可获得环境变量 FIREBASE_TOKEN 的值。如果不是在本机登陆（例如通过 SSH 在远程服务器上进行操作的），可以通过`firebase login:ci --no-localhost` 命令完成登陆。填写了环境变量值之后，还需要在源码根目录下添加 **firebase.json** 文件配置基本的发布信息。示例：

``` json
{
  "hosting": {
    "public": "public",
  }
}
```
**hosting** 代表我们使用的是 Hosting 服务，子节点 **public** 指定了需要托管的静态资源的目录，因为 Hexo 生成的文件默认在 public 文件夹中，故指定为该文件夹。
现在万事俱备了，只需要我们轻轻敲动键盘，把源码推送到 Github 上便大功告成了。待 CI 任务执行完成，便可以在 Firebase Hosting 指定的域名上访问我们的博客了。 