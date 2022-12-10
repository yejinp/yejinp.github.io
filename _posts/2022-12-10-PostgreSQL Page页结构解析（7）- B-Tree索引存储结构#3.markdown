---
layout: post
title: PostgreSQL Page页结构解析（7）- B-Tree索引存储结构#3
date: 2022-12-10 09:20:23 +0800
category: PostgreSQL
---

本文简单介绍了在PG数据库B-Tree索引的物理存储结构，包括root index block、branch index block、leaf block index等等相关索引结构信息。

## 一、测试数据
我们继续使用上一节使用的测试数据，这一次我们追加插入>1000行的数据。
```
-- 为方便对比，插入数据前先查看索引元数据页
testdb=# select * from bt_metap('pk_t_index');
 magic  | version | root | level | fastroot | fastlevel | oldest_xact | last_cleanup_num_tuples 
--------+---------+------+-------+----------+-----------+-------------+-------------------------
 340322 |       3 |    1 |     0 |        1 |         0 |           0 |                      -1
(1 row)

testdb=# do $$
testdb$# begin
testdb$#   for i in 19..1020 loop
testdb$#   insert into t_index (id, c1, c2) values (i, '#'||i||'#', '#'||i||'#');
testdb$#   end loop;
testdb$# end $$;
DO
testdb=# select count(*) from t_index;
 count 
-------
  1008
(1 row)
```
## 二、索引存储结构
插入数据后，重新查看索引元数据页信息：
```
testdb=# select * from bt_metap('pk_t_index');
 magic  | version | root | level | fastroot | fastlevel | oldest_xact | last_cleanup_num_tuples 
--------+---------+------+-------+----------+-----------+-------------+-------------------------
 340322 |       3 |    3 |     1 |        3 |         1 |           0 |                      -1
(1 row)
root block从原来的block 1变为block 3，查看block 3的的Special space：

testdb=# select * from bt_page_stats('pk_t_index',3);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------+------------
     3 | r    |          3 |          0 |            13 |      8192 |      8096 |         0 |         0 |    1 |          2
(1 row)
```
type=r，表示root index block，这个block有3个index entries（live_items=3，该index block只是root block（btpo_flags=BTP_ROOT）。下面我们来看看这个block中的index entries：
```
testdb=# select * from bt_page_items('pk_t_index',3);
 itemoffset |  ctid   | itemlen | nulls | vars |          data           
------------+---------+---------+-------+------+-------------------------
          1 | (1,0)   |       8 | f     | f    | 
          2 | (2,53)  |      16 | f     | f    | 7b 01 00 00 00 00 00 00
          3 | (4,105) |      16 | f     | f    | e9 02 00 00 00 00 00 00
(3 rows)
```
root/branch index block存储的是指向其他index block的指针。

第1行，index entries指向第1个index block，由于该block没有left block，因此，itemlen只有8个字节，数据范围为1-\x0000017b（十进制值为379）；

第2行，index entries指向第2个index block，数据范围为380-\x000002e9（745）；

第3行，index entries指向第4个index block，数据范围为大于745的值。

这里有个疑惑，正常来说，root index block中的entries应指向index block，但ctid的值（2,53）和（4,105）指向的却是Heap Table Block，PG11 Beta2的Bug？


In a B-tree leaf page, ctid points to a heap tuple. In an internal page, the block number part of ctid points to another page in the index itself, while the offset part (the second number) is ignored and is usually 1.

```
testdb=# select * from heap_page_items(get_raw_page('t_index',2)) where t_ctid = '(2,53)';
 lp | lp_off | lp_flags | lp_len | t_xmin  | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |                  t_data                  
----+--------+----------+--------+---------+--------+----------+--------+-------------+------------+--------+--------+-------+------------------------------------------
 53 |   5648 |        1 |     43 | 1612755 |      0 |      360 | (2,53) |           3 |       2306 |     24 |        |       | \x7b0100001323333739232020200d2333373923
(1 row)

testdb=# select * from heap_page_items(get_raw_page('t_index',4)) where t_ctid = '(4,105)';
 lp  | lp_off | lp_flags | lp_len | t_xmin  | t_xmax | t_field3 | t_ctid  | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |                  t_data                  
-----+--------+----------+--------+---------+--------+----------+---------+-------------+------------+--------+--------+-------+------------------------------------------
 105 |   3152 |        1 |     43 | 1612755 |      0 |      726 | (4,105) |           3 |       2306 |     24 |        |       | \xe90200001323373435232020200d2337343523
(1 row)
```
回到正题，我们首先看看index block 1的相关数据：
```
testdb=# select * from bt_page_stats('pk_t_index',1);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------+------------
     1 | l    |        367 |          0 |            16 |      8192 |       808 |         0 |         2 |    0 |          1
(1 row)

testdb=# select * from bt_page_items('pk_t_index',1) limit 10;
 itemoffset |  ctid  | itemlen | nulls | vars |          data           
------------+--------+---------+-------+------+-------------------------
          1 | (2,53) |      16 | f     | f    | 7b 01 00 00 00 00 00 00
          2 | (0,1)  |      16 | f     | f    | 02 00 00 00 00 00 00 00
          3 | (0,2)  |      16 | f     | f    | 04 00 00 00 00 00 00 00
          4 | (0,3)  |      16 | f     | f    | 08 00 00 00 00 00 00 00
          5 | (0,4)  |      16 | f     | f    | 10 00 00 00 00 00 00 00
          6 | (0,6)  |      16 | f     | f    | 11 00 00 00 00 00 00 00
          7 | (0,5)  |      16 | f     | f    | 12 00 00 00 00 00 00 00
          8 | (0,8)  |      16 | f     | f    | 13 00 00 00 00 00 00 00
          9 | (0,9)  |      16 | f     | f    | 14 00 00 00 00 00 00 00
         10 | (0,10) |      16 | f     | f    | 15 00 00 00 00 00 00 00
(10 rows)
```
第1个block的Special space，其中type=l，表示leaf index block，btpo_flags=BTP_LEAF表示该block仅仅为leaf index block，block的index entries指向heap table。同时，这个block里面有367个items，右边兄弟block号是2（btpo_next）。

