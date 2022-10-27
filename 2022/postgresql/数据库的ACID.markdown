
* ACID：

  - A(tomicity): Either all operations of the transaction are reflected properly in the database, or none are.

  - C(onsistency): Execution of a transaction in isolation (i.e., with no other transaction executing concurrently) preserves the consistency of the database.

  - I(solation): Even though multiple transactions may execute concurrently, the system guarantees that, for every pair of transactions Ti and Tj, it appears to Ti that either Tj finished execution before Ti started or Tj started execution after Ti finished. Thus, each transaction is unaware of other transactions executing concurrently in
the system.

  - D(urability): After a transaction completes successfully, the changes it has made to the database persist, even if there are system failures.