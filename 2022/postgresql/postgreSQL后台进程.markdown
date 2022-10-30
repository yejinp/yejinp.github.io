process	description	reference

| 进程名 | 描述 | 备注 |
|--|--|--|
| background writer |	这个进程中，定期的基于一些基本的规则，将共享缓存中的脏页写到持久化存储（比如HDD或者SSD）中。<br>在版本9.1及之前，这个进程也负责checkpoint处理 ||
| checkpointer | 在9.2已经之后的版本中，处理checkpoint| |
| autovacuum launcher | 这个处理进程在定期处理vacuum的时候唤醒（更精确的说，它给服务器创建autovacuum工作进程） | |
| WAL writer | 这个进程周期性的将WAL缓存中的WAL数据写到持久化存储中 ||
| statistics collector | 这个进程中，收集统计信息，这些信息用于pg_stat_activity和pg_stat_database||
| logging collector (logger) | 这个进程将错误消息写到日志文件中||
| archiver | 这个进程中执行日志归档工作||

进程图示例
![picture](/2022/postgresql/interdb/fig-2-01.png "processor")

 * **参考**
   - https://www.interdb.jp/pg/pgsql02.html