值得注意到，index entries的第1个条目，是最大值\x017b，第2个条目才是最小值，接下来的条目是按顺序存储的其他值。源码的README（src/backend/access/nbtree/README）里面有解释：

On a page that is not rightmost in its tree level, the "high key" is
kept in the page's first item, and real data items start at item 2.
The link portion of the "high key" item goes unused. A page that is
rightmost has no "high key", so data items start with the first item.
Putting the high key at the left, rather than the right, may seem odd,
but it avoids moving the high key as we add data items.

官方文档也有相关解释：

Note that the first item on any non-rightmost page (any page with a non-zero value in the btpo_next field) is the page's “high key”, meaning its data serves as an upper bound on all items appearing on the page, while its ctid field is meaningless. Also, on non-leaf pages, the first real data item (the first item that is not a high key) is a “minus infinity” item, with no actual value in its data field. Such an item does have a valid downlink in its ctid field, however.

下面我们再来看看index block 2&4：
```
testdb=# select * from bt_page_stats('pk_t_index',2);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------+------------
     2 | l    |        367 |          0 |            16 |      8192 |       808 |         1 |         4 |    0 |          1
(1 row)

testdb=# select * from bt_page_items('pk_t_index',2) limit 10;
 itemoffset |  ctid   | itemlen | nulls | vars |          data           
------------+---------+---------+-------+------+-------------------------
          1 | (4,105) |      16 | f     | f    | e9 02 00 00 00 00 00 00
          2 | (2,53)  |      16 | f     | f    | 7b 01 00 00 00 00 00 00
          3 | (2,54)  |      16 | f     | f    | 7c 01 00 00 00 00 00 00
          4 | (2,55)  |      16 | f     | f    | 7d 01 00 00 00 00 00 00
          5 | (2,56)  |      16 | f     | f    | 7e 01 00 00 00 00 00 00
          6 | (2,57)  |      16 | f     | f    | 7f 01 00 00 00 00 00 00
          7 | (2,58)  |      16 | f     | f    | 80 01 00 00 00 00 00 00
          8 | (2,59)  |      16 | f     | f    | 81 01 00 00 00 00 00 00
          9 | (2,60)  |      16 | f     | f    | 82 01 00 00 00 00 00 00
         10 | (2,61)  |      16 | f     | f    | 83 01 00 00 00 00 00 00
(10 rows)

testdb=# select * from bt_page_stats('pk_t_index',4);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------+------------
     4 | l    |        276 |          0 |            16 |      8192 |      2628 |         2 |         0 |    0 |          1
(1 row)

testdb=# select * from bt_page_items('pk_t_index',4) limit 10;
 itemoffset |  ctid   | itemlen | nulls | vars |          data           
------------+---------+---------+-------+------+-------------------------
          1 | (4,105) |      16 | f     | f    | e9 02 00 00 00 00 00 00
          2 | (4,106) |      16 | f     | f    | ea 02 00 00 00 00 00 00
          3 | (4,107) |      16 | f     | f    | eb 02 00 00 00 00 00 00
          4 | (4,108) |      16 | f     | f    | ec 02 00 00 00 00 00 00
          5 | (4,109) |      16 | f     | f    | ed 02 00 00 00 00 00 00
          6 | (4,110) |      16 | f     | f    | ee 02 00 00 00 00 00 00
          7 | (4,111) |      16 | f     | f    | ef 02 00 00 00 00 00 00
          8 | (4,112) |      16 | f     | f    | f0 02 00 00 00 00 00 00
          9 | (4,113) |      16 | f     | f    | f1 02 00 00 00 00 00 00
         10 | (4,114) |      16 | f     | f    | f2 02 00 00 00 00 00 00
(10 rows)
```

相关解析参照index block 1，大同小异，不再累述。

## 三、小结
知识要点：

1、root index block的存储结构：指向branch或leaf index block

2、leaf index block的存储结构：指向heap table block；btpo_next <> 0时，最大值存储在第1个item中

### 转自：
[http://blog.itpub.net/6906/viewspace-2374917/](http://blog.itpub.net/6906/viewspace-2374917/)