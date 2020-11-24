---
layout: post
author: angtylook
title: 使用Docker安装Redis
---

# 使用Docker安装Redis

比起在Docker中安装MySQL，安装Redis方便很多，因为使用Redis一般是用为缓存，就减少了使用Volume保存数据的麻烦。

## 安装和启动
使用`docker run --name redis-test -p 6379:6379 -d redis`就可以启动一个redis服务端了并可以通过本机的6397端口访问。使用`docker ps`可以查看相关信息。
```powershell
docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES     
9bb4890ab920        redis               "docker-entrypoint.s…"   4 seconds ago       Up 3 seconds        0.0.0.0:6379->6379/tcp   redis-test
```

## 使用redis-cli访问

使用`docker exec -it redis-test redis-cli -h localhost`可以连接到刚才创建的容器中，并执行redis-cli连接上去操作redis。
```powershell
docker exec -it redis-test redis-cli -h localhost
localhost:6379> set foo bar
OK
localhost:6379> get foo
"bar"
localhost:6379>
```
上面连接进去并设置了一个foo->bar的KV

## 在宿主机使用python访问redis

1 使用`pip install redis`安装redis模块

2 打开idle测试把foo读出来。
```python
>>> import redis
>>> r = redis.Redis(host='localhost', port=6379)
>>> r.get('foo')
b'bar'
>>> 
```
## 其它的选项

使用自定义选项

`docker run --name redis-test -p 6379:6379 -v /myredis/conf/redis.conf:/usr/local/etc/redis/redis.conf -d redis redis-server /usr/local/etc/redis/redis.conf`

存储数据

`docker run --name redis-test -p 6379:6379 -v /docker/host/dir:/data -d redis redis-server --appendonly yes /usr/local/etc/redis/redis.conf`

两者结合起来应该也OK