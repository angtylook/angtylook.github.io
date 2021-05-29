---
layout: post
author: angtylook
title: MySQL中的索引和约束
---

翻了MySQL的手册文档，对索引、约束和KEY的介绍。这三个概念交织难以理解。
从它们的作用和目的来理解或许会容易一些。索引是用来加快查询效率的，约束
是对用来指示数据满足的一致性关系，而KEY则表示一组约束。

例如表:
```sql
# 游戏玩家表
CREATE TABLE game_user (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `uid` BIGINT NOT NULL DEFAULT 0,
    `name` VARCHAR(128) NOT NULL DEFAULT '',
    `currency` INT NOT NULL DEFAULT 0,
    PRIMARY KEY (`id`),
    UNIQUE INDEX `uid_idx` (`uid`),
    CONSTRAINT name_unique UNIQUE(`name`)
);

INSERT INTO `game_user`(`uid`, `name`, `currency`) VALUE(1, 'zhao', 100);
INSERT INTO `game_user`(`uid`, `name`, `currency`) VALUE(2, 'qiao', 110);
INSERT INTO `game_user`(`uid`, `name`, `currency`) VALUE(3, 'shun', 120);
INSERT INTO `game_user`(`uid`, `name`, `currency`) VALUE(4, 'li', 200);
```
这里的name_unqiue约束就表示，这个表中的name列，必须是唯一不重复的。当想插入有相同名字的行时会报错。
比如下面的语句，就报告`li`这个值有重复，违反了`game_user.name_unique`约束。
```sql
# 运行插入语句会报错
INSERT INTO `game_user`(`uid`, `name`, `currency`) VALUE(5, 'li', 400);
# ERROR 1062 (23000): Duplicate entry 'li' for key 'game_user.name_unique'
```
而索引uid_idx表示给uid列创建一个索引，UNIQUE表示该列是唯一的，插入相同的值也会报错。
```sql
# 运行插入语句会报错
INSERT INTO `game_user`(`uid`, `name`, `currency`) VALUE(4, 'li2', 400);
# ERROR 1062 (23000): Duplicate entry '4' for key 'game_user.uid_idx'
```
唯一索引虽然可以做唯一约束，但是索引主要是用来加快查询的。先看下下面两条语句的explain。
```
mysql> explain select * from game_user where uid = 3;
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | game_user | NULL       | const | uid_idx       | uid_idx | 8       | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from game_user where name = "li";
+----+-------------+-----------+------------+-------+---------------+-------------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type  | possible_keys | key         | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+-------+---------------+-------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | game_user | NULL       | const | name_unique   | name_unique | 514     | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+-------+---------------+-------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
看一下输出的possible_keys就很明显的，按uid查询时，会使用uid_idx索引，而按name查询时，竟然会使用name_unique这个约束，
可以看到有些情况约束和索引的作用是相同的。
而KEY表示一种索引和约束的集合，比如PRIMARY KEY就表示唯一(UNIQUE)、非空(NOT NULL)这些约束，并且会创建为相应的列创建索引。
如下，如果使用id来查询，就发现会使用PRIMARY这个主键索引。
```
mysql> explain select * from game_user where id = 1;
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | game_user | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
### 索引的增删

除了在建表时，指定索引，建表后也是可以给表增加索引，或者删除表的索引。

显示索引和约束使用`show indexes from table_name;`查一下game_user索引
```sql
mysql> show indexes from game_user;
+-----------+------------+-------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table     | Non_unique | Key_name    | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-----------+------------+-------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| game_user |          0 | PRIMARY     |            1 | id          | A         |           4 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| game_user |          0 | uid_idx     |            1 | uid         | A         |           4 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| game_user |          0 | name_unique |            1 | name        | A         |           4 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+-----------+------------+-------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
3 rows in set (0.00 sec)
```
除了uid_idx以外，可以看到主键索引和name_unqiue约束也在里面。这里不能很好的区分，它们底层的实现有相同之处吧。

