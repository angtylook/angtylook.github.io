---
layout: post
author: angtylook
title: 使用Docker安装Mongo
---

跟安装MySQL的形式差不多，先创建数据存储Volume，之后使用`docker pull mongo`直接拉取镜像再运行。
指定初始化的帐号密码，存储路径，对外端口等

```shell
docker volume create mongo_data
docker run -d --name mongo-free -e MONGO_INITDB_ROOT_USERNAME=root -e MONGO_INITDB_ROOT_PASSWORD=mongo123456 -v mongo_data:/data/db -p 27017:27017 mongo
```

如果有需要也可以加上`-v /my/custom:/etc/mongo`指定自定义配置的目录，然后在启动时加入启动参数`--config /etc/mongo/mongod.conf`

数据备份可以使用下面的命令

`docker exec mongo-free sh -c 'exec mongodump -d <database_name> --archive' > /some/path/on/your/host/all-collections.archive`

要使用mongo的shell可以使用

`docker exec -it mongo-free mongo`

连接上去之后，给`test`数据库创建一个用户用于日常操作

```javascript
use admin
db.auth("root", "mongo123456")
db.createUser({user:"test", pwd:"test123", roles: ["readWriteAnyDatabase"]})
```

### 一些指令
#### db操作

1. 使用或者创建某个数据库
    `use db_name`
2. 显示所有的数据库
    `show dbs`
3. 查看当前的数据库
    `db`

#### 插入数据

```javascript
   db.user.insertOne({uid:"12345", gold: 1000, rank: {rank:1, score: 100}})
   db.user.insertMany([
        {uid:"12346", gold: 1100, rank: {rank:1, score: 100}},
        {uid:"12347", gold: 1200, rank: {rank:2, score: 200}},
        {uid:"12348", gold: 1300, rank: {rank:3, score: 300}},
        {uid:"12349", gold: 1400, rank: {rank:4, score: 400}},
        {uid:"12350", gold: 1500, rank: {rank:5, score: 500}}
    ])
```