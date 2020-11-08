---
layout: post
author: angtylook
title: Jenkins在Windows运行CocosCreator卡死
---

项目要搭建Jenkins自动构建，Jenkins Master运行在一台
Linux机器上，用于构建服务端。Jenkins Slave运行在一台
Windows7机器上。


项目是一个用CocosCreator开发的小游戏，根据[官方文档](http://docs.cocos.com/creator/manual/zh/publish/publish-in-command-line.html)
构造好命令行参数。在开发机，和Windows构建机上，测试通过。
Windows7机器上，已装好的CocosCreator, Ant, Android SDK, NDK。
但是使用Jenkins运行时，发现Jenkins Job 卡死在CocosCreator运行中，
没有退出和报错。

折腾好一会儿，发现Jenkins的Slave是以Windows7的System帐户
运行的。对于CocosCreator来说，这是一个全新的帐号，并没有
配置过CocosCreator，第一次运行CocosCreator，会弹一个选择
语言的界面。让用户选择语言，因为是System用户，Jenkins服务
也没有开启“允许服务与桌面交互”。所以一直没有看到这个提示
界面，同时在System用户下，也没有给CocosCreator配置NDK等
路径，无法构建项目。

进服务把Jenkins Slave的服务，改为使用当前已经配置好CocosCreator
的帐号，重启服务搞定。

CocosCreator支持的命令行参数构建，不够完善，像是随便赶工出来。
考虑不周全，应该去掉这些交互。

PS
Widnows7的System用户的Home(profile)目录是`C:\Windows\System32\config\systemprofile`。
