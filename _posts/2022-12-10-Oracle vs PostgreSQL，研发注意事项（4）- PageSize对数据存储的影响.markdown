---
layout: post
title: Oracle vs PostgreSQL，研发注意事项（4）- PageSize对数据存储的影响
date: 2022-12-10 10:00:23 +0800
category: PostgreSQL
---

本文简单介绍了PageSize对数据存储的影响。Oracle有BlockSize的概念，但对数据存储没有太多的影响，看起来对于使用者来说是透明的，但在PG中，BlockSize对于不同的Column Storage选项会有不同的影响。

## 一、Oracle
Oracle数据库，Block Size设定为8K
```
TEST-cndb@pc15.infogov>show parameter db_block_size
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_block_size                        integer     8192
-- 为方便分析，创建一个只有128K的表空间
TEST-cndb@pc15.infogov>create tablespace tbs_tmp datafile '/data/oradata/testtbs/tbs_tmp01.dbf' size 128K autoextend off;
Tablespace created.
-- 创建一张表，存储在tbs_tmp表空间，一行的大小 > 8K
TEST-cndb@pc15.infogov>drop table t2 purge;
Table dropped.
TEST-cndb@pc15.infogov>
TEST-cndb@pc15.infogov>create table t2(id int,c1 char(2000),c2 char(2000),c3 char(2000),c4 char(2000),c5 char(2000)) tablespace tbs_tmp;
Table created.
TEST-cndb@pc15.infogov>insert into t2 values(1,rpad('1',2000,'1'),rpad('2',2000,'2'),rpad('3',2000,'3'),rpad('4',2000,'4'),rpad('5',2000,'5'));
1 row created.
TEST-cndb@pc15.infogov>commit;
Commit complete.
-- 执行checkpoint，flush数据到磁盘
TEST-cndb@pc15.infogov>alter system checkpoint;
System altered.
```

在一行数据大于Block Size时，Oracle使用行链接的方式实现跨块存储。

```
TEST-cndb@pc15.infogov>drop table CHAINED_ROWS purge;
Table dropped.
TEST-cndb@pc15.infogov>create table CHAINED_ROWS (
  2  owner_name varchar2(30),
  3  table_name varchar2(30),
  4  cluster_name varchar2(30),
  5  partition_name varchar2(30),
  6  subpartition_name varchar2(30),
  7  head_rowid rowid,
  8  analyze_timestamp date
  9  );
Table created.
TEST-cndb@pc15.infogov>analyze table t2 list chained rows;
Table analyzed.
TEST-cndb@pc15.infogov>select table_name,head_rowid from CHAINED_ROWS;

TABLE_NAME                     HEAD_ROWID
------------------------------ ------------------
T2                             AAAZroAB9AAAAAPAAA
```

## 二、PG
PG的默认Block Size为8K，可以在编译安装时修改，不作任何调整，创建一张预期行占用空间可能 > 8K的数据表，插入测试数据：

