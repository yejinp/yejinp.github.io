Vacuum Processing
空间回收处理

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

<h2>Full VACUUM<h2>

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

