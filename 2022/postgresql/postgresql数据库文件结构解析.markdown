**数据库集群规划**

|  目录（文件）名  |  描述 |  |
|---------|------|----|
| PG_VERSION      |  包含PostgreSQL主版本信息    |  |
| pg_hba.conf      | 控制PostgreSQL客户授权的文件 |  |
| pg_ident.conf      | 控制PostgreSQL用户名映射的文件 |  |
| postgresql.conf |设置配置参数的文件 | |
| postgresql_auto.conf | 用来存放由命令ALTER SYSTEM（9.4及或之后的版本）配置参数 | |
| postmastr.opts | 记录服务器命令启动选项的文件 | |
| base | 子目录包括每个数据库的子目录 | |
| global | 子目录包括集群范围的表，比如 pg_database和pg_control | |
| pg_commit_ts | 子目录包括事务提交时间戳数据。 版本9.5及其之后版本中 | |
| pg_clog/(版本9.6或其之前版本) | 子目录包括事提交状态数据。在10.0版本中重命名未pg_xact ||
| pg_dynshmem | 9.4或者之后的版本中，子目录包含用于动态共享内存系统的文件， ||
| pg_logical | 9.4或者之后的版本中，子目录包含用于逻辑解码的数据 ||
| pg_multixact | 子目录包含错事务状态数据（用于共享行锁）||
| pg_notify | 子目录包括LISTEN/NOTIFY状态数据 ||
| pg_repslot | 9.4或者之后的版本中，子目录包含replication slot 数据||
| pg_serial | 9.1或者之后的版本中，包含序列化事务提交的信息 ||
| pg_snapshots | 子目录包含导出快照（9.2及之后的版本）。PostgreSQL的函数 pg_export_snapshot在这个目录中创建快照选项文件 ||
| pg_stat | 包含用于统计子系统的永久文件||
| pg_stat_tmp | 包含子事务状态数据||
| pg_tblspc | 包括表空间的符号链接 ||
| pg_twophase | 包含准备好事务的状态文件 ||
| pg_wal(10.0及之后的版本)| 包含WAL段文件，在10.0版本中重命名未pg_xlog ||
| pg_xact(10.0及之后的版本) | 包含事务提交状态数据，在10.0版本中重命名未pg_clog ||
| pg_xlog(9.6及之前的版本)|包含WAL段文件，在10.0版本中重命名未pg_wal||


数据存放在base目录中，每个数据库对应其中一个目录(目录名为数据库的oid)，数据库中的每个表对应一组文件，文件名为表名的oid；
![picture](/interdb/fig-1-02.png "database directory")


每个page的大小默认为8kb, 在configure源代码的时候可以通过参数block_size指定；