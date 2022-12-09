---
layout: post
title: PostgreSQL Page页结构解析（3）- 行数据
date: 2022-12-09 22:20:23 +0800
category: PostgreSQL
---


本文介绍了PG数据页Page中存储的原始内容以及如何阅读它们，这一节主要介绍行数据（Items）。

## 一、测试数据

详见上一节，数据文件中的内容如下：
```
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

## 二、Items（Tuples）

每个Tuple包括两部分，第一部分是Tuple头部信息，第二部分是实际的数据。

### 1、HeapTupleHeader

相关数据结构如下：
```
//--------------------- src/include/storage/off.h
/*
* OffsetNumber:
*
* this is a 1-based index into the linp (ItemIdData) array in the
* header of each disk page.
*/
typedef uint16 OffsetNumber;

//--------------------- src/include/storage/block.h
/*
* BlockId:
*
* this is a storage type for BlockNumber.  in other words, this type
* is used for on-disk structures (e.g., in HeapTupleData) whereas
* BlockNumber is the type on which calculations are performed (e.g.,
* in access method code).
*
* there doesn't appear to be any reason to have separate types except
* for the fact that BlockIds can be SHORTALIGN'd (and therefore any
* structures that contains them, such as ItemPointerData, can also be
* SHORTALIGN'd).  this is an important consideration for reducing the
* space requirements of the line pointer (ItemIdData) array on each
* page and the header of each heap or index tuple, so it doesn't seem
* wise to change this without good reason.
*/
typedef struct BlockIdData
{
    uint16      bi_hi;
    uint16      bi_lo;
} BlockIdData;

typedef BlockIdData *BlockId; /* block identifier */

//--------------------- src/include/storage/itemptr.h
/*
  * ItemPointer:
  *
  * This is a pointer to an item within a disk page of a known file
  * (for example, a cross-link from an index to its parent table).
  * blkid tells us which block, posid tells us which entry in the linp
  * (ItemIdData) array we want.
  *
  * Note: because there is an item pointer in each tuple header and index
  * tuple header on disk, it's very important not to waste space with
  * structure padding bytes.  The struct is designed to be six bytes long
  * (it contains three int16 fields) but a few compilers will pad it to
  * eight bytes unless coerced.  We apply appropriate persuasion where
  * possible.  If your compiler can't be made to play along, you'll waste
  * lots of space.
  */
 typedef struct ItemPointerData
 {
     BlockIdData ip_blkid;
     OffsetNumber ip_posid;
 }

//--------------------- src/include/access/htup_details.h
typedef struct HeapTupleFields
{
    TransactionId t_xmin;       /* inserting xact ID */
    TransactionId t_xmax;       /* deleting or locking xact ID */
    union
    {
        CommandId   t_cid;      /* inserting or deleting command ID, or both */
        TransactionId t_xvac;   /* old-style VACUUM FULL xact ID */
    }           t_field3;
} HeapTupleFields;

typedef struct DatumTupleFields
{
    int32       datum_len_;     /* varlena header (do not touch directly!) */

    int32       datum_typmod;   /* -1, or identifier of a record type */

    Oid         datum_typeid;   /* composite type OID, or RECORDOID */

    /*
     * datum_typeid cannot be a domain over composite, only plain composite,
     * even if the datum is meant as a value of a domain-over-composite type.
     * This is in line with the general principle that CoerceToDomain does not
     * change the physical representation of the base type value.
     *
     * Note: field ordering is chosen with thought that Oid might someday
     * widen to 64 bits.
     */
} DatumTupleFields;

struct HeapTupleHeaderData
{
    union
    {
        HeapTupleFields t_heap;
        DatumTupleFields t_datum;
    }           t_choice;

    ItemPointerData t_ctid;     /* current TID of this or newer tuple (or a
                                 * speculative insertion token) */

    /* Fields below here must match MinimalTupleData! */

#define FIELDNO_HEAPTUPLEHEADERDATA_INFOMASK2 2
    uint16      t_infomask2;    /* number of attributes + various flags */

#define FIELDNO_HEAPTUPLEHEADERDATA_INFOMASK 3
    uint16      t_infomask;     /* various flag bits, see below */

#define FIELDNO_HEAPTUPLEHEADERDATA_HOFF 4
    uint8       t_hoff;         /* sizeof header incl. bitmap, padding */

