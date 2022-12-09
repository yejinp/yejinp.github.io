---
layout: post
title: PostgreSQL Page页结构解析（2）- 页头和行数据指针
date: 2022-12-08 22:20:23 +0800
category: PostgreSQL
---

本文简单介绍了PG数据页Page中存储的原始内容以及如何阅读它们，包括页头PageHeader和行数据指针ItemId（Line Pointer）。

一、测试数据

```
-- 创建一张表，插入几行数据
drop table if exists t_page;
create table t_page (id int,c1 char(8),c2 varchar(16));
insert into t_page values(1,'1','a');
insert into t_page values(2,'2','b');
insert into t_page values(3,'3','c');
insert into t_page values(4,'4','d');
-- 获取该表对应的数据文件
testdb=# select pg_relation_filepath('t_page');
pg_relation_filepath
----------------------
base/16477/24801
(1 row)
-- Dump数据文件中的数据
[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/24801
00000000  01 00 00 00 88 20 2a 12  00 00 00 00 28 00 60 1f  |..... *.....(.`.|
00000010  00 20 04 20 00 00 00 00  d8 9f 4e 00 b0 9f 4e 00  |. . ......N...N.|
00000020  88 9f 4e 00 60 9f 4e 00  00 00 00 00 00 00 00 00  |..N.`.N.........|
00000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001f60  e5 1b 18 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001f70  04 00 03 00 02 08 18 00  04 00 00 00 13 34 20 20  |.............4  |
00001f80  20 20 20 20 20 05 64 00  e4 1b 18 00 00 00 00 00  |    .d.........|
00001f90  00 00 00 00 00 00 00 00  03 00 03 00 02 08 18 00  |................|
00001fa0  03 00 00 00 13 33 20 20  20 20 20 20 20 05 63 00  |.....3      .c.|
00001fb0  e3 1b 18 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001fc0  02 00 03 00 02 08 18 00  02 00 00 00 13 32 20 20  |.............2  |
00001fd0  20 20 20 20 20 05 62 00  e2 1b 18 00 00 00 00 00  |    .b.........|
00001fe0  00 00 00 00 00 00 00 00  01 00 03 00 02 08 18 00  |................|
00001ff0  01 00 00 00 13 31 20 20  20 20 20 20 20 05 61 00  |.....1      .a.|
00002000
```

二、PageHeader

上一节提到过PageHeaderData，其数据结构如下：
```
typedef struct PageHeaderData
{
/* XXX LSN is member of *any* block, not only page-organized ones */
PageXLogRecPtr pd_lsn; /* LSN: next byte after last byte of xlog
* record for last change to this page */
uint16 pd_checksum; /* checksum */
uint16 pd_flags; /* flag bits, see below */
LocationIndex pd_lower; /* offset to start of free space */
LocationIndex pd_upper; /* offset to end of free space */
LocationIndex pd_special; /* offset to start of special space */
uint16 pd_pagesize_version;
TransactionId pd_prune_xid; /* oldest prunable XID, or zero if none */
ItemIdData pd_linp[1]; /* beginning of line pointer array */
}  PageHeaderData;
```

下面根据数据文件中的数据使用hexdump查看并逐个进行解析。

```
pd_lsn(8bytes)
[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/24801 -s 0 -n 8
00000000  01 00 00 00 88 20 2a 12                          |..... *.|
00000008
```
数据文件的8个Bytes存储的是LSN，其中最开始的4个Bytes是TimelineID，在这里是\x0000 0001（即数字1），后面的4个Bytes是\x122a2088，组合起来LSN为1/122A2088

注意：

A、0000000&0000008是hexdump工具的输出，不是数据内容

B、X86使用小端模式，阅读字节码时注意高低位变换
```
pd_checksum(2bytes)
[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/24801 -s 8 -n 2
00000008  00 00                                            |..|
0000000a
```

checksum为\x0000

pd_flags(2bytes)
```
[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/24801 -s 10 -n 2
0000000a  00 00                                            |..|
0000000c
```

flags为\x0000

```
pd_lower(2bytes)
[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/24801 -s 12 -n 2
0000000c  28 00                                            |(.|
0000000e
```

lower为\x0028，十进制值为40

pd_upper(2bytes)
```
[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/24801 -s 14 -n 2
0000000e  60 1f                                            |`.|
00000010
[xdb@localhost utf8db]$ echo $((0x1f60))
8032
```

upper为\x1f60，十进制为8032
```
pd_special(2bytes)
[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/24801 -s 16 -n 2
00000010  00 20                                            |. |
00000012
```
Special Space为\x2000，十进制值为8192

pd_pagesize_version(2bytes)
```
[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/24801 -s 18 -n 2
00000012  04 20                                            |. |
00000014
```

pagesize_version为\x2004，十进制为8196（即版本4）

pd_prune_xid(4bytes)
```
[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/24801 -s 20 -n 4
00000014  00 00 00 00                                      |....|
00000018
```

prune_xid为\x0000，即0

三、ItemIds

PageHeaderData之后是ItemId数组，每个元素占用的空间为4Bytes，数据结构：
```
typedef struct ItemIdData
{
    unsigned lp_off:15,/* offset to tuple (from start of page) */
    lp_flags:2,/* state of item pointer, see below */
    lp_len:15;/* byte length of tuple */
} ItemIdData;

typedef ItemIdData* ItemId;
```

lp_off
```
[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/24801 -s 24 -n 2
00000018  d8 9f                                            |..|
0000001a
```

取低15位
```
[xdb@localhost utf8db]$ echo $((0x9fd8 & ~$((1<<15))))
8152
```

表示第1个Item（tuple）从8152开始

lp_len
```
[xdb@localhost utf8db]$ hexdump -C $PGDATA/base/16477/24801 -s 26 -n 2
0000001a  4e 00                                            |N.|
0000001c
```

取高15位
```
[xdb@localhost utf8db]$ echo $((0x004e >> 1))
39
```

表示第1个Item（tuple）的大小为39

lp_flags

取第17-16位，01，即1

四、小结

以上简单介绍了如何阅读Raw Datafile，包括页头和数据行指针信息，有兴趣的同学可在此基础上实现自己的“pageinspect"。下一节将介绍数据行信息。


转自：

[http://blog.itpub.net/6906/viewspace-2374922/](http://blog.itpub.net/6906/viewspace-2374922/)