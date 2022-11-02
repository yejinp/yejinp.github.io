WAL日志是数据库实现持久化（即ACID中的D）的关键；其本质是将随机写磁盘改为顺序写磁盘，从而提高数据持久化性能，避免IO性能瓶颈；

在引入WAL（7.0及其之前的版本）前，为了保证持久化，PostgreSQL将内存中修改的page通过sync系统调用同步写到磁盘。因此，修改命令，如INSERT和UPDATE的性能很低。

没有WAL的情况：
![picture](/2022/postgresql/interdb/fig-9-01.png "没有WAL的情况")


从上图中可以看出，没有WAL的数据库在系统崩溃的时候很脆弱；

有WAL的情况：
![picture](/2022/postgresql/interdb/fig-9-02.png "有WAL的情况")


从WAL恢复数据
![picture](/2022/postgresql/interdb/fig-9-03.png "从WAL恢复数据")

有些情况下，数据库写到持久化的文件可以存在被毁坏，这种情况下，则需要还原整个Page，因此在WAL中就需要备份整个Page。

写全page
![picture](/2022/postgresql/interdb/fig-9-04.png "写全page")


从写全page恢复数据
![picture](/2022/postgresql/interdb/fig-9-05.png "从写全page恢复数据")


 * **参考**
   - https://www.interdb.jp/pg/pgsql09.html
