---
layout: post
title: Oracle vs PostgreSQL，研发注意事项（3）- 事务回滚之UPDATE操作解析
date: 2022-12-06 22:20:23 +0800
category: PostgreSQL
---

Oracle事务的回滚，通过回滚段保存原有数据实现，但，PG没有回滚段！以下以Update操作为例，说明PG实现机制上存在的空间暴涨问题。

在执行Update时，Oracle就地更新，如出现原block空间不足的情况，通过link的方式链接至新block上（不精确，大体表述）；PG的Update，不是原地更新，而是保留原有数据，通过新增新的tuple（数据行）保存新增数据，原有数据通过Vacuum机制清理。Vacuum机制需要满足MVCC（多版本并发控制）的要求，在某些情况下，不会清理“垃圾”数据，在事务繁忙的时候导致会导致数据表空间不断增长。
```
--------------------------- Session A
-- 开启事务
begin;

-- 查询当前事务
select txid_current();
txid_current
--------------
      1500987
(1 row)

-- 什么都不做，会导致Vacuum不能清理“垃圾”数据
--------------------------- Session B
-- 开启事务
begin;
select txid_current();
txid_current
--------------
      1500988
(1 row)
-- 创建表&插入100数据
drop table if exists t1;
create table t1(id int,c1 varchar(50));
insert into t1 select generate_series(1,100),'#TESTDATA#';
------------------- 以上操作省略输出

select txid_current();
txid_current
--------------
      1500988
(1 row)
-- 提交事务
end;

-- 查看数据表
select ctid, xmin, xmax, cmin, cmax,id from t1;
testdb=# select ctid, xmin, xmax, cmin, cmax,id from t1;
  ctid  |  xmin  | xmax | cmin | cmax | id 
---------+---------+------+------+------+-----
(0,1)  | 1500988 |    0 |    4 |    4 |  1
(0,2)  | 1500988 |    0 |    4 |    4 |  2
(0,3)  | 1500988 |    0 |    4 |    4 |  3
(0,4)  | 1500988 |    0 |    4 |    4 |  4
......

-- 查看数据占用空间
\set v_tablename t1
SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
testdb=# SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
pg_size_pretty
--------------
8192 bytes
(1 row)

-- 使用pgbench进行压力测试，不断更新数据
cat update.sql
\set rowid random(1,100)
begin;
update t1 set c1=c1||:rowid where id= :rowid;
end;
pgbench -c 2 -C -f ./update.sql -j 1 -n -T 600 -U xdb testdb

-- 一段时间后查看数据占用空间

SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
testdb=# SELECT pg_size_pretty( pg_total_relation_size(:'v_tablename') );
pg_size_pretty
----------------
  1344 kB
(1 row)
```

从原来的8192 Bytes变成了1344 KB，空间“暴涨”。

究其原因，是因为PG的MVCC实现机制导致的：如果存在某个事务，在更新数据前开启，那么更新数据时前后的数据都要存储，无论更新多少次都要存储！

转自：
[http://blog.itpub.net/6906/viewspace-2158216/](http://blog.itpub.net/6906/viewspace-2158216/)