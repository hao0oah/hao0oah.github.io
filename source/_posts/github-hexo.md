---
title: Github+Hexo博客搭建过程
date: 2017-11-08 16:53:12
tags:
    - 博客
    - Hexo
categories: 博客
---

> 本文是基于Windows平台，以最简单的步骤，展示我自己搭建的过程。

#### 1. 准备工作

- [注册](https://github.com/join?source=header-home)一个Github账户
- [安装](https://nodejs.org/en/)Node.js，[Node.js教程](http://nodejs.jakeyu.top/index.html)
- [安装](https://github.com/waylau/git-for-win)Git，[Git教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)

<!-- more -->

#### 2. 创建Github仓库

登录你的GitHub账户，新建一个名为`username.github.io`的仓库，**username**就是你的Github用户名，之后就可以通过`username.github.io`来访问博客了。

#### 3. 配置SSH Key

我们在本地写博客、编写代码，最终要提交到服务器上去，怎样建立一种本地与服务器的连接呢？可以使用github用户名和密码，但这有些繁琐且不安全，所以用到SSH Key来建立加密连接。

- 首先检查本机有没有SSH Key，使用Git Bash命令行窗口输入：
  `cd ~/.ssh`
  若提示：`No such file or directory`则说明本机还没有SSH Key；

- 继续输入生成SSH Key的命令：
  `ssh-keygen -t rsa -C "邮件地址"`（邮件地址为你的github邮箱）
  然后连续回车3次来生成SSH Key，生成的文件在用户目录下（用户目录在git窗口中有提示），找到`.ssh\id_rsa.pub`文件后，用记事本打开并复制里面的所有内容，接着打开你的github主页，进入Setting -> SSH and GPG keys -> New SSH key，粘贴到Key内容框中，Title框中可写明该SSH Key的用处，保存即可；

- 验证是否有效：
  `ssh -T git@github.com`，回车，出现”Are you sure you want to continue connecting (yes/no)?”后，输入yes，当你看到”Hi ‘username’! You’ve successfully authenticated, but GitHub does not provide shell access.”时，说明你的SSH已经配置成功；（username为你的用户名）

- 设置git全局变量
  `git config --global user.name "github用户名"`
  `git config --global user.email "github邮箱"`

#### 4. 初始化博客框架

[Hexo官方文档](https://hexo.io/zh-cn/docs/)

- 可以使用cmd或者Git Bash命令行窗口，来安装Hexo：
  `npm install -g hexo-cli`

- 安装好之后，切换到你希望本地博客存放的目录，开始初始化博客框架：
  `hexo init <yourFolder> #如果yourFolder不存在，执行后会自动生成目录`
  `hexo init #如果yourFolder存在，切换到该目录下执行该命令`

- 安装依赖包
  `npm install`

- 查看生成好的博客结构
  `cd <yourFolder>`
  
  ```
   ├── _config.yml //网站的配置信息，可以在此配置大部分的参数。
   ├── package.json
   ├── scaffolds //模版文件夹。当新建文章时，Hexo会根据scaffold来建立文件。
   ├── source //资源文件夹是存放用户资源的地方。
   | ├── _drafts
   | └── _posts
   └── themes //主题文件夹。Hexo会根据主题来生成静态页面。
  ```

#### 5. 配置博客

在博客主目录下，打开`_config.yml`文件，修改参数信息

- 网站基本信息
  
  ```
  title: 主标题
  subtitle: 副标题
  description: 网页描述
  author: 作者名
  language: zh-CN
  timezone: Asia/Shanghai
  ```

- 部署到Github的配置
  
  ```
  deploy: 
  type: git
  repo: https://github.com/zhihuya/username.github.io.git
  branch: master
  ```

- 预览本地博客
  `hexo generate`
  `hexo server`
  在浏览器输入`http://localhost:4000`就可以看见默认的博客了。

- 部署到Github代码库
  `hexo deploy`

- 如果部署出现错误，执行以下命令：
  `npm install hexo-deployer-git --save`

#### 6. 常用的Hexo命令

```
hexo help #查看帮助
hexo init #初始化一个目录
hexo new "postName" #新建文章
hexo new page "pageName" #新建页面
hexo generate #生成网页，可以在 public 目录查看整个网站的文件
hexo server #本地预览，'Ctrl+C'关闭
hexo deploy #部署.deploy目录到Github上
hexo clean #清除缓存，建议每次执行命令前先清理缓存
```

#### 7. 使用Next主题

[Next主题官网](http://theme-next.iissnan.com/)

- 下载主题包
  `cd <yourFolder> #切换到本地博客主目录`
  `git clone https://github.com/iissnan/hexo-theme-next themes/next`

- 启用Next主题
  打开主目录下的`_config.yml`文件，找到theme字段，并将其值更改为next即可。

###### [参考1](http://blog.liuxianan.com/build-blog-website-by-hexo-github.html)，[参考2](https://getwang.github.io/2017/09/03/%E4%BD%BF%E7%94%A8github-hexo%E6%90%AD%E5%BB%BA%E5%85%8D%E8%B4%B9%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/)，[参考3](https://zhangslob.github.io/2017/02/28/%E6%95%99%E4%BD%A0%E5%85%8D%E8%B4%B9%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2%EF%BC%8CHexo-Github/)，[参考4](http://www.jianshu.com/p/701b1095da11)，[参考5](http://www.jianshu.com/p/61987cec0fad#)，[参考6](http://www.jianshu.com/p/380290deb8f0)
