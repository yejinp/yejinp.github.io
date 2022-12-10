---
layout: post
title: PostgreSQL Page页结构解析（5）- B-Tree索引存储结构#1
date: 2022-12-09 22:20:23 +0800
category: PostgreSQL
---

本文简单介绍了在PG数据库B-Tree索引的物理存储内容。

## 一、测试数据
创建数据表，插入数据并创建索引。
```
testdb=# -- 创建一张表，插入几行数据
testdb=# drop table if exists t_index;
 t_index values(16,'4','d');

-- 创建索引
alter table t_index add constraint pk_t_index primary key(id);DROP TABLE
testdb=# create table t_index (id int,c1 char(8),c2 varchar(16));
CREATE TABLE
testdb=# insert into t_index values(2,'1','a');
INSERT 0 1
testdb=# insert into t_index values(4,'2','b');
INSERT 0 1
testdb=# insert into t_index values(8,'3','c');
INSERT 0 1
testdb=# insert into t_index values(16,'4','d');
INSERT 0 1
testdb=# 
testdb=# -- 创建索引
testdb=# alter table t_index add constraint pk_t_index primary key(id);
ALTER TABLE
testdb=# -- 索引物理文件
testdb=# SELECT pg_relation_filepath('pk_t_index');
 pg_relation_filepath 
----------------------
 base/16477/26637
(1 row)
```
索引文件raw data
```
[xdb@localhost utf8db]$ hexdump -C base/16477/26637
00000000  01 00 00 00 20 5d 0e db  00 00 00 00 40 00 f0 1f  |.... ]......@...|
00000010  f0 1f 04 20 00 00 00 00  62 31 05 00 03 00 00 00  |... ....b1......|
00000020  01 00 00 00 00 00 00 00  01 00 00 00 00 00 00 00  |................|
00000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 f0 bf  |................|
00000040  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001ff0  00 00 00 00 00 00 00 00  00 00 00 00 08 00 00 00  |................|
00002000  01 00 00 00 98 5c 0e db  00 00 00 00 28 00 b0 1f  |.....\......(...|
00002010  f0 1f 04 20 00 00 00 00  e0 9f 20 00 d0 9f 20 00  |... ...... ... .|
00002020  c0 9f 20 00 b0 9f 20 00  b0 9f 20 00 00 00 00 00  |.. ... ... .....|
00002030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00003fb0  00 00 00 00 04 00 10 00  10 00 00 00 00 00 00 00  |................|
00003fc0  00 00 00 00 03 00 10 00  08 00 00 00 00 00 00 00  |................|
00003fd0  00 00 00 00 02 00 10 00  04 00 00 00 00 00 00 00  |................|
00003fe0  00 00 00 00 01 00 10 00  02 00 00 00 00 00 00 00  |................|
00003ff0  00 00 00 00 00 00 00 00  00 00 00 00 03 00 00 00  |................|
00004000
```
## 二、B-Tree索引物理存储
我们可以通过pageinspect插件查看索引的存储结构。

