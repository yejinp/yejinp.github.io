WAL日志是数据库实现持久化（即ACID中的D）的关键；其本质是将随机写磁盘改为顺序写磁盘，从而提高数据持久化性能，避免IO性能瓶颈；

在引入WAL（7.0及其之前的版本）前，为了保证持久化，PostgreSQL将内存中修改的page通过sync系统调用同步写到磁盘。因此，修改命令，如INSERT和UPDATE的性能很低。

Before WAL was introduced (version 7.0 or earlier), PostgreSQL did synchronous writes to the disk by issuing a sync system call whenever changing a page in memory in order to ensure durability. Therefore, the modification commands such as INSERT and UPDATE were very poor-performance.

