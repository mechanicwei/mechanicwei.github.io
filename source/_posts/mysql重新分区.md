---
title: mysql重新分区
date: 2025.07.09
tags:
---

## 背景
message等几个表由于没有提前创建好分区，导致近半年的数据已经写入pMAX分区，为了降低存储成本，需要定期清理半年之前的数据，所以需要重新构建正确的分区

## 方案一
**用REORGANIZE PARTITION重新分区**
```sql
ALTER TABLE message REORGANIZE PARTITION pMAX INTO (
  PARTITION p202513 VALUES LESS THAN (202513),
  PARTITION p202526 VALUES LESS THAN (202526),
  PARTITION p202539 VALUES LESS THAN (202539),
  PARTITION p202552 VALUES LESS THAN (202552),
  PARTITION pMAX VALUES LESS THAN MAXVALUE
);
```
但polardb 内核版本8.0.2才支持在线分区维护功能，之前的版本`REORGANIZE PARTITION`会阻塞读写，造成业务不可用
内核版本支持在线热升级，升级过程仅闪断一次
REORGANIZE的速度：大约4w/s的速度

## 方案二
**仅保证新数据进正确的分区，当前数据统一分入一个分区pOLD，6个月后删除 pOLD分区**
1. 先创建一个表结构一致的临时表message_old，**但不需要分区**
```sql
CREATE TABLE `message_old`(
    ...
);
```
2. 将已有数据交换出来
```sql
ALTER TABLE message EXCHANGE PARTITION pMAX WITH TABLE message_old;
```
3. 创建新分区，因为没有数据了，这一步会很快，并发产生的新数据会进p202539里
```sql
ALTER TABLE message REORGANIZE PARTITION pMAX INTO (
  PARTITION pOLD VALUES LESS THAN (202527),
  PARTITION p202539 VALUES LESS THAN (202539),
  PARTITION p202552 VALUES LESS THAN (202552),
  PARTITION pMAX VALUES LESS THAN MAXVALUE
);
```
再将以前的数据换回去，使用`WITHOUT VALIDATION`跳过检查，秒级完成
```sql
ALTER TABLE message EXCHANGE PARTITION pOLD WITH TABLE message_old WITHOUT VALIDATION;
```
**注意**：
为了保证这段时间并发写入的数据和老数据库在不同的分区，同时为了尽可能减少数据在错误的分区
应该选在周天凌晨处理（yearweek(`created_on`,0)是从周日开始算一周的第一天），这样只有迁移之前周天的数据会进入错误的分区
这部分错误分区的数据：
- 如果通过分区键查询，可能会查不到
- 更新会失败，无法更新

实际测试发现表不能增加过字段，否则会报`(1731, "Non matching attribute 'INSTANT COLUMN(s)' between partition and table")`
需要手动执行一次 `ALTER TABLE table_name ENGINE=InnoDB`，虽然是online DDL，但本质是重建表，会耗时很久

```sql
// 模拟测试
CREATE TABLE orders_test (
  id INT NOT NULL,
  order_date DATE NOT NULL,
  amount DECIMAL(10, 2),
  PRIMARY KEY (id, order_date)
)
PARTITION BY RANGE (YEARWEEK(order_date)) (
  PARTITION p202452 VALUES LESS THAN (202452),
  PARTITION pmax  VALUES LESS THAN MAXVALUE
);

// 4 5 6 7数据进pmax
insert into orders_test values (1, '2022-01-01', 100.00);
insert into orders_test values (2, '2023-01-01', 101.00);
insert into orders_test values (3, '2024-01-01', 102.00);
insert into orders_test values (4, '2025-01-01', 103.00);
insert into orders_test values (5, '2025-02-01', 103.00);
insert into orders_test values (6, '2025-03-01', 103.00);
insert into orders_test values (7, '2025-04-01', 103.00);

CREATE TABLE orders_old (
  id INT NOT NULL,
  order_date DATE NOT NULL,
  amount DECIMAL(10, 2),
  PRIMARY KEY (id, order_date)
);

ALTER TABLE orders_test EXCHANGE PARTITION pmax WITH TABLE orders_old;

ALTER TABLE orders_test
REORGANIZE PARTITION pmax INTO (
  PARTITION p202527 VALUES LESS THAN (202527),
  PARTITION p202539 VALUES LESS THAN (202539),
  PARTITION p202552 VALUES LESS THAN (202552),
  PARTITION pmax  VALUES LESS THAN MAXVALUE
);

//模拟并发写入数据
insert into orders_test values (8, now(), 103.00);

ALTER TABLE orders_test EXCHANGE PARTITION p202527 WITH TABLE orders_old WITHOUT VALIDATION;
```

## 方案三
**使用DTS同步已有数据，追平后，停写，RENAME表**
创建新表，新表有正确的分区，DST只同步数据，不同步表结构
```sql
CREATE TABLE `message_new` (
    ...
)
PARTITION pMAX INTO (
  PARTITION p202513 VALUES LESS THAN (202513),
  PARTITION p202526 VALUES LESS THAN (202526),
  PARTITION p202539 VALUES LESS THAN (202539),
  PARTITION p202552 VALUES LESS THAN (202552),
  PARTITION pMAX VALUES LESS THAN MAXVALUE
)
```
此方案停写是关键，可选方式：
1、代码上处理，所有写的地方都要处理
2、数据库层的停写：把数据库账号改为只读权限
3、预留自增主键，rename之后，迁移相差的数据（要求主键是自增类型）
rename之前查询出`select max(id) from message_new`，rename之后，`select max(id) from message_old`
中间相差的数据就是dts延迟尚未同步的数据，可用`insert ignore`插入（但该方案会造成这部分更新数据的丢失）

RENAME表，**原子操作**（不能分成2次操作）
```sql
RENAME TABLE message TO message_old, message_new TO message;
```

## 对比
方案一：会迁移数据，完成后能生成正确的分区，但整个过程耗时会比较久，期间会占用资源
方案二：只做表结构元数据的修改，不会迁移数据，耗时很短，但老数据在同一个分区，且少部分数据会进入错误的分区
方案三：最稳，dst同步数据不会对源库造成很大的影响，完成后能生成正确的分区，但要看业务上处理停写是否方便
