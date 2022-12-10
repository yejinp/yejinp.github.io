---
layout: post
title: Oracle vs PostgreSQL，研发注意事项（1）-查询锁表
date: 2022-12-04 22:20:23 +0800
category: PostgreSQL
---

Oracle数据库，查询语句不会锁表，但PostgreSQL在开启事务后，查询数据表会锁表，在试图DROP/TRUNCATE TABLE时会一直等待。
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
-- 提交事务
end;
--------------------------- Session B
-- 开启事务
begin;
-- 查询当前事务号
select txid_current(); 
-- 查询数据表
select count(*) from t1;
--------------------------- Session A
-- 重新回到Session A，删除数据表
drop table t1; -- 这时会一直等待
```

查询数据库锁信息：
```
testdb=# SELECT pid, relname , locktype, mode
testdb-# FROM pg_locks l JOIN pg_class t ON l.relation = t.oid
testdb-#      AND t.relkind = 'r'
testdb-# WHERE t.relname = 't1';
pid  | relname | locktype |        mode       
------+---------+----------+---------------------
1574 | t1      | relation | AccessShareLock
1585 | t1      | relation | AccessExclusiveLock
(2 rows)
```

发现查询t1（1574为Session B的pid）时会持有AccessShareLock，导致drop table一直等待该锁释放后才能执行。

参考：

转自：
[http://blog.itpub.net/6906/viewspace-2158210/](http://blog.itpub.net/6906/viewspace-2158210/)