---
layout: post
title: PostgreSQL Page页结构解析（1）-基础
date: 2022-12-07 22:20:23 +0800
category: PostgreSQL
---

    本文简单介绍了PG数据表的存储基础知识以及可用于解析数据页Page内容的pageinspcet插件。

一、PG数据表存储基础
       一般来说，数据表数据物理存储在非易失性存储设备上面，PG也不例外。如下图所示，数据表中的数据存储在N个数据文件中，每个数据文件有N个Page（大小默认为8K，可在编译安装时指定）组成。Page为PG的最小存取单元。

普通数据表存储结构

<h3>数据页（Page）</h3>

数据页Page由页头、数据指针数组（ItemIdData）、可使用的空闲空间（Free Space）、实际数据（Items）和特殊空间（Special Space）组成。

    A、页头存储LSN号、校验位等元数据信息，占用24Bytes
    B、数据指针数组存储指向实际数据的指针，数组中的元素ItemId可理解为相应数据行在Page中的实际开始偏移，数据行指针ItemID由三部分组成，前15位为Page内偏移，中间2位为标志，后面15位为长度，详细解析可参见附录
    C、空闲空间为未使用可分配的空间，ItemID从空闲空间的头部开始分配，Item（数据行）从空闲空间的尾部开始
    D、实际数据为数据的行数据（每种数据类型的存储格式后续再解析）
    E、特殊空间用于存储索引访问使用的数据，不同的访问方法数据不同

二、pageinspect插件
    如何简单快速方便的查看Page中的内容？不同于Oracle各种dump，在PG中可以方便的使用pageinspect插件提供的各种函数查看Page中的内容。

安装
得益于PG良好的扩展性，安装很简单：

#cd $PGSRC/contrib/pageinspect
#make
#make install

简单使用
```
#psql -d testdb
testdb#create extension pageinspect; -- 首次使用需创建Extension
```
-- 创建测试表
```
drop table if exists t_new;
create table t_new (id char(4),c1 varchar(20));
insert into t_new values('1','#');
```
-- 查看page header&item
```
SELECT * FROM page_header(get_raw_page('t_new', 0));
select * from heap_page_items(get_raw_page('t_new',0));
```
Page Header&Item
-- 查看Page中的raw内容
```
select * from get_raw_page('t_new', 0);
```

Page头部内容

Page尾部内容
pageinspect的实现，可以查看pageinspect的源代码，更深入的可以通过直接分析Page中的数据内容进行理解，这部分内容在下一节再行介绍。

参考文档：
https://www.postgresql.org/docs/11/static/storage-page-layout.html
https://www.postgresql.org/docs/11/static/pageinspect.html

附：相关数据结构
头文件：src/include/storage/bufpage.h
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
    ItemIdData pd_linp[FLEXIBLE_ARRAY_MEMBER]; /* line pointer array */
} PageHeaderData;

typedef PageHeaderData *PageHeader;
```
头文件：src/include/storage/itemid.h
```
/*
* An item pointer (also called line pointer) on a buffer page
* In some cases an item pointer is "in use" but does not have any associated
* storage on the page.  By convention, lp_len == 0 in every item pointer
* that does not have storage, independently of its lp_flags state.
*/

typedefstruct ItemIdData

{
    unsignedlp_off:15,/* offset to tuple (from start of page) */
    lp_flags:2,/* state of item pointer, see below */
    lp_len:15;/* byte length of tuple */
}ItemIdData;

typedefItemIdData*ItemId;

/*
* lp_flags has these possible states.  An UNUSED line pointer is available
* for immediate re-use, the other states are not.
*/

#define LP_UNUSED      0      /* unused (should always have lp_len=0) */
#define LP_NORMAL      1      /* used (should always have lp_len>0) */
#define LP_REDIRECT    2      /* HOT redirect (should have lp_len=0) */
#define LP_DEAD        3      /* dead, may or may not have storage */

/*
* Item offsets and lengths are represented by these types when
* they're not actually stored in an ItemIdData.
*/

typedefuint16ItemOffset;
typedefuint16ItemLength;
```

转自：[PostgreSQL Page页结构解析（1）-基础](http://blog.itpub.net/6906/viewspace-2374923/)