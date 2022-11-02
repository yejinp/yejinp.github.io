WAL日志是数据库实现持久化（即ACID中的D）的关键；其本质是将随机写磁盘改为顺序写磁盘，从而提高数据持久化性能，避免IO性能瓶颈；

在引入WAL（7.0及其之前的版本）前，为了保证持久化，PostgreSQL将内存中修改的page通过sync系统调用同步写到磁盘。因此，修改命令，如INSERT和UPDATE的性能很低。

没有WAL的情况：
![picture](/2022/postgresql/interdb/fig-9-01.png "没有WAL的情况")

  (1) 提交第一个 INSERT，PostgreSQL从数据库集群中加载page内容到内存的共享缓存池中，插入一条元组到page中。这个page没有被立即写到数据库集群中（如第8章所述，一个已经修改过的page一般称为脏页）。
  
  (2) 提交第二个INSERT, PostgreSQL插入一个新的tuple到缓存池的页中。这个页也没有写入到存储中。
  
  (3) 如果操作系统或者PostgreSQL服务器因为任何原因（比如断电）崩溃，所有插入的数据都会丢失。

从上图中可以看出，没有WAL的数据库在系统崩溃的时候很脆弱；

有WAL的情况：
![picture](/2022/postgresql/interdb/fig-9-02.png "有WAL的情况")

(1) 后台进程定期执行checkpoint任务。当这个进程启动的时候，它会写一条XLOG记录，也叫做checkpoint记录到当前WAL段中。这个记录包含上次REDO点的位置。

(2) 

(3) 随着事务提交，PostgreSQL创建一个这个提交动作的XLOG记录，将这个记录写到WAL缓存中，然后将所有WAL缓存中的XLOG记录写到WAL段文件，LSN_1。

(4)提交第二个INSERT语句，PostgreSQL插入一条新的元组到页中，创建并将这个页的XLOG记录写到WAL缓存的LSN_2中，将TABLE_A的LSN由LSN_1更新到LSN_2。

(5)当语句提交时，PostgreSQ以与步骤3相同的方法执行。

(6)想象当操作系统发生崩溃时。即使共享缓存池中所有的数据都丢了，对页的所有修改都作为历史数据写到了WAL段中。

(1) A checkpointer, a background process, periodically performs checkpointing. Whenever the checkpointer starts, it writes a XLOG record called checkpoint record to the current WAL segment. This record contains the location of the latest REDO point.

(2) Issuing the first INSERT statement, PostgreSQL loads the TABLE_A's page into the shared buffer pool, inserts a tuple into the page, creates and writes a XLOG record of this statement into the WAL buffer at the location LSN_1, and updates the TABLE_A's LSN from LSN_0 to LSN_1.
In this example, this XLOG record is a pair of a header-data and the tuple entire.

(3) As this transaction commits, PostgreSQL creates and writes a XLOG record of this commit action into the WAL buffer, and then, writes and flushes all XLOG records on the WAL buffer to the WAL segment file, from LSN_1.

(4) Issuing the second INSERT statement, PostgreSQL inserts a new tuple into the page, creates and writes this tuple's XLOG record to the WAL buffer at LSN_2, and updates the TABLE_A's LSN from LSN_1 to LSN_2.

(5) When this statement's transaction commits, PostgreSQL operates in the same manner as in step (3).

(6) Imagine when the operating system failure should occur. Even though all of data on the shared buffer pool are lost, all modifications of the page have been written into the WAL segment files as history data.

从WAL恢复数据
![picture](/2022/postgresql/interdb/fig-9-03.png "从WAL恢复数据")

有些情况下，数据库写到持久化的文件可以存在被毁坏，这种情况下，则需要还原整个Page，因此在WAL中就需要备份整个Page。

写全page
![picture](/2022/postgresql/interdb/fig-9-04.png "写全page")


从写全page恢复数据
![picture](/2022/postgresql/interdb/fig-9-05.png "从写全page恢复数据")


 * **参考**
   - https://www.interdb.jp/pg/pgsql09.html
