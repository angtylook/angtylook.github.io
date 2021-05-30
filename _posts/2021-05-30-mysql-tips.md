---
layout: post
author: angtylook
title: 常见MySQL技巧
---

这里记录一些在工作中大概率会用到MySQL的技巧，主要包括读取binlog、删除重复行和查看表的行数。

### 读取binlog

某次在测试环境不小心把测试人员的一个帐号删了，只好利用binlog找回记录。
```bash
./mysqlbinlog -uroot -h 127.0.0.1 -p --start-datetime="2021-05-10 19:00:00" --stop-datetime="2021-05-10 20:26:00" --read-from-remote-server -vv mysql-bin.000010 > row1.sql
```
从远程MySQL服务器名为`mysql-bin.000010`的bin文件中，读取从"2021-05-10 19:00:00"到"2021-05-10 20:26:00"的biglog日志，结果被重定向到row1.sql文件中。如此，就可以从row1.sql看到删除的行了。

### mysql删除重复的行

某天收反馈，有一个表里有重复的数据，每个重复的数据有两行，需要去掉其中的一行。经过几翻查找，找到了下面的方法，这里记一下，以免遗忘。
这个表的简化(隐去了一些列)结构大概如下。
```sql
create table game_stat (
    `id` int not null primary key,
    `agent_id` int not null,
    `user_type` int not null,
    `day` date
)
```
删掉重复行的SQL语句如下，之后我会解释下这条语句。
```sql
delete from game_stat where `id` in (
    select h1.mid from (
        select min(`id`) as mid, `day`, agent_id, user_type, count(`id`) as n from game_stat group by `day`, agent_id, user_type having n > 1
    ) as h1
);
```

从最里面的子查询开始看，该查询是
```sql
(select min(`id`) as mid, `day`, agent_id, user_type, count(`id`) as n from game_stat group by `day`, agent_id, user_type having n > 1) as h1
```
先看看group by子句，按day,agent_id,user_type来分组，因为业务上这三个列组合形成一个唯一行。
分组之后，使用count(id)记算每个分组的行数，易知当n>1时，就可以判断哪些分组有重复了，通过having子句的条件过滤出来。
而min(id) as mid，就可以算出每组中id最小的重复行。查询结果命名为表h1。

再往上看一层语句，下面是原句。
```sql
(select h1.mid from (select min(`id`) as mid, day, agent_id, user_type, count(`id`) as n from game_stat group by `day`, agent_id, user_type having n > 1) as h1)
```
改写一下，省略上面的生成h1的子查询，就可以简化成为`select h1.mid from h1`，很明确的，我们把重复行的最小的id查询出来了。
最后只需要id在这个集合中的行就可以删除重复的行了。这就是原语句中`delete from game_stat where id in ...`做的工作。

# 查看表名和行数

利用下面的指令，可以查看数据库game中的，表的大致行数，并按行数降序排序
```sql
select table_name, table_rows from `information_schema`.`tables` where TABLE_SCHEMA='game' order by table_rows desc;
```