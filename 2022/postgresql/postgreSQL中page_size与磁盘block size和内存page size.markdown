postgresql中读取文件使用page保存表的数据；

磁盘的最小存取单位为block size

虚拟内存的page size

- postgresql中查看page size方式： show block_size;
```
test=# SHOW block_size;
 block_size
------------
 8192
(1 row)
```

- Linux中查看 page size : getconf PAGESIZE 
或者 getconf PAGE_SIZE

```
[root@test ~]# getconf PAGESIZE
4096
```

- 查看磁盘 block size:  blockdev --getbsz /dev/sda2
```
[root@test ~]# blockdev --getbsz /dev/sda2
4096
```

上面的几个参数含义容易搞混，为此，特别在此梳理一下：
 - postgreSQL中的pagesize是postgreSQL中存取表文件的最小单位；

 - Linux操作系统的page size是内存页的大小

 - 磁盘的 block size也叫磁盘块，是磁盘存储文件的基本单位，一个磁盘块中不能存储两个文件；

block size大于虚拟内存的 page size时，读取一个block size到内存中会引起跨page；

block size越大，越容易实现HOT功能， Index-Only功能可能性越低；
block size越小，越难实现HOT功能， Index-Only功能可能性越高；