Page 0是索引元数据页：
```
testdb=# -- 查看索引页头数据
testdb=#  select * from page_header(get_raw_page('pk_t_index',0));
    lsn     | checksum | flags | lower | upper | special | pagesize | version | prune_xid 
------------+----------+-------+-------+-------+---------+----------+---------+-----------
 1/DB0E5D20 |        0 |     0 |    64 |  8176 |    8176 |     8192 |       4 |         0
(1 row)
testdb=# -- 查看索引元数据页
testdb=# select * from bt_metap('pk_t_index');
 magic  | version | root | level | fastroot | fastlevel | oldest_xact | last_cleanup_num_tuples 
--------+---------+------+-------+----------+-----------+-------------+-------------------------
 340322 |       3 |    1 |     0 |        1 |         0 |           0 |                      -1
(1 row)
```
root=1提示root页在第1页，通过page_header查看页头数据：
```
testdb=# select * from page_header(get_raw_page('pk_t_index',1));
    lsn     | checksum | flags | lower | upper | special | pagesize | version | prune_xid 
------------+----------+-------+-------+-------+---------+----------+---------+-----------
 1/DB0E5C98 |        0 |     0 |    40 |  8112 |    8176 |     8192 |       4 |         0
(1 row)
```
每个索引entries结构为IndexTupleData+Bitmap+Value，其中IndexTupleData占8个字节，Bitmap占4个字节，Value占4字节，合计占用16个字节，数据结构如下：
```
 /*
  * Index tuple header structure
  *
  * All index tuples start with IndexTupleData.  If the HasNulls bit is set,
  * this is followed by an IndexAttributeBitMapData.  The index attribute
  * values follow, beginning at a MAXALIGN boundary.
  *
  * Note that the space allocated for the bitmap does not vary with the number
  * of attributes; that is because we don't have room to store the number of
  * attributes in the header.  Given the MAXALIGN constraint there's no space
  * savings to be had anyway, for usual values of INDEX_MAX_KEYS.
  */
 
 typedef struct IndexTupleData
 {
     ItemPointerData t_tid;      /* reference TID to heap tuple */
 
     /* ---------------
      * t_info is laid out in the following fashion:
      *
      * 15th (high) bit: has nulls
      * 14th bit: has var-width attributes
      * 13th bit: AM-defined meaning
      * 12-0 bit: size of tuple
      * ---------------
      */
 
     unsigned short t_info;      /* various info about tuple */
 
 } IndexTupleData;               /* MORE DATA FOLLOWS AT END OF STRUCT */
 
 typedef IndexTupleData *IndexTuple;
 
 typedef struct IndexAttributeBitMapData
 {
     bits8       bits[(INDEX_MAX_KEYS + 8 - 1) / 8];
 }           IndexAttributeBitMapData;
 
 typedef IndexAttributeBitMapData * IndexAttributeBitMap;
```
通过bt_page_items函数查看索引entries：
```
testdb=# select * from bt_page_items('pk_t_index',1);
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 02 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 04 00 00 00 00 00 00 00
          3 | (0,3) |      16 | f     | f    | 08 00 00 00 00 00 00 00
          4 | (0,4) |      16 | f     | f    | 10 00 00 00 00 00 00 00
(4 rows)
```
相应的物理索引文件内容：
```
[xdb@localhost utf8db]$ hexdump -C base/16477/26637
00000000  01 00 00 00 20 5d 0e db  00 00 00 00 40 00 f0 1f  |.... ]......@...|
00000010  f0 1f 04 20 00 00 00 00  62 31 05 00 03 00 00 00  |... ....b1......|
00000020  01 00 00 00 00 00 00 00  01 00 00 00 00 00 00 00  |................|
00000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 f0 bf  |................|
00000040  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
-- 以上为元数据页的头部数据
*
00001ff0  00 00 00 00 00 00 00 00  00 00 00 00 08 00 00 00  |................|
00002000  01 00 00 00 98 5c 0e db  00 00 00 00 28 00 b0 1f  |.....\......(...|
00002010  f0 1f 04 20 00 00 00 00  e0 9f 20 00 d0 9f 20 00  |... ...... ... .|
00002020  c0 9f 20 00 b0 9f 20 00  b0 9f 20 00 00 00 00 00  |.. ... ... .....|
00002030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
-- 以上为索引数据Page 0的头部数据
*
00003fb0  00 00 00 00 04 00 10 00  10 00 00 00 00 00 00 00  |................|
00003fc0  00 00 00 00 03 00 10 00  08 00 00 00 00 00 00 00  |................|
00003fd0  00 00 00 00 02 00 10 00  04 00 00 00 00 00 00 00  |................|
00003fe0  00 00 00 00 01 00 10 00  02 00 00 00 00 00 00 00  |................|
00003ff0  00 00 00 00 00 00 00 00  00 00 00 00 03 00 00 00  |................|
00004000
-- 以上为索引数据Page 0的索引数据
```
**ItemPointerData**
```
[xdb@localhost utf8db]$ hexdump -C base/16477/26637 -s 16304 -n 6
00003fb0  00 00 00 00 04 00                                 |......|
00003fb6
-- blockid=\x0000，offset=\x0004
```
**t_info**
```
[xdb@localhost utf8db]$ hexdump -C base/16477/26637 -s 16310 -n 2
00003fb6  10 00                                             |..|
00003fb8
t_info=\x0010，即16，表示tuple（索引项）大小为16个字节
```
## 三、小结
小结一下，主要有以下几点：

1. 数据存储：索引数据页头和与普通数据表页头一样的结构，占用24个字节，ItemIds占用4个字节；

2. 索引entries：结构为IndexTupleData+Bitmap+Value；

3. 内容查看：可通过pageinspect插件，推荐通过hexdump物理文件查看，有助于理解数据结构和数据的底层存储格式

更详细的B-Tree物理/逻辑结构在下一节解析。

#### 转自：
[http://blog.itpub.net/6906/viewspace-2374919/](http://blog.itpub.net/6906/viewspace-2374919/)