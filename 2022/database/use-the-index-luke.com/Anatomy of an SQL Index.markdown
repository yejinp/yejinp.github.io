“An index makes the query fast” is the most basic explanation of an index I have ever seen. Although it describes the most important aspect of an index very well, it is—unfortunately—not sufficient for this book. This chapter describes the index structure in a less superficial way but doesn’t dive too deeply into details. It provides just enough insight for one to understand the SQL performance aspects discussed throughout the book.

An index is a distinct structure in the database that is built using the create index statement. It requires its own disk space and holds a copy of the indexed table data. That means that an index is pure redundancy. Creating an index does not change the table data; it just creates a new data structure that refers to the table. A database index is, after all, very much like the index at the end of a book: it occupies its own space, it is highly redundant, and it refers to the actual information stored in a different place.

Clustered Indexes (SQL Server, MySQL/InnoDB)
```
SQL Server and MySQL (using InnoDB) take a broader view of what “index” means. They refer to tables that consist of the index structure only as clustered indexes. These tables are called Index-Organized Tables (IOT) in the Oracle database.

Chapter 5, “Clustering Data”, describes them in more detail and explains their advantages and disadvantages.
```

Searching in a database index is like searching in a printed telephone directory. The key concept is that all entries are arranged in a well-defined order. Finding data in an ordered data set is fast and easy because the sort order determines each entry’s position.


A database index is, however, more complex than a printed directory because it undergoes constant change. Updating a printed directory for every change is impossible for the simple reason that there is no space between existing entries to add new ones. A printed directory bypasses this problem by only handling the accumulated updates with the next printing. An SQL database cannot wait that long. It must process insert, delete and update statements immediately, keeping the index order without moving large amounts of data.

The database combines two data structures to meet the challenge: a doubly linked list and a search tree. These two structures explain most of the database’s performance characteristics.

Contents
The Leaf Nodes — A doubly linked list
The B-Tree — It’s a balanced tree
Slow Indexes, Part I — Two ingredients make the index slow