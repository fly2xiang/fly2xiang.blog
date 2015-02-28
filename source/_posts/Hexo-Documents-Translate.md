title: "Hexo 文档翻译完成"
date: 2015-02-27 17:54:57
tags:
- Hexo
- 博客
- Github
categories: 
---

## 引言

Hexo 文档终于翻译完成了，链接: [Hexo 中文网](http://www.liuzhixiang.com/hexo_site_cn/)

## 什么是 Hexo?

Hexo 是一个快速，简单，强大的博客框架。 它使用 [Markdown](http://daringfireball.net/projects/markdown/)  (或者其他的渲染引擎)转换你的文章并结合一个漂亮的主题生成静态文件，而这一切都在数秒之内完成。

## 安装

准备好 Hexo 只需要花费几分钟时间。如果你在安装过程中遇到问题并且在这个文档中不能找到解决方法, 请 [提交问题](https://github.com/hexojs/hexo/issues) ，我会试着解决你的问题。

### 准备

安装 Hexo 非常简单。在开始安装之前需要先安装以下组件：

- [Node.js](http://nodejs.org/)
- [Git](http://git-scm.com/)

如果你的电脑已经安装了它们，恭喜！你只需要使用 npm 安装 Hexo 即可。

``` bash
$ npm install -g hexo
```

如果还没有安装需要的组件，请先安装它们。

### 安装 Git

- Windows: 下载并安装 [msysgit](http://code.google.com/p/msysgit/).
- Mac: 安装 [Homebrew](http://mxcl.github.com/homebrew/), [MacPorts](http://www.macports.org/) or [installer](http://code.google.com/p/git-osx-installer/).
- Linux (Ubuntu, Debian): `sudo apt-get install git-core`
- Linux (Fedora, Red Hat, CentOS): `sudo yum install git-core`

### 安装 Node.js

安装 Node.js 最好的方式是通过 [nvm](https://github.com/creationix/nvm) 安装。

cURL:

``` bash
$ curl https://raw.github.com/creationix/nvm/master/install.sh | sh
```

Wget:

``` bash
$ wget -qO- https://raw.github.com/creationix/nvm/master/install.sh | sh
```

安装后，重启终端并且运行下面的命令安装 Node.js。

``` bash
$ nvm install 0.10
```

或者你可以直接下载 [安装包](http://nodejs.org/) 安装它.

### 安装 Hexo

以上组件安装好后，你可以通过 npm 安装 Hexo。

``` bash
$ npm install -g hexo
```
