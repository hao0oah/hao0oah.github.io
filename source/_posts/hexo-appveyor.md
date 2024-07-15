---
title: Hexo博客源文件备份
date: 2017-11-14 00:32:09
tags:
    - 博客
    - AppVeyor
    - Hexo
categories:
    - 博客
---

### 前言

关于备份Hexo博客源文件，搜索了一遍网上的资料，我感觉比较有意思的有两种方式。一种是将Github上托管的博客版本库创建两个分支，缺点是操作比较繁琐，对于Git新手具有一定的挑战，一不小心可能会搞错；一种是另外创建一个存放博客源文件的Github代码库，利用AppVeyor进行持续集成和自动化部署，缺点是需要创建两个代码库，依赖一个新的东西AppVeyor。

既然想折腾，那就想点更好玩的吧。借鉴上面两种方式合起来操作，实现只用一个代码库，并使用AppVeyor进行自动化部署。

<!-- more -->

我想要的效果的实现原理是，博客源文件放在远程hexo分支，博客资源放在远程master分支，Github Pages默认以master分支的文件来呈现博客网页。在本地修改完博客之后，push到hexo分支，触发AppVeyor执行，配置AppVeyor从hexo分支拉取代码，执行`hexo generate`后将在public文件夹下生成博客网页，再从master分支拉取完整的博客版本库存放到`.depoly_git`文件夹下，执行`hexo deploy`将自动部署最新的博客网页到Github代码库的master分支上。

