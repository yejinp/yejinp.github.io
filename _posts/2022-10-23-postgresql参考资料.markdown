---
layout: post
title: postgresql参考资料
date: 2022-10-23 20:20:23 +0800
category: PostgreSQL
---


* 源代码参考
  - [官方文档](https://www.postgresql.org/docs/15/internals.html)
 
  - [The Internals of PostgreSQL](https://www.interdb.jp/pg/index.html)

  - [postgrespro](https://postgrespro.com/blog/pgsql/3994098)

  - [源码解读1](http://blog.itpub.net/6906/)

  - [源码解读2](https://blog.csdn.net/cuichao1900?type=blog （转自源码解读1）)

  - [PostgreSQL技术爱好者](https://foucus.blog.csdn.net/category_9332424.html)

  - [干货-PostgreSQL数据表文件底层结构布局分析](https://blog.csdn.net/MyySophia/article/details/120724075)

* 索引与性能
  - [Use The Index Luke](https://use-the-index-luke.com/sql/table-of-contents)

* 理论
  - Dan R. K. Ports, and Kevin Grittner, "[Serializable Snapshot Isolation in PostgreSQL](https://drkp.net/papers/ssi-vldb12.pdf)", VDBL 2012

  - Transaction Processing (听说很经典的书，豆瓣评分接近满分10.0分)

  - Abraham Silberschatz, Henry F. Korth, and S. Sudarshan, "[Database System Concepts](https://www.amazon.com/dp/0073523321)", McGraw-Hill Education, ISBN-13: 978-0073523323

  - Thomas M. Connolly, and Carolyn E. Begg, "[Database Systems](https://www.amazon.com/dp/0321523067)", Pearson, ISBN-13: 978-0321523068


* 标准
   - [sql1992标准](https://datacadamia.com/_media/data/type/relation/sql/sql1992.txt)

* PostgreSQL自带工具
  - pg_freespacemap 扩展，用于查看fsm信息

  - pageinspect 扩展，编译安装后，CREATE EXTENSION pageinspect； 后即可使用

  - pg_stats 扩展： This is an extension of PostgreSQL, which contains some customized statistics views.

  - pg_archivecleanup — clean up PostgreSQL WAL archive files


* 辅助工具
  - pg_filedump: 分析pg文件的工具
 
  - pg_hexedit

  - hexdump: 直接文件内容的利器，可以直接查看任何文件的二进制内容

* 其他
  - [预写日志(WAL)介绍](https://www.cnblogs.com/xuwc/p/14037750.html)