```
testdb=# drop table if exists t2;
n_filepath('t2');DROP TABLE
testdb=# create table t2(id int,c1 char(4000),c2 char(4000),c3 char(4000));
CREATE TABLE
testdb=# 
testdb=# \d+ t2
                                         Table "public.t2"
 Column |      Type       | Collation | Nullable | Default | Storage  | Stats target | Description 
--------+-----------------+-----------+----------+---------+----------+--------------+-------------
 id     | integer         |           |          |         | plain    |              | 
 c1     | character(4000) |           |          |         | extended |              | 
 c2     | character(4000) |           |          |         | extended |              | 
 c3     | character(4000) |           |          |         | extended |              | 

testdb=# 
testdb=# insert into t2 values(1,'11','12','13');
INSERT 0 1
testdb=# insert into t2 values(2,rpad('2',4000,'2'),rpad('3',4000,'3'),rpad('4',4000,'4'));
INSERT 0 1
testdb=# 
testdb=# checkpoint;
CHECKPOINT
testdb=# select pg_relation_filepath('t2');
 pg_relation_filepath 
----------------------
 base/16477/26700
(1 row)

[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/26700
00000000  01 00 00 00 d0 d8 26 db  00 00 00 00 20 00 68 1e  |......&..... .h.|
00000010  00 20 04 20 00 00 00 00  30 9f 9e 01 68 9e 88 01  |. . ....0...h...|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001e60  00 00 00 00 00 00 00 00  07 9c 18 00 00 00 00 00  |................|
00001e70  00 00 00 00 00 00 00 00  02 00 04 00 02 08 18 00  |................|
00001e80  02 00 00 00 e2 00 00 00  a0 0f 00 00 fe 32 0f 01  |.............2..|
00001e90  ff 0f 01 ff 0f 01 ff 0f  01 ff 0f 01 ff 0f 01 ff  |................|
00001ea0  0f 01 ff ff 0f 01 ff 0f  01 ff 0f 01 ff 0f 01 ff  |................|
00001eb0  0f 01 ff 0f 01 ff 0f 01  ff 0f 01 9f e2 00 00 00  |................|
00001ec0  a0 0f 00 00 fe 33 0f 01  ff 0f 01 ff 0f 01 ff 0f  |.....3..........|
00001ed0  01 ff 0f 01 ff 0f 01 ff  0f 01 ff ff 0f 01 ff 0f  |................|
00001ee0  01 ff 0f 01 ff 0f 01 ff  0f 01 ff 0f 01 ff 0f 01  |................|
00001ef0  ff 0f 01 9f e2 00 00 00  a0 0f 00 00 fe 34 0f 01  |.............4..|
00001f00  ff 0f 01 ff 0f 01 ff 0f  01 ff 0f 01 ff 0f 01 ff  |................|
00001f10  0f 01 ff ff 0f 01 ff 0f  01 ff 0f 01 ff 0f 01 ff  |................|
00001f20  0f 01 ff 0f 01 ff 0f 01  ff 0f 01 9f 00 00 00 00  |................|
00001f30  06 9c 18 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001f40  01 00 04 00 02 08 18 00  01 00 00 00 ee 00 00 00  |................|
00001f50  a0 0f 00 00 f8 31 31 20  0f 01 ff 0f 01 ff 0f 01  |.....11 ........|
00001f60  ff 0f 01 ff 0f 01 ff ff  0f 01 ff 0f 01 ff 0f 01  |................|
00001f70  ff 0f 01 ff 0f 01 ff 0f  01 ff 0f 01 ff 0f 01 ff  |................|
00001f80  03 0f 01 ff 0f 01 9d 00  ee 00 00 00 a0 0f 00 00  |................|
00001f90  f8 31 32 20 0f 01 ff 0f  01 ff 0f 01 ff 0f 01 ff  |.12 ............|
00001fa0  0f 01 ff ff 0f 01 ff 0f  01 ff 0f 01 ff 0f 01 ff  |................|
00001fb0  0f 01 ff 0f 01 ff 0f 01  ff 0f 01 ff 03 0f 01 ff  |................|
00001fc0  0f 01 9d 00 ee 00 00 00  a0 0f 00 00 f8 31 33 20  |.............13 |
00001fd0  0f 01 ff 0f 01 ff 0f 01  ff 0f 01 ff 0f 01 ff ff  |................|
00001fe0  0f 01 ff 0f 01 ff 0f 01  ff 0f 01 ff 0f 01 ff 0f  |................|
00001ff0  01 ff 0f 01 ff 0f 01 ff  03 0f 01 ff 0f 01 9d 00  |................|
00002000
```

可以看到，虽然行数据>8K，但在PG中，这些数据都存储在一个block中（显然使用了压缩），00000000~00002000为一个block 8K。

实际上，在PG中，PG使用称为TOAST (The Oversized-Attribute Storage Technique)的技术：

>If any of the columns of a table are TOAST-able, the table will have an associated TOAST table, whose OID is stored in the table's pg_class.reltoastrelid entry. On-disk TOASTed values are kept in the TOAST table, as described in more detail below.

对于数据表的列，有四种存储选项：

> **PLAIN** prevents either compression or out-of-line storage; furthermore it disables use of single-byte headers for varlena types. This is the only possible strategy for columns of non-TOAST-able data types.

> **EXTENDED** allows both compression and out-of-line storage. This is the default for most TOAST-able data types. Compression will be attempted first, then out-of-line storage if the row is still too big.

> **EXTERNAL** allows out-of-line storage but not compression. Use of EXTERNAL will make substring operations on wide text and bytea columns faster (at the penalty of increased storage space) because these operations are optimized to fetch only the required parts of the out-of-line value when it is not compressed.

> **MAIN** allows compression but not out-of-line storage. (Actually, out-of-line storage will still be performed for such columns, but only as a last resort when there is no other way to make the row small enough to fit on a page.)

默认选项为EXTENDED。

