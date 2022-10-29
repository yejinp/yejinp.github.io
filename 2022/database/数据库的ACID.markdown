 * **什么是事务？** **事务** 是一些程序执行组成的单元，这些单元可以访问或者更新不同的数据项；

* **ACID** 指数据库中事务的4种属性，分别是：
 
  - **A**是Atomicity的首字母，代表原子性，指一个事务，要么全部执行，要么都不执行，不能部分执行；

  - **C**是Consistency的首字母，代码一致性，指执行一个事务是隔离的（比如没有其他事务并发执行），保证一致性；

  - **I**是Isolation的首字母，尽管多个事务可能并发执行，但系统保证，对每一个事务对T<sub>i</sub>和T<sub>j</sub>, 要么T<sub>i</sub>在T<sub>j</sub>执行完毕后开始，要么T<sub>j</sub>在T<sub>i</sub>结束后开始。 因此，每个事务对系统中并发执行的其他事务是感知不到的；

  - **D**是Durability的首字母，指一个事务执行完毕后，这个事务所做的修改应该是持久化的，即使系统崩溃。

* A transaction is a unit of program execution that accesses and possibly updates various data items.

* ACID：

  - **A**(tomicity): Either all operations of the transaction are reflected properly in the database, or none are.

  - **C**(onsistency): Execution of a transaction in isolation (i.e., with no other transaction executing concurrently) preserves the consistency of the database.

  - **I**(solation): Even though multiple transactions may execute concurrently, the system guarantees that, for every pair of transactions Ti and Tj, it appears to Ti that either Tj finished execution before Ti started or Tj started execution after Ti finished. Thus, each transaction is unaware of other transactions executing concurrently in the system.

  - **D**(urability): After a transaction completes successfully, the changes it has made to the database persist, even if there are system failures.


* **参考：**

  - Abraham Silberschatz, Henry F. Korth, and S. Sudarshan, "[Database System Concepts](https://www.amazon.com/dp/0073523321)", McGraw-Hill Education, ISBN-13: 978-0073523323