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
| background writer |In this process, dirty pages on the shared buffer pool are written to a persistent storage (e.g., HDD, SSD) on a regular basis gradually. (In version 9.1 or earlier, it was also responsible for checkpoint process.)||
| checkpointer |	In this process in version 9.2 or later, checkpoint process is performed. ||
| autovacuum launcher|	The autovacuum-worker processes are invoked for vacuum process periodically. (More precisely, it requests to create the autovacuum workers to the postgres server.)	||
| WAL writer |	This process writes and flushes periodically the WAL data on the WAL buffer to persistent storage. ||
| statistics collector |	In this process, statistics information such as for pg_stat_activity and for pg_stat_database, etc. is collected. ||
| logging collector (logger) |	This process writes error messages into log files. ||	 
| archiver |	In this process, archiving logging is executed. | |