开始下面操作之前，假设你已经搭建好本地Hexo博客环境，并已经将博客提交到Github Pages上，且可以正常访问到你的博客。否则请先操作这篇[快速搭建Hexo博客](https://hao0oah.github.io/2017/11/08/github-hexo/)。

### 使用AppVeyor配置自动部署功能

#### 注册并登陆AppVeyor

访问[AppVeyor登陆](https://ci.appveyor.com/login)页面，使用你的GitHub账号登陆即可。
![appveyor-login.png](https://s2.loli.net/2024/07/07/ipgefwqPJxdXNbs.png)

#### 添加Project

在`https://ci.appveyor.com/projects/new`页面，添加相应的GitHub Repo，格式应为`xxx.github.io`。

#### 在该项目的settings中设置General

将`Branches to build`选项改为`Only branches specified below`，然后点击`Add branch`填入hexo，点击最底下的Save按钮保存。该设置是让AppVeyor从代码库的hexo分支拉取代码进行编译。
![appveyor_setting_general.jpg](https://s2.loli.net/2024/07/06/DkpcnltmPr7UFuB.jpg)

#### 在该项目的settings中设置Envirommemt

添加以下四个变量

<style>
table th:first-of-type {
    width: 150px;
}
</style>

| KEY              | VALUE                                          |
|:----------------:|:----------------------------------------------:|
| STATIC_SITE_REPO | 格式应为`https://github.com/xxx/xxx.github.io.git` |
| TARGET_BRANCH    | 博客网页文件所在分支，使用默认的master                         |
| GIT_USER_EMAIL   | 注册GitHub的邮箱                                    |
| GIT_USER_NAME    | 注册GitHub的用户名                                   |


#### 添加appveyor.yml配置文件

接下来，你需要在本地博客的根目录下创建appveyor.yml，具体内容如下：

```
clone_depth: 5
environment:
  nodejs_version: 6
  access_token:
    secure: [Your GitHub Access Token]

install:
  # 解决可能会遇到的模块安装失败的问题
  - ps: Install-Product node $env:nodejs_version
  - node --version
  - npm --version
  - npm install
  - npm install hexo-cli -g
  # 安装hexo deploy工具
  - npm install hexo-deployer-git --save
  # 解决偶尔出现"Error: Cannot find module 'hexo-util'"的问题
  - npm install hexo-util --save-dev 

build_script:
  - hexo generate

on_success:
  - git config --global credential.helper store
  - ps: Add-Content "$env:USERPROFILE\.git-credentials" "https://$($env:access_token):x-oauth-basic@github.com`n"
  - git config --global user.email "%GIT_USER_EMAIL%"
  - git config --global user.name "%GIT_USER_NAME%"

  - mkdir %APPVEYOR_BUILD_FOLDER%\.deploy_git
  # 从Github上拉取代码库版本，可以保留提交记录；如果直接部署相当于一个新的代码库提交，覆盖了之前的提交版本记录
  - git clone --depth 5 -q --branch=%TARGET_BRANCH% %STATIC_SITE_REPO% %APPVEYOR_BUILD_FOLDER%\.deploy_git
  - hexo deploy
```

你需要做的就是替换[Your GitHub Access Token]，关于生成Access Token，可以参考这篇[文章](https://help.github.com/articles/creating-an-access-token-for-command-line-use/)。在GitHub生成好Access Token之后，你需要到[AppVeyor加密页面](https://ci.appveyor.com/tools/encrypt)把Access Token加密之后再替换[Your GitHub Access Token]
![appveyor-encrypt.png](https://s2.loli.net/2024/07/07/wSINErqWtmonM7O.png)

#### 设置Github Webhooks

设置完上面的步骤还不能实现自动部署，需要将Github的push事件通知给AppVeyor，这就需要配置Webhooks了。
还是在AppVeyor网站中，在该项目的settings中的General页面，找到`Webhook URL`这一项，复制其中的URL。

然后进入Github的博客代码库页面，切换到Settings页，点击左边的Webhooks选项，点击`Add Webhooks`按钮。
![add_webhooks](https://s2.loli.net/2024/07/06/pde4S5jCfTruGa8.jpg)

将复制的URL粘贴到`Payload URL`中，其他保持默认，点击`Add Webhooks`按钮保存即可。至此关于AppVeyor的相关配置全部完成。
![add_webhooks_page](https://s2.loli.net/2024/07/06/f6hWGuCLEIV1k3Q.jpg)

### 代码库分支操作

如果使用了其他主题，例如NexT，需要先删除`themes/next`文件夹下的.git目录，否则会出现上传到代码库的next文件夹为空，导致生成的都是空页面。

1. 初始化本地代码库
   `git init`

2. 创建分支hexo
   `git checkout -b hexo`
   为了方便管理和操作，在本地只创建了hexo分支

3. 添加和提交代码
   `git add .`,`git commit -m "xxx"`

4. 添加远程库
   `git remote add origin https://github.com/xxx/xxx.github.io.git`

5. 提交代码到远程hexo分支
   `git push -u origin hexo`
   该命令会在Github代码库创建hexo分支，并提交代码。`-u`的作用是记住提交到远程仓库的地址和分支，下次提交直接用`git push`就可以了。
- 如果你本地不是hexo分支，而是其他分支，如master分支，则需要执行
  `git push -u origin master:hexo`

- 如果你在远程代码库先创建了hexo分支(从master分支merge过来的)，由于版本不同步，需要强制提交
  `git push -f origin hexo`

- 如果想删除远程分支hexo
  `git push origin :hexo`

### 最终效果

push完之后，此时你的Github代码库应该就有两个分支，一个默认的master分支，用来发布博客静态网页，支撑着`https://xxx.github.io`的页面访问；一个是hexo分支，用来存放博客的源文件。
而push事件会通知AppVeyor执行编译和部署，几分钟后会更新完master分支的代码，并有邮件通知执行结果。

以后修改完博客源文件之后，只需要依次执行`git add .`,`git commit -m "xxx"`,`git push`命令即可。

在其他电脑上编写博客时，需要先从hexo分支拉取博客源文件
`git clone -b hexo origin https://github.com/xxx/xxx.github.io.git`
在该目录配置好博客环境
`npm install -g hexo-cli`
`npm install`
`npm install hexo-deployer-git`

#### 参考

[Hexo的版本控制与持续集成](https://formulahendry.github.io/2016/12/04/hexo-ci/)
[hexo+github+AppVeyor实现不同电脑写博客](https://killerlei.github.io/2017/04/06/hexo-github-AppVeyor%E5%AE%9E%E7%8E%B0%E4%B8%8D%E5%90%8C%E7%94%B5%E8%84%91%E5%86%99%E5%8D%9A%E5%AE%A2/)
[GitHub Pages + Hexo搭建博客](http://crazymilk.github.io/2015/12/28/GitHub-Pages-Hexo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2/#more)
[用 AppVeyor 持续集成 Github 中的 JS 项目](https://sebastianblade.com/using-appveyor-continuous-integration-in-javascript-project/)
[利用 AppVeyor 实现 GitHub 托管项目的自动化集成](http://www.gulu-dev.com/post/2015-05-01-appveyor-ci)
