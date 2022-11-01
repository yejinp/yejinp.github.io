

PostgreSQL中通过MVCC（Multi-version Concurrency Control）来实现并发控制；
每个事务对应一个事务ID号；每个事务ID号对应一个数据快照；

由事务管理器提供事务快照。在READ COMMITED隔离级别中，当执行SQL完毕后就会获取快照；否则（REPEATABLE READ或者SERIALIZABLE隔离级别），只在第一条SQL命令执行的时候获取快照；

当用获取的快照检查可见性时，必须把快照中活动的事务当作处理中，即使他们实际上已经提交或者取消了。 这个规则很重要，因为它会引起在处理READ COMMITED 和 REAPEATABLE READ(或者 SERIALIZABLE)的不同行为；

PostgreSQL中就是提供上述方法实现各种隔离级别；

**从上面的描述看出了REAPEATABLE READ和READ COMMITED之间的区别，没看出怎么区分或者说实现REAPEATABLE READ和SERIALIZABLE 两个不同隔离级别的？？**

答：通过 Serializable Snapshot Isolation 实现真正的可序列化。


Transaction snapshots are provided by the transaction manager. In the READ COMMITTED isolation level, the transaction obtains a snapshot whenever an SQL command is executed; otherwise (REPEATABLE READ or SERIALIZABLE), the transaction only gets a snapshot when the first SQL command is executed.

When using the obtained snapshot for the visibility check, active transactions in the snapshot must be treated as in progress even if they have actually been committed or aborted. This rule is important because it causes the difference in the behaviour between READ COMMITTED and REPEATABLE READ (or SERIALIZABLE). 