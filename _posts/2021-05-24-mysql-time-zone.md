---
layout: post
author: angtylook
title: MySQL时区的相关常识
---

MySQL中的时区为分三层:
- 第一层是系统时区(`system_time_zone`)，在MySQL启动时从操作系统获取，之后不再改变。
- 第二层是MySQL服务器的全局时区，可以在启动时使用参数`--default-time-zone`指定，也可以在配置文件中使用配置项`default-time-zone='timezone'`指定。在运行中，如果有`SYSTEM_VARIABLES_ADMIN`权限，也可以使用下面的命令指定。
```sql
SET GLOBAL time_zone = timezone;
```
- 第三层是会话时区，每个连接上来的客户端会话，都可以有自己的时区设置(`time_zone`)。可以使用下面的命令来切换。
```sql
SET time_zone = timezone;
```

MySQL启动时，会把全局时区设为系统时区，并把会话时区设为全局时区。使用可以下面的命令查看当前的时区配置。

```sql
SELECT @@GLOBAL.time_zone, @@SESSION.time_zone;
```

会话时区会影响时区敏感的时间值的显示和存储。包括`Now()`,`CURTIME()`函数返回值的显示，
`TIMESTAMP`列时间值的存取，`TIMESTAMP`存储时会把值从会话时区转换为UTC时区，读取时会从UTC转换为会话时区。会话时区不会影响像`UTC_TIMESTAMP()`函数的返回值，和`DATE`，`TIME`，`DATETIME`这些列的值。只有这些类型的值从`TIMESTAMP`值转换得来时才会受会话时区的影响。

对于主从复制，主从服务器会假设它们在同一个时区，如果在不同时区之间复制，需要在主从服务器中设置为相同的时区，否则依赖本地时间的语句复制会出错。比如使用了`NOW()`或者`FROM_UNIXTIME()`函数的语句。
