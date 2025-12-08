---
title: 搭建自己的博客（一）
tag: 其他
date: 2019-06-11T11:06:00+08:00
description: 搭建自己的博客
---

+ 前言：
   + 昨天用linux搭建自己了的博客，但是早上起床突然想到学习可能用的是win10，写blog的时候还要换到Linux下
	 会很不方便，所以想着能不能实现多终端管理博客，然后想到git托管代码的方式与之相似，可以用git 进行控制
	 下面开始实现
+ 实现：
   + **linux系统下**
   + 先在自己的服务器上搭建git（步骤省略）
   + 进入 linux 的hexo的源码，切换到hexo分支
   ```
   git init
   git add *
   git commit -m ""
   git checkout -b hexo
   git remote add origin git@guoxy.top:/example/blog.git
   git push origin hexo
   ```
   + 这样就把源码推送到了服务器上
   
   + **Windows下**
   + 首先肯定配置环境 git node hexo
   + 再clone到本地
     ```
	  git clone -b git@guoxy.top:/example/blog.git hexo
	  ```
   + 写完博客后
    ```
	 git add source
	 git commit -m ""
	 git push
	 ```
   + 部署
    ```
	 hexo g -d
	 ```
+ 后言：
   + 下次切换设备或系统要git pull 一下
   
***
<a href="https://www.jianshu.com/p/1ae341483683" target="_blank">一篇参考博客</a>
***