    /* ^ - 23 bytes - ^ */

#define FIELDNO_HEAPTUPLEHEADERDATA_BITS 5
    bits8       t_bits[FLEXIBLE_ARRAY_MEMBER];  /* bitmap of NULLs */

    /* MORE DATA FOLLOWS AT END OF STRUCT */
};
```
结构体展开，详见下表：
```
Field           Type            Length  Offset  Description
t_xmin          TransactionId   4 bytes 0       insert XID stamp
t_xmax          TransactionId   4 bytes 4       delete XID stamp
t_cid           CommandId       4 bytes 8       insert and/or delete CID stamp (overlays with t_xvac)
t_xvac          TransactionId   4 bytes 8       XID for VACUUM operation moving a row version
t_ctid          ItemPointerData 6 bytes 12      current TID of this or newer row version
t_infomask2     uint16          2 bytes 18      number of attributes, plus various flag bits
t_infomask      uint16          2 bytes 20      various flag bits
t_hoff          uint8           1 byte  22      offset to user data
//注意：t_cid和t_xvac为联合体，共用存储空间
```
从上一节我们已经得出第1个Tuple的偏移为8152，下面使用hexdump对其中的数据逐个解析：
t_xmin
```
[xdb@localhost ~]$ hexdump -C $PGDATA/base/16477/24801 -s 8152 -n 4
00001fd8  e2 1b 18 00                                       |....|
00001fdc
[xdb@localhost ~]$ echo $((0x00181be2))
1580002
```
t_xmax
```
[xdb@localhost ~]$ hexdump -C $PGDATA/base/16477/24801 -s 8156 -n 4
00001fdc  00 00 00 00                                       |....|
00001fe0
```
t_cid/t_xvac
```
[xdb@localhost ~]$ hexdump -C $PGDATA/base/16477/24801 -s 8160 -n 4
00001fe0  00 00 00 00                                       |....|
00001fe4
```
t_ctid
```
[xdb@localhost ~]$ hexdump -C $PGDATA/base/16477/24801 -s 8164 -n 6
00001fe4  00 00 00 00 01 00                                 |......|
00001fea
```
//ip_blkid=\x0000，即blockid=0
//ip_posid=\x0001，即posid=1，第1个tuple
t_infomask2
```
[xdb@localhost ~]$ hexdump -C $PGDATA/base/16477/24801 -s 8170 -n 2
00001fea  03 00                                             |..|
00001fec

//t_infomask2=\x0003，3代表什么意思？我们看看t_infomask2的说明
 /*
  * information stored in t_infomask2:
  */

 #define HEAP_NATTS_MASK 0x07FF /* 11 bits for number of attributes */
 /* bits 0x1800 are available */
 #define HEAP_KEYS_UPDATED 0x2000 /* tuple was updated and key cols
  * modified, or tuple deleted */
 #define HEAP_HOT_UPDATED 0x4000 /* tuple was HOT-updated */
 #define HEAP_ONLY_TUPLE 0x8000 /* this is heap-only tuple */
 #define HEAP2_XACT_MASK 0xE000 /* visibility-related bits */
//根把十六进制值转换为二进制显示
     11111111111 #define HEAP_NATTS_MASK         0x07FF 
  10000000000000 #define HEAP_KEYS_UPDATED       0x2000  
 100000000000000 #define HEAP_HOT_UPDATED        0x4000  
1000000000000000 #define HEAP_ONLY_TUPLE         0x8000  
1110000000000000 #define HEAP2_XACT_MASK         0xE000 
1111111111111110 #define SpecTokenOffsetNumber       0xfffe
//前（低）11位为属性的个数，3意味着有3个属性（字段）
```
t_infomask
```
[xdb@localhost ~]$ hexdump -C $PGDATA/base/16477/24801 -s 8172 -n 2
00001fec  02 08                                             |..|
00001fee
[xdb@localhost ~]$ echo $((0x0802))
2050
[xdb@localhost ~]$ echo "obase=2;2050"|bc
100000000010

