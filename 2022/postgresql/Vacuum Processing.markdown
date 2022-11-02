<h2>Vacuum Processing</h2>
空间回收处理

随着不断的插入，删除或者更新表，表中的page可能存在很多碎片；这些碎片会造成表膨胀，影响访问效率，因此有必要进行整理；


并发 VACUUM的伪代码：

```
(1)  FOR each table
(2)       Acquire ShareUpdateExclusiveLock lock for the target table

          /* The first block */
(3)       Scan all pages to get all dead tuples, and freeze old tuples if necessary 
(4)       Remove the index tuples that point to the respective dead tuples if exists

          /* The second block */
(5)       FOR each page of the table
(6)            Remove the dead tuples, and Reallocate the live tuples in the page
(7)            Update FSM and VM
           END FOR

          /* The third block */
(8)       Clean up indexes
(9)       Truncate the last page if possible
(10       Update both the statistics and system catalogs of the target table
           Release ShareUpdateExclusiveLock lock
       END FOR

        /* Post-processing */
(11)  Update statistics and system catalogs
(12)  Remove both unnecessary files and pages of the clog if possible
```

从上面可以看出来，这个过程使用的是共享更新排它锁，也即；

<h2>Visibility Map</h2>
回收空间过程代价很高，因此，在8.4版本中引入VM，以减少代价；

VM的基本概念很简单。每张表都有一个单独的Visibility map，存放表中每个page的可见性。page的可见性决定了是否需要处理每个page。回收处理可以跳过一些没有死元组的page。

下图可以看到如何使用VM。 假设表含有3个page，另外，0号page和2号page含有死元组，1号page则没有。 这个表的VM保留哪些page含有死元组的信息。本例中，回收处理根据VM信息跳过1号page。

![picture](/2022/postgresql/interdb/fig-6-02.png "processor")

每个VM由一个或者多个8Kb的page组成，这个文件以vm后缀保存。举个例子，一个表文件的relfilenode为18751，它的FSM（18751_fsm）和VM(18751_vm)文件如下所示：

```
$ cd $PGDATA
$ ls -la base/16384/18751*
-rw------- 1 postgres postgres  8192 Apr 21 10:21 base/16384/18751
-rw------- 1 postgres postgres 24576 Apr 21 10:18 base/16384/18751_fsm
-rw------- 1 postgres postgres  8192 Apr 21 10:18 base/16384/18751_vm
```

Vacuum processing is costly; thus, the VM was been introduced in version 8.4 to reduce this cost.

The basic concept of the VM is simple. Each table has an individual visibility map that holds the visibility of each page in the table file. The visibility of pages determines whether each page has dead tuples. Vacuum processing can skip a page that does not have dead tuples.

Figure 6.2 shows how the VM is used. Suppose that the table consists of three pages, and the 0th and 2nd pages contain dead tuples and the 1st page does not. The VM of this table holds information about which pages contain dead tuples. In this case, vacuum processing skips the 1st page by referring to the VM's information.

Each VM is composed of one or more 8 KB pages, and this file is stored with the 'vm' suffix. As an example, one table file whose relfilenode is 18751 with FSM (18751_fsm) and VM (18751_vm) files shown in the following.

<h2>Full VACUUM</h2>

* Full VACUUM模式的概要过程：

![picture](/2022/postgresql/interdb/fig-6-09.png "processor")



```
(1)  FOR each table
(2)       Acquire AccessExclusiveLock lock for the table
(3)       Create a new table file
(4)       FOR each live tuple in the old table
(5)            Copy the live tuple to the new table file
(6)            Freeze the tuple IF necessary
            END FOR
(7)        Remove the old table file
(8)        Rebuild all indexes
(9)        Update FSM and VM
(10)      Update statistics
            Release AccessExclusiveLock lock
       END FOR
(11)  Remove unnecessary clog files and pages if possible
```



从上面可以看出来，这个过程使用的是访问排它锁；

 * 对于VACUUM FULL命令，有两点需要注意：
   - 表在命令执行期间，没人可以访问（读或者写）这个表；

   - 最大可能会使用原表两倍的临时磁盘空间；因此，当处理一个大表时，有必要先检查磁盘容量是否够用；


从这个图片可以看出来，处理过程可以回收磁盘空间，因此可以减少磁盘IO，提高效率；

由于使用了排它锁，因此，在中国过程中其他人不能访问这个表，因此，需要减少这类操作。

那么，什么时候执行FULL VACUUM就很关键；这恐怕没有简单的答案，一般的建议是，使用pg_freespacemap来查看表的FSM，当表的使用率很低时才执行 VACUUM FULL操作；


 * **参考**
   - https://www.interdb.jp/pg/pgsql06.html
