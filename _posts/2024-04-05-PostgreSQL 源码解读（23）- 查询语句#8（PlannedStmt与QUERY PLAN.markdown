---
layout: post
title: PostgreSQL 源码解读（23）- 查询语句#8（PlannedStmt与QUERY PLAN
date: 2024-04-05 18:20:23 +0800
category: PostgreSQL
---


## 一、测试脚本
```
testdb=# explain 
testdb-# select * from (
testdb(# select t_dwxx.dwmc,t_grxx.grbh,t_grxx.xm,t_jfxx.ny,t_jfxx.je
testdb(# from t_dwxx inner join t_grxx on t_dwxx.dwbh = t_grxx.dwbh
testdb(# inner join t_jfxx on t_grxx.grbh = t_jfxx.grbh
testdb(# where t_dwxx.dwbh IN ('1001')
testdb(# union all
testdb(# select t_dwxx.dwmc,t_grxx.grbh,t_grxx.xm,t_jfxx.ny,t_jfxx.je
testdb(# from t_dwxx inner join t_grxx on t_dwxx.dwbh = t_grxx.dwbh
testdb(# inner join t_jfxx on t_grxx.grbh = t_jfxx.grbh
testdb(# where t_dwxx.dwbh IN ('1002') 
testdb(# ) as ret
testdb-# order by ret.grbh
testdb-# limit 4;
                                              QUERY PLAN                                              
------------------------------------------------------------------------------------------------------
 Limit  (cost=96.80..96.81 rows=4 width=360)
   ->  Sort  (cost=96.80..96.83 rows=14 width=360)
         Sort Key: t_grxx.grbh
         ->  Append  (cost=16.15..96.59 rows=14 width=360)
               ->  Nested Loop  (cost=16.15..48.19 rows=7 width=360)
                     ->  Seq Scan on t_dwxx  (cost=0.00..12.00 rows=1 width=256)
                           Filter: ((dwbh)::text = '1001'::text)
                     ->  Hash Join  (cost=16.15..36.12 rows=7 width=180)
                           Hash Cond: ((t_jfxx.grbh)::text = (t_grxx.grbh)::text)
                           ->  Seq Scan on t_jfxx  (cost=0.00..17.20 rows=720 width=84)
                           ->  Hash  (cost=16.12..16.12 rows=2 width=134)
                                 ->  Seq Scan on t_grxx  (cost=0.00..16.12 rows=2 width=134)
                                       Filter: ((dwbh)::text = '1001'::text)
               ->  Nested Loop  (cost=16.15..48.19 rows=7 width=360)
                     ->  Seq Scan on t_dwxx t_dwxx_1  (cost=0.00..12.00 rows=1 width=256)
                           Filter: ((dwbh)::text = '1002'::text)
                     ->  Hash Join  (cost=16.15..36.12 rows=7 width=180)
                           Hash Cond: ((t_jfxx_1.grbh)::text = (t_grxx_1.grbh)::text)
                           ->  Seq Scan on t_jfxx t_jfxx_1  (cost=0.00..17.20 rows=720 width=84)
                           ->  Hash  (cost=16.12..16.12 rows=2 width=134)
                                 ->  Seq Scan on t_grxx t_grxx_1  (cost=0.00..16.12 rows=2 width=134)
                                       Filter: ((dwbh)::text = '1002'::text)
(22 rows)
```

## 二、对照关系

对照关系详见下图：

![picture](/2024/postgresql/yuanmajiedu/ded7901613fd625d.jpeg "对照关系")


对照关系

结合关系代数的理论和相关术语，可以更好的理解执行计划。

## 三、小结
上一小节已通过分析日志的方式介绍了PlannedStmt的结构，这一小节结合执行计划进行对照解析，可以加深对执行计划的理解和认识。


### 转自：
[来源](https://blog.itpub.net/6906/viewspace-2374893/)