日常开发中，总是免不了对表结构的增改删，这里记录针对表列的一些常用MySQL操作的语句。

## 表列的增删改

假设我们有一张表，记录某款游戏的一些营收相关的数据，建表语句如下:
```sql
CREATE TABLE game_revenue (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键',
    `day` DATE NOT NULL UNIQUE COMMENT '日期',
    `device_count` INT NOT NULL DEFAULT 0 COMMENT '新增设备',
    `register_count` INT NOT NULL DEFAULT 0 COMMENT '新增注册',
    `active_count` INT NOT NULL DEFAULT 0 COMMENT '日活',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `day_idx` (`day`) USING BTREE
);
```

### 增加列

现在要增加一列用来统计用户充值的总额，可以使用下面的语句：
```sql
ALTER TABLE game_revenue ADD COLUMN `recharge` INT NOT NULL DEFAULT 0 COMMENT '充值总额';
```
默认新列是加到最后面的，可以使用`FIRST`来指定加到前面，像下面增加一列来统计提现的总额:
```sql
ALTER TABLE game_revenue ADD COLUMN `withdraw` INT NOT NULL DEFAULT 0 COMMENT '提现总额' FIRST;
```
也可以使用`AFTER`关键字指定在某列的后面，比如在`recharge`前端增加一个列表示充值的人数，可以这样写：
```sql
ALTER TABLE game_revenue ADD COLUMN `recharge_count` INT NOT NULL DEFAULT 0 COMMENT '充值人数' AFTER `active_count`;
```

### 重命名与修改列

可以使用`RENAME`对列名重命名，`recharge`表示充值总额不够达意，想改成`recharge_amount`，可以这么做：
```sql
ALTER TABLE game_revenue RENAME COLUMN `recharge` TO `recharge_amount`;
```
重命名不会更改列的类型定义，如果要更改列的类型定义，可以使用`MODIFY`，下面增加一列表示充值率之后更改它的类型为`FLOAT`。
```sql
# 使用INT表示充值率不妥
ALTER TABLE game_revenue ADD COLUMN `recharge_rate` INT NOT NULL DEFAULT 0 COMMENT '充值率';
# 更改为使用FLOAT
ALTER TABLE game_revenue MODIFY COLUMN `recharge_rate` FLOAT NOT NULL DEFAULT 0 COMMENT '充值率';
```
如果在MODIFY语句后面使用FIRST、AFTER则可以调整列的顺序，`withdraw`在`id`的前面不够美观，把它挪到最后面去：
```sql
ALTER TABLE game_revenue MODIFY COLUMN `withdraw` INT NOT NULL DEFAULT 0 COMMENT '提现总额' AFTER `recharge_rate`;
```
如果你想把重命名，更改类型和调整位置一起做也是可以的，使用`CHANGE`就可以达到目的，举个列子：
```sql
# 在前面增加一列类型为INT，表示提现率的列
ALTER TABLE game_revenue ADD COLUMN `withdraw_percent` INT NOT NULL DEFAULT 0 COMMENT '提现率' FIRST;
# 现在调整到尾部，更改名称为withdraw_rate，并更改类型为FLOAT
ALTER TABLE game_revenue CHANGE COLUMN `withdraw_percent` `withdraw_rate` FLOAT NOT NULL DEFAULT 0 COMMENT '充值率' AFTER `withdraw`;
```

### 删除列

删除列使用`DROP`，还是很简单的，比如游戏去掉了提现功能，提现相关的字段不再使用，要把相关的列删除：
```sql
ALTER TABLE game_revenue DROP COLUMN `withdraw`;
ALTER TABLE game_revenue DROP COLUMN `withdraw_rate`;
# 也可以同时删除两列
ALTER TABLE game_revenue DROP COLUMN `withdraw`, DROP COLUMN `withdraw_rate`;
```

## 算法、性能、并发

ALTER TABLE的操作，在MySQL中会使用下列三种算法中的一种：

- 复制副本(COPY): 表数据将一行接一行地从原表复制到副本，操作在副本上修改。修改期间不允许DML操作。

- 使用原表(INPLACE): 操作避免了复制数据，但是有可能原地重建表(影响性能)。在操作的准备和执行阶段，有可能会施加表的元数据排它锁。一般来说，这种算法支持并发DML。

- 即时操作(INSTANT): 操作只需修改数据目录下的元数据。在操作的准备的执行阶段，不需要元数据的排它锁，表数据不受影响，几乎立即完成操作。允许并发的DML。(MySQL 8.0.12版本引入的功能)

通过查阅[MySQL8.0的官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html#online-ddl-column-operations)，发现对于列的操作:
- 只有更改列的数据类型，会复制副本并且不支持并发的DML
- 会重建表的操作有：
    - 删除列
    - 重排列
    - 更改列的数据类型
    - 使列支持NULL
    - 使列不支持，即支持NOT NULL
