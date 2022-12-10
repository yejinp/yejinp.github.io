---
layout: post
title: PostgreSQL Page页结构解析（4）- 执行DML时表占用空间解析
date: 2022-12-09 12:20:23 +0800
category: PostgreSQL
---

本文介绍了在长事务（开启事务，一直不提交/回滚）的情况下，通过使用pageinspect插件分析Update数据表导致数据表占用空间“暴涨”的原因。

## 一、测试场景
使用psql启动会话Session B
```
testdb=# --------------------------- Session B
testdb=# -- 开启事务
testdb=# begin;
BEGIN
testdb=# 
testdb=# select txid_current();  
 txid_current 
--------------
      1600322
(1 row)

testdb=# -- 创建表&插入100行数据
testdb=# drop table if exists t1;
DROP TABLE
testdb=# create table t1(id int,c1 varchar(50));
CREATE TABLE
testdb=# insert into t1 select generate_series(1,100),'#abcd#';
INSERT 0 100
testdb=# select txid_current();  
 txid_current 
--------------
      1600322
(1 row)

testdb=# select count(*) from t1;
 count 
-------
   100
(1 row)

testdb=# 
testdb=# -- 提交事务
testdb=# end;
COMMIT
testdb=# 
```
开启新的Console创建，使用psql启动会话Session A
```
testdb=# --------------------------- Session A
testdb=# -- 开启事务
testdb=# begin;
BEGIN
testdb=# 
testdb=# -- 查询当前事务
testdb=# select txid_current();  
 txid_current 
--------------
      1600324
(1 row)

testdb=# 
testdb=# -- do nothing
```
虽然什么都不做，但Session A仍然可以开启一个事务，在这里这个事务一直不提交。
回到Session B，查看数据表t1的数据：
```
testdb=# --------------------------- Session B
testdb=# -- 查看数据表
testdb=# select ctid, xmin, xmax, cmin, cmax,id from t1 limit 8;
 ctid  |  xmin   | xmax | cmin | cmax | id 
-------+---------+------+------+------+----
 (0,1) | 1600322 |    0 |    4 |    4 |  1
 (0,2) | 1600322 |    0 |    4 |    4 |  2
 (0,3) | 1600322 |    0 |    4 |    4 |  3
 (0,4) | 1600322 |    0 |    4 |    4 |  4
 (0,5) | 1600322 |    0 |    4 |    4 |  5
 (0,6) | 1600322 |    0 |    4 |    4 |  6
 (0,7) | 1600322 |    0 |    4 |    4 |  7
 (0,8) | 1600322 |    0 |    4 |    4 |  8
(8 rows)

testdb=# -- 查看数据占用空间
testdb=# \set v_tablename t1
testdb=# SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
 pg_size_pretty 
----------------
 8192 bytes
(1 row)

testdb=# -- page_header
testdb=# SELECT * FROM page_header(get_raw_page('t1', 0));
    lsn     | checksum | flags | lower | upper | special | pagesize | version | prune_xid 
------------+----------+-------+-------+-------+---------+----------+---------+-----------
 1/4476E4A0 |        0 |     0 |   424 |  4192 |    8192 |     8192 |       4 |         0
(1 row)
再打开一个Shell窗口，使用pgbench持续不断的更新t1，在此过程进行数据分析。

[xdb@localhost benchmark]$ cat update.sql 
\set rowid random(1,100)
begin;
update t1 set c1=:rowid where id= :rowid;
end;

[xdb@localhost benchmark]$ pgbench -c 2 -C -f ./update.sql -j 1 -n -T 600 -U xdb testdb
```
## 二、数据分析
下面通过pageinspect插件分析t1数据页中的数据。
```
testdb=# \set v_tablename t1
testdb=# 
testdb=# SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
 pg_size_pretty 
----------------
 160 kB
(1 row)

testdb=# -- 查看第0个数据页的头部数据和用户数据
testdb=# SELECT * FROM page_header(get_raw_page('t1', 0));
    lsn     | checksum | flags | lower | upper | special | pagesize | version | prune_xid 
------------+----------+-------+-------+-------+---------+----------+---------+-----------
 1/44787990 |        0 |     2 |   840 |   864 |    8192 |     8192 |       4 |   1600325
(1 row)

testdb=# select * from heap_page_items(get_raw_page('t1',0)) limit 10;
 lp | lp_off | lp_flags | lp_len | t_xmin  | t_xmax  | t_field3 | t_ctid  | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |          t_data          
----+--------+----------+--------+---------+---------+----------+---------+-------------+------------+--------+--------+-------+--------------------------
  1 |   8152 |        1 |     35 | 1600322 | 1600365 |        0 | (0,141) |       16386 |       1282 |     24 |        |       | \x010000000f236162636423
  2 |   8112 |        1 |     35 | 1600322 | 1600325 |        0 | (0,101) |       16386 |       1282 |     24 |        |       | \x020000000f236162636423
  3 |   8072 |        1 |     35 | 1600322 | 1600421 |        0 | (0,197) |       16386 |       1282 |     24 |        |       | \x030000000f236162636423
  4 |   8032 |        1 |     35 | 1600322 | 1600435 |        0 | (1,7)   |           2 |       1282 |     24 |        |       | \x040000000f236162636423
  5 |   7992 |        1 |     35 | 1600322 | 1600474 |        0 | (1,46)  |           2 |       1282 |     24 |        |       | \x050000000f236162636423
  6 |   7952 |        1 |     35 | 1600322 | 1600538 |        0 | (1,110) |           2 |       1282 |     24 |        |       | \x060000000f236162636423
  7 |   7912 |        1 |     35 | 1600322 | 1600396 |        0 | (0,172) |       16386 |       1282 |     24 |        |       | \x070000000f236162636423
  8 |   7872 |        1 |     35 | 1600322 | 1600331 |        0 | (0,107) |       16386 |       1282 |     24 |        |       | \x080000000f236162636423
  9 |   7832 |        1 |     35 | 1600322 | 1600531 |        0 | (1,103) |           2 |       1282 |     24 |        |       | \x090000000f236162636423
 10 |   7792 |        1 |     35 | 1600322 | 1600413 |        0 | (0,189) |       16386 |       1282 |     24 |        |       | \x0a0000000f236162636423
(10 rows)

testdb=# -- 再次查看空间占用
testdb=# SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
 pg_size_pretty 
----------------
 360 kB
(1 row)
```
可以看出，数据表占用空间一直在增长，已远远超出一个数据页的范围。同时，我们注意到t_xmax、t_infomask2、t_infomask中的部分值与先前首次插入的数据的取值不同。