增加索引有两种语法，第一种是使用CREATE INDEX，第二种是使用ALTER TABLE。
```sql
CREATE INDEX index_name ON table_name(col_list);
ALTER TABLE table_name ADD INDEX index_name (col_list);
```
为game_user的name创建索引，可以使用下面两种语句
```sql
CREATE INDEX name_idx ON game_user(name);
# 或者使用
ALTER TABLE game_user ADD INDEX name_idx(name);
```
要删除索引也可以使用两种语法，分别如下
```sql
DROP INDEX name_idx ON game_user;
ALTER TABLE game_user DROP INDEX name_idx;
```
当然也可能使用下面的语句给索引重命名
```sql
ALTER TABLE tbl_name RENAME INDEX old_index_name TO new_index_name;
```

在MySQL中可以对多个列建立复合索引，当对多个列使用group by时，针对这些列创建索引可以提高性能。
```sql
mysql> explain select uid, name, sum(currency) from game_user group by uid, name;
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-----------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra           |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-----------------+
|  1 | SIMPLE      | game_user | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    4 |   100.00 | Using temporary |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-----------------+
1 row in set, 1 warning (0.00 sec)

mysql> create index uid_name_idx on game_user(uid,name);
Query OK, 0 rows affected (0.06 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select uid, name, sum(currency) from game_user group by uid, name;
+----+-------------+-----------+------------+-------+---------------+--------------+---------+------+------+----------+-------+
| id | select_type | table     | partitions | type  | possible_keys | key          | key_len | ref  | rows | filtered | Extra |
+----+-------------+-----------+------------+-------+---------------+--------------+---------+------+------+----------+-------+
|  1 | SIMPLE      | game_user | NULL       | index | uid_name_idx  | uid_name_idx | 522     | NULL |    4 |   100.00 | NULL  |
+----+-------------+-----------+------------+-------+---------------+--------------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
如上面的执行过程，在没有创建uid_name_idx时，group by的查询使用了临时表，而创建之后，就会使索引。

### 约束的增删

约束的创建和修改语句，跟索引差不多，几乎是把关键字INDEX改为CONSTRAINT就可以了，不细说。

可以在建表时声明约束

```sql
CREATE TABLE contacts (
    contact_id INT(11) PRIMARY KEY AUTO_INCREMENT,
    reference_number INT(11) NOT NULL,
    last_name VARCHAR(30) NOT NULL,
    first_name VARCHAR(25),
    birthday DATE,
    CONSTRAINT contacts_unique UNIQUE (last_name, first_name)
);
```
也可以在建表之后，再使用下面的语句，增加和删除约束。
```sql
# 修改表增加唯一约束
ALTER TABLE table_name ADD CONSTRAINT constraint_name UNIQUE (column1, column2, ... column_n);
# 删掉约束
ALTER TABLE table_name DROP INDEX constraint_name;
```

例如
```sql
ALTER TABLE contacts ADD CONSTRAINT contact_name_unique UNIQUE (last_name, first_name);
ALTER TABLE contacts DROP INDEX contacts_unique;
```

### 性能和并发

当在线操作索引时，就怕长时间的不允许并发的DML操作，引起线上无法读写数据，下面的列表来自[MySQL文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html#online-ddl-index-operations)，标明了操作索引的影响。


Table 15.16 Online DDL Support for Index Operations

| Operation | Instant | In Place | Rebuilds Table | Permits Concurrent DML | Only Modifies Metadata |
|---|---|---|---|---|---|
| Creating or adding a secondary index | No | Yes | No | Yes | No |
|Dropping an index | No | Yes | No | Yes | Yes |
|Renaming an index | No | Yes | No | Yes | Yes |
|Adding a FULLTEXT index | No | Yes* | No* | No | No |
|Adding a SPATIAL index | No | Yes | No | No | No |
|Changing the index type | Yes | Yes | No | Yes | Yes |


Table 15.17 Online DDL Support for Primary Key Operations

Operation|Instant|In Place|Rebuilds Table|Permits Concurrent DML|Only Modifies Metadata
---|---|---|---|---|---
Adding a primary key|No|Yes*|Yes*|Yes|No
Dropping a primary key|No|No|Yes|No|No
Dropping a primary key and adding another|No|Yes|Yes|Yes|No
