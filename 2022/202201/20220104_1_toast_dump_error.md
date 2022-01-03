# [PG Upgrade Series] Toast Dump Error

# Toast Dump Error

While upgrading one production database cluster from PostgreSQL 9.1 to PostgreSQL 11, the dump error stood in the way as follows:

```
pg_dump: Dumping the contents of table "tb_test_toast" failed: PQgetResult() failed.
pg_dump: Error message from server: ERROR:  missing chunk number 0 for toast value 720200101 in pg_toast_10242048
pg_dump: The command was: COPY public.tb_test_toast (test_id, ...) TO stdout;
```

Then what measures should be taken? Take some time to fix the error?

Considering that there was window time for production upgrade, I directly dumped from one of the standby servers instead of the primary server given the fact that:

1. Before the upgrade, several tests were done using pg_dump on the standby server
2. Write operations were stopped and confirmed that there were no replication lags

The production upgrade was carried out successfully as planned. Now it is time to  find out the root cause and how to fix it in case it occurs again. 

From the error message, we can suppose that there might be data corruption for toast values.

```
ERROR:  missing chunk number 0 for toast value 720200101 in pg_toast_10242048
```

With the following SQL, we can get the toast object:

```
alvindb=# SELECT reltoastrelid::regclass FROM pg_class WHERE relname = 'tb_test_toast';
       reltoastrelid        
----------------------------
 pg_toast.pg_toast_10242048
(1 row)
```

# Steps Planned

After doing some investigation, following steps were listed to try one by one:

First, try to locate corrupted rows that block the dump by looping the table with primary key.

```
SELECT * FROM tb_test_toast WHERE test_id >= $start AND test_id< $end;
```

Then set responsible columns to null or delete rows related.

After that run following SQLs:

```
VACUUM VERBOSE tb_test_toast;
ANALYZE VERBOSE tb_test_toast;
REINDEX TABLE pg_toast.pg_toast_10242048;
REINDEX TABLE tb_test_toast;
```

#  Steps Carried Out

After searching through the table's data, no bad data was found.

Then VACUUM the table,

```
VACUUM VERBOSE tb_test_toast;
```

From the  VACUUM outputs we can see that obsolete row versions were removed.

```
...
psql:vacuum_tb_test_toast.sql:1: INFO:  vacuuming "pg_toast.pg_toast_10242048"
psql:vacuum_tb_test_toast.sql:1: INFO:  scanned index "pg_toast_10242048_index" to remove 413151 row versions
DETAIL:  CPU 0.19s/1.45u sec elapsed 1.71 sec.
psql:vacuum_tb_test_toast.sql:1: INFO:  "pg_toast_10242048": removed 413151 row versions in 129411 pages
DETAIL:  CPU 0.91s/0.40u sec elapsed 1.31 sec.
psql:vacuum_tb_test_toast.sql:1: INFO:  index "pg_toast_10242048_index" now contains 6576652 row versions in 140655 pages
DETAIL:  413151 index row versions were removed.
9153 index pages have been deleted, 3576 are currently reusable.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
psql:vacuum_tb_test_toast.sql:1: INFO:  "pg_toast_10242048": found 364889 removable, 5763282 nonremovable row versions in 2251437 out of 2508399 pages
DETAIL:  0 dead row versions cannot be removed yet.
There were 5729845 unused item pointers.
0 pages are entirely empty.
CPU 15.41s/8.92u sec elapsed 31.02 sec.
VACUUM
```

After the VACUUM, the dump was successful!
