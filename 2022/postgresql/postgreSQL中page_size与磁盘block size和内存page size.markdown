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

- Linux中查看 page size : getcon PAGESIZE 
```
[root@test ~]# getconf PAGESIZE
4096
```

- 查看磁盘 block size:  blockdev --getbsz /dev/sda2
```
[root@test ~]# blockdev --getbsz /dev/sda2
4096
```