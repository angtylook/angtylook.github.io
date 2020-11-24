---
layout: post
author: angtylook
title: 通过ssh端口转发访问数据库
---

某天收到需求，要求导出线上环境的数据库中的数据。线上数据库配置了IP白名单，只能通过白名单上的机器访问。其中有一台公共跳板机可以访问，只是该跳板机没有mysql，又不允许安装。

可以利用ssh的端口转发功能，只要把本地的端口通过跳板机转发到线上的mysql-server上，然后通过连接本机的端口即可。

1. 配置ssh
在用户目录下的`.ssh/config`文件中配置跳板机，以方便使用：
```
Host jump
    HostName domain.jump.com
    IdentityFile ~/.ssh/Identity
    Port 22
    User xiaojiu
```
之后通过`ssh -L 3306:10.70.17.19:3306 jump` 打开端口转发。上面这命令的意思是，通过jump把发往本机3306端口的数据，转发到10.70.17.19的3306端口上去。

之后，就可以用mysql直接连接本地的端口，操作线上的数据库了。