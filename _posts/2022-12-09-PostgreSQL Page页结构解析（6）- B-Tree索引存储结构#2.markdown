---
layout: post
title: PostgreSQL Page页结构解析（6）- B-Tree索引存储结构#2
date: 2022-12-09 22:46:23 +0800
category: PostgreSQL
---


本文简单介绍了在PG数据库B-Tree索引的物理存储结构中Special space部分，包括根节点、左右兄弟节点等相关索引结构信息，以及初步探讨了PG在物理存储上如何保证索引的有序性。

## 一、测试数据
测试数据同上一节，索引文件raw data：
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

## 二、索引结构信息
索引物理存储结构在上一节已大体介绍，这里主要介绍索引的结构信息。通过pageinspect插件的bt_page_stats函数可以获得索引结构信息，包括root/leaf page,next & previous page：
```
testdb=# select * from bt_page_stats('pk_t_index',1);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------+------------
     1 | l    |          4 |          0 |            16 |      8192 |      8068 |         0 |         0 |    0 |          3
(1 row)
```
相关的数据结构如下：
```
//---------------------- src/include/access/nbtree.h
/*
 *  BTPageOpaqueData -- At the end of every page, we store a pointer
 *  to both siblings in the tree.  This is used to do forward/backward
 *  index scans.  The next-page link is also critical for recovery when
 *  a search has navigated to the wrong page due to concurrent page splits
 *  or deletions; see src/backend/access/nbtree/README for more info.
 *
 *  In addition, we store the page's btree level (counting upwards from
 *  zero at a leaf page) as well as some flag bits indicating the page type
 *  and status.  If the page is deleted, we replace the level with the
 *  next-transaction-ID value indicating when it is safe to reclaim the page.
 *
 *  We also store a "vacuum cycle ID".  When a page is split while VACUUM is
 *  processing the index, a nonzero value associated with the VACUUM run is
 *  stored into both halves of the split page.  (If VACUUM is not running,
 *  both pages receive zero cycleids.)  This allows VACUUM to detect whether
 *  a page was split since it started, with a small probability of false match
 *  if the page was last split some exact multiple of MAX_BT_CYCLE_ID VACUUMs
 *  ago.  Also, during a split, the BTP_SPLIT_END flag is cleared in the left
 *  (original) page, and set in the right page, but only if the next page
 *  to its right has a different cycleid.
 *
 *  NOTE: the BTP_LEAF flag bit is redundant since level==0 could be tested
 *  instead.
 */

typedef struct BTPageOpaqueData
{
    BlockNumber btpo_prev;      /* left sibling, or P_NONE if leftmost */
    BlockNumber btpo_next;      /* right sibling, or P_NONE if rightmost */
    union
    {
        uint32      level;      /* tree level --- zero for leaf pages */
        TransactionId xact;     /* next transaction ID, if deleted */
    }           btpo;
    uint16      btpo_flags;     /* flag bits, see below */
    BTCycleId   btpo_cycleid;   /* vacuum cycle ID of latest split */
} BTPageOpaqueData;

typedef BTPageOpaqueData *BTPageOpaque;

/* Bits defined in btpo_flags */
#define BTP_LEAF        (1 << 0)    /* leaf page, i.e. not internal page */
#define BTP_ROOT        (1 << 1)    /* root page (has no parent) */
#define BTP_DELETED     (1 << 2)    /* page has been deleted from tree */
#define BTP_META        (1 << 3)    /* meta-page */
#define BTP_HALF_DEAD   (1 << 4)    /* empty, but still in tree */
#define BTP_SPLIT_END   (1 << 5)    /* rightmost page of split group */
#define BTP_HAS_GARBAGE (1 << 6)    /* page has LP_DEAD tuples */
#define BTP_INCOMPLETE_SPLIT (1 << 7)   /* right sibling's downlink is missing */
```
查询结果中，type=l（字母l），表示这个block（page）是leaf block，在这个block中有4个item（live_items=4），没有废弃的items（dead_items=0），没有left sibling（btpo_prev =0）也没有right sibling（btpo_next =0），也就是左右两边都没有同级节点。btpo是一个union，值为0，表示该page为叶子page，btpo_flags值为3即BTP_LEAF | BTP_ROOT，既是叶子page也是根page。

这些信息物理存储在先前介绍过的PageHeader中的special space中，共占用16个字节：
```
testdb=# select * from page_header(get_raw_page('pk_t_index',1));
    lsn     | checksum | flags | lower | upper | special | pagesize | version | prune_xid 
------------+----------+-------+-------+-------+---------+----------+---------+-----------
 1/DB0E5C98 |        0 |     0 |    40 |  8112 |    8176 |     8192 |       4 |         0
(1 row)

testdb=# select 8192+8176;
 ?column? 
----------
    16368
(1 row)

[xdb@localhost utf8db]$ hexdump -C base/16477/26637 -s 16368 -n 16
00003ff0  00 00 00 00 00 00 00 00  00 00 00 00 03 00 00 00  |................|
00004000
```
## 三、索引有序性
我们都知道，B-Tree索引是有序的，下面我们看看在物理存储结构上如何保证有序性。

