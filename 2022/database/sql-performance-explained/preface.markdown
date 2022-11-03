From [https://use-the-index-luke.com/](https://use-the-index-luke.com/)

SQL performance problems are as old as SQL itself—some might even say that SQL is inherently slow. Although this might have been true in the early days of SQL, it is definitely not true anymore. Nevertheless SQL performance problems are still commonplace. How does this happen?

The SQL language is perhaps the most successful fourth-generation programming language (4GL). Its main benefit is the capability to separate “what” and “how”. An SQL statement is a straight description of what is needed without instructions as to how to get it done. Consider the following example:

```
SELECT date_of_birth
  FROM employees
 WHERE last_name = 'WINAND'
```

The SQL query reads like an English sentence that explains the requested data. Writing SQL statements generally does not require any knowledge about inner workings of the database or the storage system (such as disks, files, etc.). There is no need to tell the database which files to open or how to find the requested rows. Many developers have years of SQL experience yet they know very little about the processing that happens in the database.

The separation of concerns—what is needed versus how to get it—works remarkably well in SQL, but it is still not perfect. The abstraction reaches its limits when it comes to performance: the author of an SQL statement by definition does not care how the database executes the statement. Consequently, the author is not responsible for slow execution. However, experience proves the opposite; i.e., the author must know a little bit about the database to prevent performance problems.

It turns out that the only thing developers need to learn is how to index. Database indexing is, in fact, a development task. That is because the most important information for proper indexing is not the storage system configuration or the hardware setup. The most important information for indexing is how the application queries the data. This knowledge—about the access path—is not very accessible to database administrators (DBAs) or external consultants. Quite some time is needed to gather this information through reverse engineering of the application: development, on the other hand, has that information anyway.

This book covers everything developers need to know about indexes—and nothing more. To be more precise, the book covers the most important index type only: the B-tree index.

On my Own Behalf
I make my living from SQL training, SQL tuning and consulting and my book “SQL Performance Explained”. Learn more at https://winand.at/.

The B-tree index works almost identically in many databases. The book primarily uses the terminology of the Oracle® database, but refers to the corresponding terms of other database where appropriate. Side notes provide more information about MySQL, PostgreSQL and SQL Server®.

The structure of the book is tailor-made for developers; most chapters correspond to a particular part of an SQL statement.

CHAPTER 1 - Anatomy of an Index
The first chapter is the only one that doesn’t cover SQL specifically; it is about the fundamental structure of an index. An understanding of the index structure is essential to following the later chapters—don’t skip this!

Although the chapter is rather short—only about eight pages—after working through the chapter you will already understand the phenomenon of slow indexes.

CHAPTER 2 - The Where Clause
This is where we pull out all the stops. This chapter explains all aspects of the where clause, from very simple single column lookups to complex clauses for ranges and special cases such as LIKE.

This chapter makes up the main body of the book. Once you learn to use these techniques, you will write much faster SQL.

CHAPTER 3 - Performance and Scalability
This chapter is a little digression about performance measurements and database scalability. See why adding hardware is not the best solution to slow queries.

CHAPTER 4 - The Join Operation
Back to SQL: here you will find an explanation of how to use indexes to perform a fast table join.

CHAPTER 5 - Clustering Data
Have you ever wondered if there is any difference between selecting a single column or all columns? Here is the answer—along with a trick to get even better performance.

CHAPTER 6 - Sorting and Grouping
Even order by and group by can use indexes.

CHAPTER 7 - Partial Results
This chapter explains how to benefit from a “pipelined” execution if you don’t need the full result set.

CHAPTER 8 - Insert, Delete and Update
How do indexes affect write performance? Indexes don’t come for free—use them wisely!

APPENDIX A - Execution Plans
Asking the database how it executes a statement.

APPENDIX B - Myth Directory
Lists some common myth and explains the truth. Will be extended as the book grows.

APPENDIX C - Example Schema
All create and insert statements for the tables from the book.