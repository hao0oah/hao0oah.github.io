---
title: Hexo博客源文件备份
date: 2017-11-14 00:32:09
tags:
    - 博客
    - AppVeyor
    - Hexo
categories: 博客
---

关于备份Hexo博客源文件，搜索了一遍网上的资料，我感觉比较有意思的有两种方式。一种是将Github上托管的博客版本库创建两个分支，缺点是操作比较繁琐，对于Git新手简直是种精神折磨，一不小心可能会搞错；一种是另外创建一个存放博客源文件的Github代码库，利用AppVeyor进行持续集成和自动化部署，缺点是需要创建两个代码库，依赖一个新的东西AppVeyor。
既然想折腾，那就想点更好玩的喽。借鉴上面两种方式，可以合起来操作，实现只用一个代码库，并使用AppVeyor进行自动化部署。

#### 参考
[用 AppVeyor 持续集成 Github 中的 JS 项目](https://sebastianblade.com/using-appveyor-continuous-integration-in-javascript-project/)
[利用 AppVeyor 实现 GitHub 托管项目的自动化集成](http://www.gulu-dev.com/post/2015-05-01-appveyor-ci)
[Hexo的版本控制与持续集成](https://formulahendry.github.io/2016/12/04/hexo-ci/)
[hexo+github+AppVeyor实现不同电脑写博客](https://killerlei.github.io/2017/04/06/hexo-github-AppVeyor%E5%AE%9E%E7%8E%B0%E4%B8%8D%E5%90%8C%E7%94%B5%E8%84%91%E5%86%99%E5%8D%9A%E5%AE%A2/)
[GitHub Pages + Hexo搭建博客](http://crazymilk.github.io/2015/12/28/GitHub-Pages-Hexo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2/#more)
