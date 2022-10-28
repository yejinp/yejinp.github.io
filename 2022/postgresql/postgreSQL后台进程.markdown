process	description	reference

|               |         |
|               |         |
|---------------|---------|
|               |         |
|               |         |
|               |         |



background writer	In this process, dirty pages on the shared buffer pool are written to a persistent storage (e.g., HDD, SSD) on a regular basis gradually. (In version 9.1 or earlier, it was also responsible for checkpoint process.)
checkpointer	In this process in version 9.2 or later, checkpoint process is performed.
autovacuum launcher	The autovacuum-worker processes are invoked for vacuum process periodically. (More precisely, it requests to create the autovacuum workers to the postgres server.)	
WAL writer	This process writes and flushes periodically the WAL data on the WAL buffer to persistent storage.
statistics collector	In this process, statistics information such as for pg_stat_activity and for pg_stat_database, etc. is collected.	 
logging collector (logger)	This process writes error messages into log files.	 
archiver	In this process, archiving logging is executed.