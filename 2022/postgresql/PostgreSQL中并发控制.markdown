

PostgreSQL中通过MVCC（Multi-version Concurrency Control）来实现并发控制；
每个事务对应一个事务ID号；每个事务ID号对应一个数据快照；