我们不妨尝试PLAIN和EXTERNAL这两种选项：
### PLAIN
```
-- PLAIN
testdb=# drop table if exists t3;
11','12','13');
insert into t3 values(2,rpad('2',4000,'2'),rpad('3',4000,'3'),rpad('4',4000,'4'));DROP TABLE
testdb=# create table t3(id int,c1 char(4000),c2 char(4000),c3 char(4000));
CREATE TABLE
testdb=# 
testdb=# alter table t3 alter c1 set storage plain;
ALTER TABLE
testdb=# alter table t3 alter c2 set storage plain;
ALTER TABLE
testdb=# alter table t3 alter c3 set storage plain;
ALTER TABLE
testdb=# 
testdb=# \d+ t3
                                        Table "public.t3"
 Column |      Type       | Collation | Nullable | Default | Storage | Stats target | Description 
--------+-----------------+-----------+----------+---------+---------+--------------+-------------
 id     | integer         |           |          |         | plain   |              | 
 c1     | character(4000) |           |          |         | plain   |              | 
 c2     | character(4000) |           |          |         | plain   |              | 
 c3     | character(4000) |           |          |         | plain   |              | 

testdb=# 
testdb=# insert into t3 values(1,'11','12','13');
2018-07-28 11:32:07.561 CST [1576] ERROR:  row is too big: size 12040, maximum size 8160
2018-07-28 11:32:07.561 CST [1576] STATEMENT:  insert into t3 values(1,'11','12','13');
ERROR:  row is too big: size 12040, maximum size 8160
testdb=# insert into t3 values(2,rpad('2',4000,'2'),rpad('3',4000,'3'),rpad('4',4000,'4'));
2018-07-28 11:32:08.687 CST [1576] ERROR:  row is too big: size 12040, maximum size 8160
2018-07-28 11:32:08.687 CST [1576] STATEMENT:  insert into t3 values(2,rpad('2',4000,'2'),rpad('3',4000,'3'),rpad('4',4000,'4'));
ERROR:  row is too big: size 12040, maximum size 8160
-- 如果使用PLAIN选项，数据行限制为8160（32Bytes用于头部信息、ItemId和Special space等）
```
### EXTERNAL
```
testdb=# -- EXTERNAL选项，RowSize > 8K
testdb=# drop table if exists t4;
TERNAL;

insert into t4 values(1,'11','12','13');
insert into t4 values(2,rpad('2',4000,'2'),rpad('3',4000,'3'),rpad('4',4000,'4'));DROP TABLE
testdb=# create table t4(id int,c1 char(4000),c2 char(4000),c3 char(4000));
CREATE TABLE
testdb=# 
testdb=# alter table t4 alter c1 set storage EXTERNAL;
ALTER TABLE
testdb=# alter table t4 alter c2 set storage EXTERNAL;
ALTER TABLE
testdb=# alter table t4 alter c3 set storage EXTERNAL;
ALTER TABLE
testdb=# 
testdb=# insert into t4 values(1,'11','12','13');
INSERT 0 1
testdb=# insert into t4 values(2,rpad('2',4000,'2'),rpad('3',4000,'3'),rpad('4',4000,'4'));
INSERT 0 1
testdb=# 
testdb=# checkpoint;
CHECKPOINT
testdb=# 
testdb=# select pg_relation_filepath('t4');
 pg_relation_filepath 
----------------------
 base/16477/26712
(1 row)

[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/26712
00000000  01 00 00 00 30 57 2b db  00 00 00 00 20 00 50 1f  |....0W+..... .P.|
00000010  00 20 04 20 00 00 00 00  a8 9f a4 00 50 9f a4 00  |. . ........P...|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001f50  15 9c 18 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001f60  02 00 04 00 06 08 18 00  02 00 00 00 01 12 a4 0f  |................|
00001f70  00 00 a0 0f 00 00 61 68  00 00 5b 68 00 00 01 12  |......ah..[h....|
00001f80  a4 0f 00 00 a0 0f 00 00  62 68 00 00 5b 68 00 00  |........bh..[h..|
00001f90  01 12 a4 0f 00 00 a0 0f  00 00 63 68 00 00 5b 68  |..........ch..[h|
00001fa0  00 00 00 00 00 00 00 00  14 9c 18 00 00 00 00 00  |................|
00001fb0  00 00 00 00 00 00 00 00  01 00 04 00 06 08 18 00  |................|
00001fc0  01 00 00 00 01 12 a4 0f  00 00 a0 0f 00 00 5e 68  |..............^h|
00001fd0  00 00 5b 68 00 00 01 12  a4 0f 00 00 a0 0f 00 00  |..[h............|
00001fe0  5f 68 00 00 5b 68 00 00  01 12 a4 0f 00 00 a0 0f  |_h..[h..........|
00001ff0  00 00 60 68 00 00 5b 68  00 00 00 00 00 00 00 00  |..`h..[h........|
00002000
-- 在数据表的block没有存储用户数据，实际的存储数据文件使用以下SQL获得：
testdb=# select reltoastrelid from pg_class where relname = 't4';
 reltoastrelid 
---------------
         26715
(1 row)

[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/26715
00000000  01 00 00 00 08 08 2b db  00 00 00 00 28 00 00 08  |......+.....(...|
00000010  00 20 04 20 00 00 00 00  10 98 e0 0f 20 90 e0 0f  |. . ........ ...|
00000020  f0 8f 58 00 00 88 e0 0f  00 00 00 00 00 00 00 00  |..X.............|
00000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000800  14 9c 18 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000810  04 00 03 00 02 08 18 00  5f 68 00 00 00 00 00 00  |........_h......|
00000820  40 1f 00 00 31 32 20 20  20 20 20 20 20 20 20 20  |@...12          |
00000830  20 20 20 20 20 20 20 20  20 20 20 20 20 20 20 20  |                |
*
00000ff0  14 9c 18 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001000  03 00 03 00 02 08 18 00  5e 68 00 00 02 00 00 00  |........^h......|
00001010  30 00 00 00 20 20 20 20  20 20 20 20 00 00 00 00  |0...        ....|
00001020  14 9c 18 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001030  02 00 03 00 02 08 18 00  5e 68 00 00 01 00 00 00  |........^h......|
00001040  40 1f 00 00 20 20 20 20  20 20 20 20 20 20 20 20  |@...            |
00001050  20 20 20 20 20 20 20 20  20 20 20 20 20 20 20 20  |                |
*
00001810  14 9c 18 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001820  01 00 03 00 02 08 18 00  5e 68 00 00 00 00 00 00  |........^h......|
00001830  40 1f 00 00 31 31 20 20  20 20 20 20 20 20 20 20  |@...11          |
00001840  20 20 20 20 20 20 20 20  20 20 20 20 20 20 20 20  |                |
*
00002000  01 00 00 00 30 22 2b db  00 00 00 00 2c 00 d0 07  |....0"+.....,...|
00002010  00 20 04 20 00 00 00 00  10 98 e0 0f e0 97 58 00  |. . ..........X.|
00002020  f0 8f e0 0f 00 88 e0 0f  d0 87 58 00 00 00 00 00  |..........X.....|
00002030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
000027d0  14 9c 18 00 00 00 00 00  00 00 00 00 00 00 01 00  |................|
000027e0  05 00 03 00 02 08 18 00  60 68 00 00 02 00 00 00  |........`h......|
000027f0  30 00 00 00 20 20 20 20  20 20 20 20 00 00 00 00  |0...        ....|
00002800  14 9c 18 00 00 00 00 00  00 00 00 00 00 00 01 00  |................|
00002810  04 00 03 00 02 08 18 00  60 68 00 00 01 00 00 00  |........`h......|
00002820  40 1f 00 00 20 20 20 20  20 20 20 20 20 20 20 20  |@...            |
00002830  20 20 20 20 20 20 20 20  20 20 20 20 20 20 20 20  |                |
*
00002ff0  14 9c 18 00 00 00 00 00  00 00 00 00 00 00 01 00  |................|
00003000  03 00 03 00 02 08 18 00  60 68 00 00 00 00 00 00  |........`h......|
00003010  40 1f 00 00 31 33 20 20  20 20 20 20 20 20 20 20  |@...13          |
00003020  20 20 20 20 20 20 20 20  20 20 20 20 20 20 20 20  |                |
*
000037e0  14 9c 18 00 00 00 00 00  00 00 00 00 00 00 01 00  |................|
000037f0  02 00 03 00 02 08 18 00  5f 68 00 00 02 00 00 00  |........_h......|
00003800  30 00 00 00 20 20 20 20  20 20 20 20 00 00 00 00  |0...        ....|
00003810  14 9c 18 00 00 00 00 00  00 00 00 00 00 00 01 00  |................|
00003820  01 00 03 00 02 08 18 00  5f 68 00 00 01 00 00 00  |........_h......|
00003830  40 1f 00 00 20 20 20 20  20 20 20 20 20 20 20 20  |@...            |
00003840  20 20 20 20 20 20 20 20  20 20 20 20 20 20 20 20  |                |
*
00004000  01 00 00 00 50 3c 2b db  00 00 00 00 28 00 00 08  |....P<+.....(...|
00004010  00 20 04 20 00 00 00 00  10 98 e0 0f 20 90 e0 0f  |. . ........ ...|
00004020  f0 8f 58 00 00 88 e0 0f  00 00 00 00 00 00 00 00  |..X.............|
00004030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00004800  15 9c 18 00 00 00 00 00  00 00 00 00 00 00 02 00  |................|
00004810  04 00 03 00 02 08 18 00  62 68 00 00 00 00 00 00  |........bh......|
00004820  40 1f 00 00 33 33 33 33  33 33 33 33 33 33 33 33  |@...333333333333|
00004830  33 33 33 33 33 33 33 33  33 33 33 33 33 33 33 33  |3333333333333333|
*
00004ff0  15 9c 18 00 00 00 00 00  00 00 00 00 00 00 02 00  |................|
00005000  03 00 03 00 02 08 18 00  61 68 00 00 02 00 00 00  |........ah......|
00005010  30 00 00 00 32 32 32 32  32 32 32 32 00 00 00 00  |0...22222222....|
00005020  15 9c 18 00 00 00 00 00  00 00 00 00 00 00 02 00  |................|
00005030  02 00 03 00 02 08 18 00  61 68 00 00 01 00 00 00  |........ah......|
00005040  40 1f 00 00 32 32 32 32  32 32 32 32 32 32 32 32  |@...222222222222|
00005050  32 32 32 32 32 32 32 32  32 32 32 32 32 32 32 32  |2222222222222222|
*
00005810  15 9c 18 00 00 00 00 00  00 00 00 00 00 00 02 00  |................|
00005820  01 00 03 00 02 08 18 00  61 68 00 00 00 00 00 00  |........ah......|
00005830  40 1f 00 00 32 32 32 32  32 32 32 32 32 32 32 32  |@...222222222222|
00005840  32 32 32 32 32 32 32 32  32 32 32 32 32 32 32 32  |2222222222222222|
*
00006000  01 00 00 00 78 56 2b db  00 00 00 00 2c 00 d0 07  |....xV+.....,...|
00006010  00 20 04 20 00 00 00 00  10 98 e0 0f e0 97 58 00  |. . ..........X.|
00006020  f0 8f e0 0f 00 88 e0 0f  d0 87 58 00 00 00 00 00  |..........X.....|
00006030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
000067d0  15 9c 18 00 00 00 00 00  00 00 00 00 00 00 03 00  |................|
000067e0  05 00 03 00 02 08 18 00  63 68 00 00 02 00 00 00  |........ch......|
000067f0  30 00 00 00 34 34 34 34  34 34 34 34 00 00 00 00  |0...44444444....|
00006800  15 9c 18 00 00 00 00 00  00 00 00 00 00 00 03 00  |................|
00006810  04 00 03 00 02 08 18 00  63 68 00 00 01 00 00 00  |........ch......|
00006820  40 1f 00 00 34 34 34 34  34 34 34 34 34 34 34 34  |@...444444444444|
00006830  34 34 34 34 34 34 34 34  34 34 34 34 34 34 34 34  |4444444444444444|
*
00006ff0  15 9c 18 00 00 00 00 00  00 00 00 00 00 00 03 00  |................|
00007000  03 00 03 00 02 08 18 00  63 68 00 00 00 00 00 00  |........ch......|
00007010  40 1f 00 00 34 34 34 34  34 34 34 34 34 34 34 34  |@...444444444444|
00007020  34 34 34 34 34 34 34 34  34 34 34 34 34 34 34 34  |4444444444444444|
*
000077e0  15 9c 18 00 00 00 00 00  00 00 00 00 00 00 03 00  |................|
000077f0  02 00 03 00 02 08 18 00  62 68 00 00 02 00 00 00  |........bh......|
00007800  30 00 00 00 33 33 33 33  33 33 33 33 00 00 00 00  |0...33333333....|
00007810  15 9c 18 00 00 00 00 00  00 00 00 00 00 00 03 00  |................|
00007820  01 00 03 00 02 08 18 00  62 68 00 00 01 00 00 00  |........bh......|
00007830  40 1f 00 00 33 33 33 33  33 33 33 33 33 33 33 33  |@...333333333333|
00007840  33 33 33 33 33 33 33 33  33 33 33 33 33 33 33 33  |3333333333333333|
*
00008000
[xdb@localhost utf8db]$ 
```

## 三、小结

知识要点：

1. 实际行大小大于PageSize：

    A. Oracle使用行链接方式存储

    B. PG默认使用EXTENDED方式，首先压缩，如果仍不足以存储数据，那么使用out-of-line方式存储

2. PG列数据存储有4种选项，分别是PLAIN/EXTENDED/EXTERNAL/MAIN

## 四、参考文档