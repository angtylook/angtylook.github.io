---
layout: post
author: angtylook
title: 使用Docker安装MySQL
---

在后台开发中，一般使用MySQL来存储用户的数据。比如开发小游戏服务端，生产环境跑在机房的Linux环境，连接机房的云数据库。平时在Windows下开发，需要使用本地的数据库。为了本机Windows系统的稳定，不安装太多的软件，决定使用Docker在安装一个MySQL用于开发。

我们开发的小游戏后端，用户的数据存储在MySQL里面，上线后跑在机房的Linux机器上，连接云数据库。平时开发使用Windows机器，为了保证本地Windows开发机的稳定，使用Docker安装一个MySQL用于本地的开发调试。

## 1. 安装Docker

在[Widnows下安装Docker](https://docs.docker.com/docker-for-windows/)是很方便的，跟安装别的软件一样，下载安装即可，也没什么特别的配置。装完之后，命令行环境输入`docker --version`能正常输出版本号，就成功了。

## 2. 在Docker中安装MySQL

在Docker中image看作container的模板，container看作image的实例。使用`docker pull mysql`就可以把[MySQL](https://hub.docker.com/_/mysql)的image下载到本地。

首先创建一个volume用来保存MySQL的数据，使之不随着container的关闭和删除而被删除，volume的生命周期独立于container。

```docker volume create sql-data```

上面的命令创建了一个名叫`sql-data`的volume，后面将引用它的名字来指定作为MySQL的数据存储位置。

创建之后运行下面的命令即可把MySQL启动起来。

```
docker run --name mysql-dev -v sql-data:/var/lib/mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=mysql123 -d mysql --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
```

`--name mysql-dev`是给启用的container起的名字，用于标识该container，后面使用docker命令管理时可以使用该名字。

`-v sql-data:/var/lib/mysql`这里把先前创建的名为sql-data的volume挂载到MySQL默认的数据存储目录`/var/lib/mysql`

`-p 3306:3306`把MySQL的端口映射到宿主本机的3306端口，这样开发调试的时候，连接localhost的3306端口就可以了。这里绑定的ip是0.0.0.0，如果要绑定特定的ip可以通过`-p 127.0.0.1:3306:3306`这种形式的参数来指定ip。

`-e MYSQL_ROOT_PASSWORD=mysql123`设置环境变量MYSQL_ROOT_PASSWORD为mysql123，该环境变量用于设置container中MySQL的root用户的密码。如果想重用以往的数据，比如创建一个新的container也使用sql-data的数据，可以省略这个参数，以免更改sql-data的数据造成破坏。

`-d`是指后台运行，跟在后面的`mysql`表示image的名称。

`--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci`是MySQL的配置参数，这些参数也可以写在/my/custom/my.cnf这样的配置文件中，通过映射目录`-v /my/custom:/etc/mysql/conf.d`的方式，配置给container。

可以通过`docker ps`查看正在运行的container

```
PS C:\Users\self> docker ps                                                                                                                                               CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
b3bea205536c        mysql               "docker-entrypoint.s…"   56 seconds ago      Up 54 seconds       0.0.0.0:3306->3306/tcp, 33060/tcp   mysql-dev
```

`docker port`查看端口映射
```
PS C:\Users\self> docker port mysql-dev                                                                                                                                   3306/tcp -> 0.0.0.0:3306
```

`docker stop/start mysql-dev`用来停止和启动container mysql-dev

在本地开发中，经常需要连接数据库做一些管理和维护的操作，这时可以通过`docker exec -it mysql-dev bash`连接到container的bash环境中，进行操作。
```
PS C:\Users\self> docker exec -it mysql-dev bash                                                                                                                          root@b3bea205536c:/# mysql -hlocalhost -uroot -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.16 MySQL Community Server - GPL

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
上面是在bash里面通过mysql命令连接的数据库，也可以直接执行mysql进行连接和管理。这里跟在`mysql-dev`后面的`mysql -hlocalhost -uroot -p`表示要执行的程序和参数。下面可以看到，连接之后还通过`create user 'wheel'@'%' identified by 'cherry';`在MySQL中创建一个叫wheel的用户密码为cherry。
```
PS C:\Users\jxhap> docker exec -it mysql-dev mysql -hlocalhost -uroot -p                                                                                                   Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.0.16 MySQL Community Server - GPL

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> create user 'wheel'@'%' identified by 'cherry';
Query OK, 0 rows affected (0.01 sec)
```
也可以另外启动一个docker的container的运行mysql来连接和管理。首先使用`docker inspect mysql-dev`查看container mysql-dev的详细信息，其中有一项是IPAddress表示该container的ip地址，另一个container将通过该ip连接MySQL。执行`docker run -it --rm mysql mysql -h 172.17.0.3 -uwheel -p`启用一个新的container并使用wheel用户登录了mysql-dev中的MySQL。

```
PS C:\Users\jxhap> docker run -it --rm mysql mysql -h 172.17.0.3 -uwheel -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 12
Server version: 8.0.16 MySQL Community Server - GPL

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

数据的备份和恢复可以使用mysqldump来进行，原理在于登入到container中执行命令和处理输入输出。
```
docker exec some-mysql sh -c 'exec mysqldump --all-databases -uroot -p"$MYSQL_ROOT_PASSWORD"' > /some/path/on/your/host/all-databases.sql

docker exec -i some-mysql sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD"' < /some/path/on/your/host/all-databases.sql
```