//t_infomask=\x0802，十进制值为2050，二进制值为100000000010
//t_infomask说明
               1 #define HEAP_HASNULL            0x0001  /* has null attribute(s) */
              10 #define HEAP_HASVARWIDTH        0x0002  /* has variable-width attribute(s) */
             100 #define HEAP_HASEXTERNAL        0x0004  /* has external stored attribute(s) */
            1000 #define HEAP_HASOID             0x0008  /* has an object-id field */
           10000 #define HEAP_XMAX_KEYSHR_LOCK   0x0010  /* xmax is a key-shared locker */
          100000 #define HEAP_COMBOCID           0x0020  /* t_cid is a combo cid */
         1000000 #define HEAP_XMAX_EXCL_LOCK     0x0040  /* xmax is exclusive locker */
        10000000 #define HEAP_XMAX_LOCK_ONLY     0x0080  /* xmax, if valid, is only a locker */
                    /* xmax is a shared locker */
                 #define HEAP_XMAX_SHR_LOCK  (HEAP_XMAX_EXCL_LOCK | HEAP_XMAX_KEYSHR_LOCK)
                 #define HEAP_LOCK_MASK  (HEAP_XMAX_SHR_LOCK | HEAP_XMAX_EXCL_LOCK | \
                          HEAP_XMAX_KEYSHR_LOCK)
       100000000 #define HEAP_XMIN_COMMITTED     0x0100  /* t_xmin committed */
      1000000000 #define HEAP_XMIN_INVALID       0x0200  /* t_xmin invalid/aborted */
                 #define HEAP_XMIN_FROZEN        (HEAP_XMIN_COMMITTED|HEAP_XMIN_INVALID)
     10000000000 #define HEAP_XMAX_COMMITTED     0x0400  /* t_xmax committed */
    100000000000 #define HEAP_XMAX_INVALID       0x0800  /* t_xmax invalid/aborted */
   1000000000000 #define HEAP_XMAX_IS_MULTI      0x1000  /* t_xmax is a MultiXactId */
  10000000000000 #define HEAP_UPDATED            0x2000  /* this is UPDATEd version of row */
 100000000000000 #define HEAP_MOVED_OFF          0x4000  /* moved to another place by pre-9.0
                                         * VACUUM FULL; kept for binary
                                         * upgrade support */
1000000000000000 #define HEAP_MOVED_IN           0x8000  /* moved from another place by pre-9.0
                                         * VACUUM FULL; kept for binary
                                         * upgrade support */
                 #define HEAP_MOVED (HEAP_MOVED_OFF | HEAP_MOVED_IN)
1111111111110000 #define HEAP_XACT_MASK          0xFFF0  /* visibility-related bits */
//\x0802，二进制100000000010表示第2位和第12位为1，
//意味着存在可变长属性（HEAP_HASVARWIDTH），XMAX无效（HEAP_XMAX_INVALID）
```
t_hoff
```
[xdb@localhost ~]$ hexdump -C $PGDATA/base/16477/24801 -s 8174 -n 1
00001fee  18                                                |.|
00001fef
[xdb@localhost ~]$ echo $((0x18))
24
//用户数据开始偏移为24，即8152+24
```

### 2、Tuple
说完了Tuple的头部数据，接下来我们看看实际的数据存储。上一节我们得到Tuple总的长度是39，计算得到数据大小为39-24=15。

```
[xdb@localhost ~]$ hexdump -C $PGDATA/base/16477/24801 -s 8176 -n 15
00001ff0  01 00 00 00 13 31 20 20  20 20 20 20 20 05 61     |.....1       .a|
00001fff
```
回顾我们的表结构：
```
create table t_page (id int,c1 char(8),c2 varchar(16));
```
第1个字段为int，第2个字段为定长字符，第3个字段为变长字符。
相应的数据：
```
id=\x00000001，数字1
c1=\x133120202020202020，字符串，无需高低位变换，第1个字节\x13为标志位，后面是字符'1'+7个空格
c2=\x0561，字符串，第1个字节\x05为标志位，后面是字符'a'
```
## 三、小结

以上简单介绍了如何阅读Raw Datafile中的数据行信息，包括Tuple头部信息和用户数据。在空间使用上面，可以看PG到为了进行实际的数据查询和MVCC等机制，数据库添加了很多的额外信息，实际占用的空间大小比实际的数据要大很多，如果使用列式存储应可有效的压缩空间。

#### 转自：
[http://blog.itpub.net/6906/viewspace-2374921/](http://blog.itpub.net/6906/viewspace-2374921/)