* **process	description	reference**


* 一组多进程协助管理数据库集群的进程通常称为 PostgreSQL服务器，其包含下面各种类型的进程：

  - 一个postgres服务器进程是数据库集群管理中所有相关进程的父进程；

  - 每个后台进程处理一个客户端发过来的所有查询和提交的声明；

  - 不同类型的后台进程为数据库管理处理不同特征的工作；

  - 在复制相关的进程中，他们执行流复制；

  - 后台工作进程从9.3版本开始支持，它可以执行任何用户实现的处理。 


 * **进程名及其作用表**

| 进程名 | 描述 | 备注 |
|--|--|--|
| background writer |	这个进程中，定期的基于一些基本的规则，将共享缓存中的脏页写到持久化存储（比如HDD或者SSD）中。<br>在版本9.1及之前，这个进程也负责checkpoint处理 ||
| checkpointer | 在9.2已经之后的版本中，处理checkpoint| |
| autovacuum launcher | 这个处理进程在定期处理vacuum的时候唤醒（更精确的说，它给服务器创建autovacuum工作进程） | |
| WAL writer | 这个进程周期性的将WAL缓存中的WAL数据写到持久化存储中 ||
| statistics collector | 这个进程中，收集统计信息，这些信息用于pg_stat_activity和pg_stat_database||
| logging collector (logger) | 这个进程将错误消息写到日志文件中||
| archiver | 这个进程中执行日志归档工作||

* **进程图示例**

![picture](/2022/postgresql/interdb/fig-2-01.png "processor")


从上图可知，每个客户发过来的链接都会导致创建一个进程，一次客户端使用连接池可以比较好的解决频繁创建进程的问题，频繁创建进程会影响服务器的性能；

* **服务器端程序和用途**

| 服务端程序名 | 用途 | |
|-|-|-|
|initdb | 创建PostgreSQL数据库集群 ||
|pg_archivecleanup | 清理PostgreSQL的WAL归档文件 ||
|pg_checksums | 开启、关闭和检查PostgreSQL数据库集群中的数据校验和功能 ||
|pg_controldata | 展示PostgreSQL数据库集群的控制信息 ||
|pg_ctl | 初始化、启动、停止或者控制一个PostgreSQL服务器||
|pg_resetwal | 重置WAL日志和其他PostgreSQL数据库集群的控制信息 ||
|pg_rewind | 将PostgreSQL数据目录同步到另外一个从这个目录分叉（forked）的数据目录 ||
|pg_test_fsync | 确定PostgreSQL的最快wal_sync_method ||
|pg_test_timing | 衡量超时||
|pg_upgrade | 升级PostgrSQL服务器实例||
|pg_waldump | 展示PostgreSQL中，人类可读的WAL日志翻译||
|postgres | PostgreSQL数据库服务器||
|postmaster | PostgreSQL数据库服务器||

* **客户端程序和用途**

| 客户程序名 | 用途 | |
|-|-|-|
|clusterdb | 建立一个PostgreSQL数据库集群 ||
|createdb | 创建一个PostgreSQL数据库 ||
|createuser | 定义一个新的PostgreSQL用户账号||
|dropdb | 清理一个PostgreSQL数据库 ||
|dropuser | 清理一个PostgreSQL用户账号 ||
|ecpg | 嵌入C预处理到SQL中 ||
|pg_amcheck | 检查一个或者多个数据库中是否由毁坏 ||
|pg_basebackup | 建立一个PostgreSQL集群备份 ||
|pgbench | 运行一个PostgrSQL基准测试 ||
|pg_config | 获取PostgreSQL安装版本的信息 ||
|pg_dump | 从PostgreSQL 数据库中将内容导出到脚本文件或者其他归档文件 ||
|pg_dumpall | 从一个PostgreSQL 数据库集群内容到一个脚本文件 ||
|pg_isready | 检查PostgreSQL服务器的连接状态 ||
|pg_receivewal | 从PostgreSQL服务器获取WAL日志流 ||
|pg_recvlogical | 控制PostgreSQL逻辑解码流 ||
|pg_restore | 从一个由pg_dump创建的归档文件中恢复PostgreSQL数据库 ||
|pg_verifybackup | 检验PostgreSQL集群基础备份的一致性 ||
|psql | PostgreSQL 交互终端 ||
|reindexdb | 重新建立PostgreSQL数据库索引 ||
|vacuumdb | 垃圾回收和分析PostgreSQL数据库 ||

 * **参考**
   - https://www.interdb.jp/pg/pgsql02.html

   - https://www.postgresql.org/docs/current/reference-server.html