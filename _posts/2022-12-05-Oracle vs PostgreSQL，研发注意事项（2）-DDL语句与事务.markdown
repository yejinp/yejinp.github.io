---
layout: post
title: Oracle vs PostgreSQL，研发注意事项（2）-DDL语句与事务
date: 2022-12-05 22:20:23 +0800
category: PostgreSQL
---

Oracle执行 DDL语句如CREATE, DROP, RENAME, or ALTER时，会隐式提交事务；PG在执行这类语句时，不会提交事务，需显式提交。

```
--------------------------- Session A
drop table if exists t1;
-- 开启事务
begin;
-- 查询当前事务号
select txid_current(); 
-- 创建表&插入100w数据
create table t1(id int,c1 varchar(20));
-- 查询当前事务号
select txid_current(); 
insert into t1 select generate_series(1,1000000),'#TESTDATA#';
rollback;-- 回滚事务
select count(*) from t1;
```

提示：

```
ERROR: relation "t1" does not exist
LINE 1: select count(*) from t1;
```
如果是Oracle数据库，创建数据表成功后会隐式提交事务，插入数据后回滚，数据表仍会存在。

参考：

转自：
[http://blog.itpub.net/6906/viewspace-2158215/](http://blog.itpub.net/6906/viewspace-2158215/)