t_xmax

该值 > 0，表示该行数据已废弃，该值为delete/update操作的事务号

t_infomask2

该值为16386，转换为十六进制显示：
```
[xdb@localhost benchmark]$ echo "obase=16;16386"|bc
4002
```
前（低）11位表示属性个数，值为2，也就是说数据表有2个属性（字段）；\x4000表示HEAP_HOT_UPDATED，官方解释如下：
```
An updated tuple, for which the next tuple in the chain is a heap-only tuple. Marked with HEAP_HOT_UPDATED flag.
```

t_infomask

该值为1282，转换为十六进制显示：
```
[xdb@localhost benchmark]$ echo "obase=16;1282"|bc
502
\0x0502 = HEAP_XMIN_COMMITTED | HEAP_XMAX_COMMITTED
```
意思是插入数据的事务和更新（或删除）的事务均已提交。

## 三、空间回收

数据表t1不管如何Update，实际的数据行数只有100行，大小远不到8K，但为何占用了几百KB的空间？原因是PG为了MVCC（多版本并发控制）的需要保留了更新前的“垃圾”数据，这些垃圾数据可以通过vacuum机制定期清理这些垃圾数据。但在本例中，由于“长”事务的存在，垃圾数据不能正常清理。
```
testdb=# SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
 pg_size_pretty 
----------------
 472 kB
(1 row)

testdb=# vacuum t1;
VACUUM
testdb=# SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
 pg_size_pretty 
----------------
 472 kB
(1 row)

testdb=# vacuum full;
VACUUM
testdb=# SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
 pg_size_pretty 
----------------
 472 kB
(1 row)
```
使用命令vacuum t1和vacuum full均不能正常回收垃圾数据，原因是PG认为这些垃圾数据对于正在活动中的事务（Session A）是可见的。

我们回顾一下，Session A的事务号：1600324，Session B插入数据时的事务号：1600322，更新数据时的事务号 > 1600324，Session A（活动事务）查询t1时，通过PG的snapshot机制实现“一致性”读。PG的snapshot可以通过txid_current_snapshot函数获得：
```
testdb=# select txid_current_snapshot();
  txid_current_snapshot  
-------------------------
 1600324:1612465:1600324
(1 row)
```

返回值分为三部分，分别是xin、xmax和xip_list：

格式：xin:xmax:xip_list
```
xin：Earliest transaction ID (txid) that is still active. 未提交并活跃的事务中最小的XID
xmax：First as-yet-unassigned txid. All txids greater than or equal to this are not yet started as of the time of the snapshot, and thus invisible.所有已提交事务中最大的XID + 1
xip_list：Active txids at the time of the snapshot. All of them are between xmin and xmax. A txid that is xmin <= txid < xmax and not in this list was already completed at the time of the snapshot, and thus either visible or dead according to its commit status.
```

数据行中的xin和xmax符合条件xmax>活动事务号xid（1600324）>xin的所有记录均不能被回收！
```
testdb=# --------------------------- Session B
testdb=# vacuum t1;
VACUUM
testdb=# SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
 pg_size_pretty 
----------------
 472 kB
(1 row)

testdb=# vacuum full;
VACUUM
testdb=# SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
 pg_size_pretty 
----------------
 472 kB
(1 row)
```
反之，活动事务提交后，垃圾数据占用的空间可正常回收：
```
testdb=# --------------------------- Session A
testdb=# -- 结束事务
testdb=# end;
COMMIT
```
执行vacuum命令回收垃圾数据：
```
testdb=# --------------------------- Session B
testdb=# vacuum t1;
VACUUM
testdb=# SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
 pg_size_pretty 
----------------
 472 kB
(1 row)
testdb=# vacuum full;

VACUUM
testdb=# SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
 pg_size_pretty 
----------------
 8192 bytes
(1 row)
```
通过vacuum t1命令还不能回收数据？为什么？请注意t_infomask2标志HEAP_HOT_UPDATED，简单来说，在update chain中的data不会回收，由于涉及到HOT机制，详细后续再解析。

## 四、小结
主要有三点需要总结：

1. 保留原数据：PG没有回滚段，在执行更新/删除操作时并没有真正的更新和删除，而是保留原有数据，在合适的时候通过vacuum机制清理垃圾数据；

2. 避免长事务：为了避免垃圾数据暴涨，在业务逻辑允许的情况下应尽可能的尽快提交事务，避免长事务的出现；

3. 查询操作：使用JDBC驱动或者其他驱动连接PG，如明确知道只执行查询操作，请开启自动提交事务。

#### 转自：
[http://blog.itpub.net/6906/viewspace-2374920/](http://blog.itpub.net/6906/viewspace-2374920/)