插入数据，id=18
```
testdb=# select * from page_header(get_raw_page('pk_t_index',1));
    lsn     | checksum | flags | lower | upper | special | pagesize | version | prune_xid 
------------+----------+-------+-------+-------+---------+----------+---------+-----------
 1/DB0E5C98 |        0 |     0 |    40 |  8112 |    8176 |     8192 |       4 |         0
(1 row)

testdb=# -- 插入数据,id=18
testdb=# insert into t_index values(18,'4','d');
INSERT 0 1
testdb=# select * from page_header(get_raw_page('pk_t_index',1));
    lsn     | checksum | flags | lower | upper | special | pagesize | version | prune_xid 
------------+----------+-------+-------+-------+---------+----------+---------+-----------
 1/DB0E6498 |        0 |     0 |    44 |  8096 |    8176 |     8192 |       4 |         0
(1 row)
-- dump索引页
[xdb@localhost utf8db]$ hexdump -C base/16477/26637 -s 8192
00002000  01 00 00 00 98 64 0e db  00 00 00 00 2c 00 a0 1f  |.....d......,...|
00002010  f0 1f 04 20 00 00 00 00  e0 9f 20 00 d0 9f 20 00  |... ...... ... .|
00002020  c0 9f 20 00 b0 9f 20 00  a0 9f 20 00 00 00 00 00  |.. ... ... .....|
00002030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00003fa0  00 00 00 00 05 00 10 00  12 00 00 00 00 00 00 00  |................|
00003fb0  00 00 00 00 04 00 10 00  10 00 00 00 00 00 00 00  |................|
00003fc0  00 00 00 00 03 00 10 00  08 00 00 00 00 00 00 00  |................|
00003fd0  00 00 00 00 02 00 10 00  04 00 00 00 00 00 00 00  |................|
00003fe0  00 00 00 00 01 00 10 00  02 00 00 00 00 00 00 00  |................|
00003ff0  00 00 00 00 00 00 00 00  00 00 00 00 03 00 00 00  |................|
00004000
```
插入数据，id=17
```
testdb=# -- 插入数据,id=17
testdb=# insert into t_index values(17,'4','d');
INSERT 0 1
testdb=# checkpoint;
CHECKPOINT
testdb=# -- 查看索引数据页头数据
testdb=# select * from page_header(get_raw_page('pk_t_index',1));
    lsn     | checksum | flags | lower | upper | special | pagesize | version | prune_xid 
------------+----------+-------+-------+-------+---------+----------+---------+-----------
 1/DB0E6808 |        0 |     0 |    48 |  8080 |    8176 |     8192 |       4 |         0
(1 row)
-- dump索引页
[xdb@localhost utf8db]$ hexdump  -C base/16477/26637 -s 8192
00002000  01 00 00 00 08 68 0e db  00 00 00 00 30 00 90 1f  |.....h......0...|
00002010  f0 1f 04 20 00 00 00 00  e0 9f 20 00 d0 9f 20 00  |... ...... ... .|
00002020  c0 9f 20 00 b0 9f 20 00  90 9f 20 00 a0 9f 20 00  |.. ... ... ... .|
00002030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00003f90  00 00 00 00 06 00 10 00  11 00 00 00 00 00 00 00  |................|
00003fa0  00 00 00 00 05 00 10 00  12 00 00 00 00 00 00 00  |................|
00003fb0  00 00 00 00 04 00 10 00  10 00 00 00 00 00 00 00  |................|
00003fc0  00 00 00 00 03 00 10 00  08 00 00 00 00 00 00 00  |................|
00003fd0  00 00 00 00 02 00 10 00  04 00 00 00 00 00 00 00  |................|
00003fe0  00 00 00 00 01 00 10 00  02 00 00 00 00 00 00 00  |................|
00003ff0  00 00 00 00 00 00 00 00  00 00 00 00 03 00 00 00  |................|
00004000
```
索引的数据区，并没有按照大小顺序排序，\x11（17）在\x12（18）的后面（从尾部开始往前），但在索引页的头部ItemId区域是有序的，第5个ItemId（\x00209f90）指向的是17，而第6个ItemId（\x00209fa0）指向的是18，通过索引数据指针的有序保证索引有序性。

## 四、小结
小结一下，知识要点：

1、Special Space：介绍了索引PageHeaderData的Special Space的存储结构，通过Special Space可以获得B-Tree的root、左右sibling等信息；

2、有序性：索引Page通过索引项指针保证有序性。

## 最后：

Don’t Be Afraid To Explore Beneath The Surface

![picture](/2022/postgresql/yuanmajiedu/59ccf58cd00f8e43.jpeg "source code")

Captain Nemo at the helm

> Professor Aronnax risked his life and career to find the elusive Nautilus and to join Captain Nemo on a long series of amazing underwater adventures. We should do the same: Don’t be afraid to dive underwater – inside and underneath the tools, languages and technologies that you use every day. You may know all about how to use Postgres, but do you really know how Postgres itself works internally? Take a look inside; before you know it, you’ll be on an underwater adventure of your own.
> Studying the Computer Science at work behind the scenes of our applications isn’t just a matter of having fun, it’s part of being a good developer. As software development tools improve year after year, and as building web sites and mobile apps becomes easier and easier, we shouldn’t loose sight of the Computer Science we depend on. We’re all standing on the shoulders of giants – people like Lehman and Yao, and the open source developers who used their theories to build Postgres. Don’t take the tools you use everyday for granted – take a look inside them! You’ll become a wiser developer and you’ll find insights and knowledge you could never have imagined before.

### 转自：
[http://blog.itpub.net/6906/viewspace-2374918/](http://blog.itpub.net/6906/viewspace-2374918/)