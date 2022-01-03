# Issue

```
pg_dump: Dumping the contents of table "crawler_hotel_info" failed: PQgetResult() failed.
pg_dump: Error message from server: ERROR:  missing chunk number 0 for toast value 735979090 in pg_toast_17239071
pg_dump: The command was: COPY public.crawler_hotel_info (id, hotel_seq, hotel_id, wrapper_id, created, updated, hotel_status, city_code, tree_id, vrsn, hotel_info, oper_status, is_hecheng, crawl_date, info_hash_code) TO stdout;
```



```
pg_dump: last built-in OID is 16383
pg_dump: reading extensions
pg_dump: identifying extension members
pg_dump: reading schemas
pg_dump: reading user-defined tables
pg_dump: reading user-defined functions
pg_dump: reading user-defined types
pg_dump: reading procedural languages
pg_dump: reading user-defined aggregate functions
pg_dump: reading user-defined operators
pg_dump: reading user-defined access methods
pg_dump: reading user-defined operator classes
pg_dump: reading user-defined operator families
pg_dump: reading user-defined text search parsers
pg_dump: reading user-defined text search templates
pg_dump: reading user-defined text search dictionaries
pg_dump: reading user-defined text search configurations
pg_dump: reading user-defined foreign-data wrappers
pg_dump: reading user-defined foreign servers
pg_dump: reading default privileges
pg_dump: reading user-defined collations
pg_dump: reading user-defined conversions
pg_dump: reading type casts
pg_dump: reading transforms
pg_dump: reading table inheritance information
pg_dump: reading event triggers
pg_dump: finding extension tables
pg_dump: finding inheritance relationships
pg_dump: reading column info for interesting tables
pg_dump: finding the columns and types of table "public.crawler_wrapper_rtdesc"
pg_dump: finding default expressions of table "public.crawler_wrapper_rtdesc"
pg_dump: flagging inherited columns in subtables
pg_dump: reading indexes
pg_dump: reading indexes for table "public.crawler_wrapper_rtdesc"
pg_dump: flagging indexes in partitioned tables
pg_dump: reading extended statistics
pg_dump: reading constraints
pg_dump: reading triggers
pg_dump: reading rewrite rules
pg_dump: reading policies
pg_dump: reading publications
pg_dump: reading publication membership
pg_dump: reading subscriptions
pg_dump: reading dependency data
pg_dump: saving encoding = UTF8
pg_dump: saving standard_conforming_strings = on
pg_dump: saving search_path = 
pg_dump: creating TABLE "public.crawler_wrapper_rtdesc"
pg_dump: creating COMMENT "public.COLUMN crawler_wrapper_rtdesc.hotel_status"
pg_dump: creating COMMENT "public.COLUMN crawler_wrapper_rtdesc.vrsn"
pg_dump: creating COMMENT "public.COLUMN crawler_wrapper_rtdesc.oper_status"
pg_dump: creating COMMENT "public.COLUMN crawler_wrapper_rtdesc.is_hecheng"
pg_dump: creating SEQUENCE "public.hecheng_wrapper_rtdesc_id_seq"
pg_dump: creating SEQUENCE OWNED BY "public.hecheng_wrapper_rtdesc_id_seq"
pg_dump: creating DEFAULT "public.crawler_wrapper_rtdesc id"
pg_dump: processing data for table "public.crawler_wrapper_rtdesc"
pg_dump: dumping contents of table "public.crawler_wrapper_rtdesc"
pg_dump: Dumping the contents of table "crawler_wrapper_rtdesc" failed: PQgetResult() failed.
pg_dump: Error message from server: ERROR:  missing chunk number 0 for toast value 732899944 in pg_toast_16940642
pg_dump: The command was: COPY public.crawler_wrapper_rtdesc (id, hotel_seq, hotel_id, wrapper_id, created, updated, hotel_status, city_code, tree_id, vrsn, rtdesc, oper_status, is_hecheng, crawl_date, rt_desc_hash_code) TO stdout;

```



## crawler_wrapper_rtdesc

```
          spcname |             relation             | total_table_size | table_only_size | total_index_size
---------+----------------------------------+------------------+-----------------+------------------
         | public.crawler_wrapper_rtdesc    | 23 GB            | 18 GB           | 4714 MB
```



```
data_close_loop=# \dt+ public.crawler_wrapper_rtdesc
                             List of relations
 Schema |          Name          | Type  |   Owner   | Size  | Description 
--------+------------------------+-------+-----------+-------+-------------
 public | crawler_wrapper_rtdesc | table | closeloop | 23 GB | 
(1 row)

data_close_loop=# \d+ crawler_wrapper_rtdesc
                                                                                                                  Table "public.crawler_wrapper_rtdesc"
      Column       |            Type             |                              Modifiers                              | Storage  |                                                             Description                                                              
-------------------+-----------------------------+---------------------------------------------------------------------+----------+--------------------------------------------------------------------------------------------------------------------------------------
 id                | integer                     | not null default nextval('hecheng_wrapper_rtdesc_id_seq'::regclass) | plain    | 
 hotel_seq         | character varying(50)       | not null default ''::character varying                              | extended | 
 hotel_id          | character varying(50)       | not null default ''::character varying                              | extended | 
 wrapper_id        | character varying(50)       | not null default ''::character varying                              | extended | 
 created           | timestamp without time zone | not null default now()                                              | plain    | 
 updated           | timestamp without time zone | not null default now()                                              | plain    | 
 hotel_status      | smallint                    | not null default 0::smallint                                        | plain    | 状态: 0 有效  1 无效
 city_code         | character varying(100)      | not null default ''::character varying                              | extended | 
 tree_id           | bigint                      | not null default 0::bigint                                          | plain    | 
 vrsn              | bigint                      | not null default 0::bigint                                          | plain    | 版本号，此版本号是与hecheng_wrapper_tree的版本号联动，勿用作其他用途，两个表的版本号不一致代表数据（hecheng_wrapper_rtdesc）需要更新
 rtdesc            | text                        | not null default ''::text                                           | extended | 
 oper_status       | smallint                    | not null default 0::smallint                                        | plain    | 操作状态，0未推送， 1，已推送到RT
 is_hecheng        | smallint                    | not null default 0::smallint                                        | plain    | 0是赫程， 1非赫程
 crawl_date        | timestamp without time zone |                                                                     | plain    | 
 rt_desc_hash_code | character varying(65)       |                                                                     | extended | 
Indexes:
    "hecheng_wrapper_rtdesc_pkey" PRIMARY KEY, btree (id)
    "crawler_wrapper_rtdesc_updated_idx" btree (updated)
    "crawler_wrapper_rtdesc_wrapper_id_crawl_date_idx" btree (wrapper_id, crawl_date)
    "idx_crawler_wrapper_rtdesc_crawldate" btree (crawl_date)
    "idx_crawler_wrapper_rtdesc_wrapperid_operstatus_updated" btree (wrapper_id, oper_status, updated)
    "idx_hechenng_rtdesc_hotelseq_status" btree (hotel_seq, hotel_status)
    "idx_hechenng_rtdesc_treeid" btree (tree_id)
Has OIDs: no

data_close_loop=# SELECT min(id),max(id) from crawler_wrapper_rtdesc;
  min   |   max   
--------+---------
 551110 | 4829377
(1 row)

data_close_loop=# SELECT * from pg_stat_user_tables where relname = 'crawler_wrapper_rtdesc';
-[ RECORD 1 ]-----+-----------------------
relid             | 16940642
schemaname        | public
relname           | crawler_wrapper_rtdesc
seq_scan          | 12
seq_tup_read      | 27579608
idx_scan          | 4025
idx_tup_fetch     | 3567384
n_tup_ins         | 0
n_tup_upd         | 748
n_tup_del         | 0
n_tup_hot_upd     | 0
n_live_tup        | 0
n_dead_tup        | 748
last_vacuum       | 
last_autovacuum   | 
last_analyze      | 
last_autoanalyze  | 
vacuum_count      | 0
autovacuum_count  | 0
analyze_count     | 0
autoanalyze_count | 0



```



```
/opt/pg11/bin/pg_dump -v -U postgres -h localhost -p 9432 -d data_close_loop -t crawler_wrapper_rtdesc &> crawler_wrapper_rtdesc2.data && echo Done|mail -s "`hostname`: crawler_wrapper_rtdesc dump succeeded" "mobile.zhenzhong.li@alert.qunar.com" || echo Done|mail -s "`hostname`: crawler_wrapper_rtdesc dump failed" "mobile.zhenzhong.li@alert.qunar.com"
```



```
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# cat crawler_wrapper_rtdesc.sh
start=551110
end=4829377
step=1000
while [ $start -le $end ]
do
  #echo "$start ..."
  tmp=$(( $start + $step ))
  echo "$start ... $tmp"
  time /opt/pg11/bin/psql -U postgres -p 9432 -d data_close_loop -c "select * from crawler_wrapper_rtdesc where id >= $start and id < $tmp" > /dev/null
  FLAG=$?
  if [ $FLAG -ge 1 ];then
    echo "Corrupted chunk read"
    echo Done|mail -s "`hostname`: Corrupted chunk read $start ... $tmp " "mobile.zhenzhong.li@alert.qunar.com"
    break;
  fi
  start=$tmp
done
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# 
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# bash -x crawler_wrapper_rtdesc.sh &> crawler_wrapper_rtdesc.sh.log

```





```
/opt/pg11/bin/psql -U postgres -p 9432 -d data_close_loop -f vacuum_crawler_wrapper_rtdesc.sql &> vacuum_crawler_wrapper_rtdesc.log  && echo Done|mail -s "`hostname`: vacuum_crawler_wrapper_rtdesc completed" "mobile.zhenzhong.li@alert.qunar.com"

/opt/pg11/bin/pg_dump -v -U postgres -h localhost -p 9432 -d data_close_loop -t crawler_wrapper_rtdesc &> crawler_wrapper_rtdesc2.data && echo Done|mail -s "`hostname`: crawler_wrapper_rtdesc dump 2 succeeded" "mobile.zhenzhong.li@alert.qunar.com" || echo Done|mail -s "`hostname`: crawler_wrapper_rtdesc dump 2 failed" "mobile.zhenzhong.li@alert.qunar.com"

SELECT * from pg_stat_user_tables where relname = 'crawler_wrapper_rtdesc';

/opt/pg11/bin/psql -U postgres -p 9432 -d data_close_loop -f analyze_crawler_wrapper_rtdesc.sql &> analyze_crawler_wrapper_rtdesc.log  && echo Done|mail -s "`hostname`: analyze_crawler_wrapper_rtdesc completed" "mobile.zhenzhong.li@alert.qunar.com"

/opt/pg11/bin/pg_dump -v -U postgres -h localhost -p 9432 -d data_close_loop -t crawler_wrapper_rtdesc &> crawler_wrapper_rtdesc3.data && echo Done|mail -s "`hostname`: crawler_wrapper_rtdesc dump 3 succeeded" "mobile.zhenzhong.li@alert.qunar.com" || echo Done|mail -s "`hostname`: crawler_wrapper_rtdesc dump 3 failed" "mobile.zhenzhong.li@alert.qunar.com"

SELECT * from pg_stat_user_tables where relname = 'crawler_wrapper_rtdesc';

/opt/pg11/bin/psql -U postgres -p 9432 -d data_close_loop -f reindex_toast_crawler_wrapper_rtdesc.sql &> reindex_toast_crawler_wrapper_rtdesc.log  && echo Done|mail -s "`hostname`: reindex_toast_crawler_wrapper_rtdesc completed" "mobile.zhenzhong.li@alert.qunar.com"

/opt/pg11/bin/pg_dump -v -U postgres -h localhost -p 9432 -d data_close_loop -t crawler_wrapper_rtdesc &> crawler_wrapper_rtdesc4.data && echo Done|mail -s "`hostname`: crawler_wrapper_rtdesc dump 4 succeeded" "mobile.zhenzhong.li@alert.qunar.com" || echo Done|mail -s "`hostname`: crawler_wrapper_rtdesc dump 4 failed" "mobile.zhenzhong.li@alert.qunar.com"


/opt/pg11/bin/psql -U postgres -p 9432 -d data_close_loop -f reindex_crawler_wrapper_rtdesc.sql &> reindex_crawler_wrapper_rtdesc.log  && echo Done|mail -s "`hostname`: reindex_crawler_wrapper_rtdesc completed" "mobile.zhenzhong.li@alert.qunar.com"

/opt/pg11/bin/pg_dump -v -U postgres -h localhost -p 9432 -d data_close_loop -t crawler_wrapper_rtdesc &> crawler_wrapper_rtdesc5.data && echo Done|mail -s "`hostname`: crawler_wrapper_rtdesc dump 5 succeeded" "mobile.zhenzhong.li@alert.qunar.com" || echo Done|mail -s "`hostname`: crawler_wrapper_rtdesc dump 5 failed" "mobile.zhenzhong.li@alert.qunar.com"
```







# Solution 1

Dump from slave

https://newbiedba.wordpress.com/2015/07/07/postgresql-missing-chunk-0-for-toast-value-in-pg_toast/

# Solution 2

https://gist.github.com/supix/80f9a6111dc954cf38ee99b9dedf187a



```
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# cat vacuum_crawler_wrapper_rtdesc.sql 
VACUUM VERBOSE crawler_wrapper_rtdesc
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# cat analyze_crawler_wrapper_rtdesc.sql
ANALYZE VERBOSE crawler_wrapper_rtdesc
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# cat reindex_toast_crawler_wrapper_rtdesc.sql 
REINDEX TABLE pg_toast.pg_toast_16940642;
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# cat reindex_crawler_wrapper_rtdesc.sql 
REINDEX TABLE crawler_wrapper_rtdesc
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# 

```



```
select pg_toast_16940642::regclass;
select reltoastrelid::regclass from pg_class where relname = 'crawler_wrapper_rtdesc';
```



```
data_close_loop=# select reltoastrelid from pg_class where relname = 'crawler_wrapper_rtdesc';
 reltoastrelid 
---------------
      16940659
(1 row)

data_close_loop=# select reltoastrelid::regclass from pg_class where relname = 'crawler_wrapper_rtdesc';
       reltoastrelid        
----------------------------
 pg_toast.pg_toast_16940642
(1 row)

```



```
data_close_loop=# select reltoastrelid::regclass from pg_class where relname = 'crawler_hotel_info';
       reltoastrelid        
----------------------------
 pg_toast.pg_toast_17239071
(1 row)

data_close_loop=# select reltoastrelid from pg_class where relname = 'crawler_hotel_info';
 reltoastrelid 
---------------
      17239087
(1 row)

data_close_loop=# 

```



```
[root@l-closeloopdb1.h.cn2 ~]# /opt/pg11/bin/psql -U postgres
psql (11.14)
Type "help" for help.

postgres=# \c data_close_loop 
You are now connected to database "data_close_loop" as user "postgres".
data_close_loop=# select reltoastrelid::regclass from pg_class where relname = 'crawler_wrapper_rtdesc';
      reltoastrelid       
--------------------------
 pg_toast.pg_toast_976233
(1 row)

data_close_loop=# select reltoastrelid::regclass from pg_class where relname = 'crawler_hotel_info';
      reltoastrelid       
--------------------------
 pg_toast.pg_toast_975759
(1 row)

data_close_loop=# 
```



```
[postgres@l-closeloopdb3.h.cn2 ~]$ /opt/pg11/bin/psql -U postgres
psql (11.14)
Type "help" for help.

postgres=# \c data_close_loop 
You are now connected to database "data_close_loop" as user "postgres".
data_close_loop=# select reltoastrelid::regclass from pg_class where relname = 'crawler_wrapper_rtdesc';
      reltoastrelid       
--------------------------
 pg_toast.pg_toast_976233
(1 row)

data_close_loop=# select reltoastrelid::regclass from pg_class where relname = 'crawler_hotel_info';
      reltoastrelid       
--------------------------
 pg_toast.pg_toast_975759
(1 row)

data_close_loop=# 
```





```
[root@l-closeloopdb1.h.cn2 ~]# /opt/pg11/bin/psql -U postgres -p 9432 -d data_close_loop -c 'select * from crawler_hotel_info where 1< limit 1' &> /dev/null || echo "Corrupted chunk read"

```





```
start=824423
#start=554423
#end=593386
end=4693387
step=1000
while [ $start -le $end ]
do
  #echo "$start ..."
  tmp=$(( $start + $step ))
  echo "$start ... $tmp"
  time /opt/pg11/bin/psql -U postgres -p 9432 -d data_close_loop -c "select * from crawler_hotel_info where id >= $start and id < $tmp " &> /dev/null
  FLAG=$?
  if [ $FLAG -ge 1 ];then
    echo "Corrupted chunk read"
    echo Done|mail -s "`hostname`: screen command completed" "mobile.zhenzhong.li@alert.qunar.com"
    break;
  fi
  start=$tmp
done

```



```
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# cat crawler_wrapper_rtdesc.sh
start=551110
end=4829377
step=1000
while [ $start -le $end ]
do
  #echo "$start ..."
  tmp=$(( $start + $step ))
  echo "$start ... $tmp"
  time /opt/pg11/bin/psql -U postgres -p 9432 -d data_close_loop -c "select * from crawler_wrapper_rtdesc where id >= $start and id < $tmp" > /dev/null
  FLAG=$?
  if [ $FLAG -ge 1 ];then
    echo "Corrupted chunk read"
    echo Done|mail -s "`hostname`: Corrupted chunk read $start ... $tmp " "mobile.zhenzhong.li@alert.qunar.com"
    break;
  fi
  start=$tmp
done
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# 
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# bash -x crawler_wrapper_rtdesc.sh &> crawler_wrapper_rtdesc.sh.log

```







# vacuuming

```
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# tailf vacuum_crawler_wrapper_rtdesc.log
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  vacuuming "public.crawler_wrapper_rtdesc"
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  scanned index "hecheng_wrapper_rtdesc_pkey" to remove 563935 row versions
DETAIL:  CPU 0.20s/1.50u sec elapsed 1.70 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  scanned index "idx_hechenng_rtdesc_treeid" to remove 563935 row versions
DETAIL:  CPU 0.40s/1.51u sec elapsed 1.93 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  scanned index "idx_hechenng_rtdesc_hotelseq_status" to remove 563935 row versions
DETAIL:  CPU 0.60s/1.76u sec elapsed 2.39 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  scanned index "idx_crawler_wrapper_rtdesc_crawldate" to remove 563935 row versions
DETAIL:  CPU 1.09s/1.53u sec elapsed 3.94 sec.


psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  scanned index "crawler_wrapper_rtdesc_wrapper_id_crawl_date_idx" to remove 563935 row versions
DETAIL:  CPU 1.00s/1.69u sec elapsed 4.21 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  scanned index "idx_crawler_wrapper_rtdesc_wrapperid_operstatus_updated" to remove 563935 row versions
DETAIL:  CPU 0.83s/1.51u sec elapsed 3.00 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  scanned index "crawler_wrapper_rtdesc_updated_idx" to remove 563935 row versions
DETAIL:  CPU 0.64s/1.34u sec elapsed 2.46 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  "crawler_wrapper_rtdesc": removed 563935 row versions in 163782 pages
DETAIL:  CPU 0.24s/0.57u sec elapsed 0.82 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "hecheng_wrapper_rtdesc_pkey" now contains 3438239 row versions in 31073 pages
DETAIL:  538489 index row versions were removed.
1 index pages have been deleted, 0 are currently reusable.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "idx_hechenng_rtdesc_treeid" now contains 3438239 row versions in 29531 pages
DETAIL:  521093 index row versions were removed.
4714 index pages have been deleted, 0 are currently reusable.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "idx_hechenng_rtdesc_hotelseq_status" now contains 3438239 row versions in 47206 pages
DETAIL:  547609 index row versions were removed.
4860 index pages have been deleted, 0 are currently reusable.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "idx_crawler_wrapper_rtdesc_crawldate" now contains 3438239 row versions in 157959 pages
DETAIL:  563935 index row versions were removed.
24262 index pages have been deleted, 16553 are currently reusable.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "crawler_wrapper_rtdesc_wrapper_id_crawl_date_idx" now contains 3438239 row versions in 139861 pages
DETAIL:  563935 index row versions were removed.
46116 index pages have been deleted, 12668 are currently reusable.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "idx_crawler_wrapper_rtdesc_wrapperid_operstatus_updated" now contains 3438235 row versions in 120584 pages
DETAIL:  374482 index row versions were removed.
72032 index pages have been deleted, 39128 are currently reusable.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "crawler_wrapper_rtdesc_updated_idx" now contains 3438239 row versions in 96239 pages
DETAIL:  563935 index row versions were removed.
58646 index pages have been deleted, 57634 are currently reusable.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  "crawler_wrapper_rtdesc": found 282407 removable, 3437724 nonremovable row versions in 391205 out of 391333 pages
DETAIL:  0 dead row versions cannot be removed yet.
There were 17057284 unused item pointers.
0 pages are entirely empty.
CPU 7.03s/13.54u sec elapsed 24.71 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  vacuuming "pg_toast.pg_toast_16940642"
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  scanned index "pg_toast_16940642_index" to remove 413151 row versions
DETAIL:  CPU 0.19s/1.45u sec elapsed 1.71 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  "pg_toast_16940642": removed 413151 row versions in 129411 pages
DETAIL:  CPU 0.91s/0.40u sec elapsed 1.31 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "pg_toast_16940642_index" now contains 6576652 row versions in 140655 pages
DETAIL:  413151 index row versions were removed.
9153 index pages have been deleted, 3576 are currently reusable.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  "pg_toast_16940642": found 364889 removable, 5763282 nonremovable row versions in 2251437 out of 2508399 pages
DETAIL:  0 dead row versions cannot be removed yet.
There were 5729845 unused item pointers.
0 pages are entirely empty.
CPU 15.41s/8.92u sec elapsed 31.02 sec.
VACUUM

```



```
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# ls -lth
total 102G
-rw-r--r-- 1 root root  72G Dec 30 19:38 crawler_wrapper_rtdesc2.data
-rw-r--r-- 1 root root  90M Dec 30 15:36 crawler_wrapper_rtdesc.data
-rw-r--r-- 1 root root  25G Dec 30 14:55 dump2.data
-rw-r--r-- 1 root root 401M Dec 29 21:51 dump.log
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# 

```



```
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# ls -lth
total 102G
-rw-r--r-- 1 root root  72G Dec 30 19:38 crawler_wrapper_rtdesc2.data
-rw-r--r-- 1 root root 3.1K Dec 30 19:29 vacuum_crawler_wrapper_rtdesc.log
-rw-r--r-- 1 root root 1.3M Dec 30 18:45 crawler_wrapper_rtdesc.sh.log
-rw-r--r-- 1 root root   37 Dec 30 16:01 reindex_crawler_wrapper_rtdesc.sql
-rw-r--r-- 1 root root   42 Dec 30 16:00 reindex_toast_crawler_wrapper_rtdesc.sql
-rw-r--r-- 1 root root   39 Dec 30 15:59 analyze_crawler_wrapper_rtdesc.sql
-rw-r--r-- 1 root root   38 Dec 30 15:58 vacuum_crawler_wrapper_rtdesc.sql
-rw-r--r-- 1 root root  610 Dec 30 15:56 crawler_wrapper_rtdesc.sh
-rw-r--r-- 1 root root  90M Dec 30 15:36 crawler_wrapper_rtdesc.data
-rw-r--r-- 1 root root  25G Dec 30 14:55 dump2.data
-rw-r--r-- 1 root root   23 Dec 29 22:10 reindx.log
-rw-r--r-- 1 root root  111 Dec 29 21:58 reindx.sql
-rw-r--r-- 1 root root 401M Dec 29 21:51 dump.log
-rw-r--r-- 1 root root 234K Dec 29 21:20 t.sh.log
-rw-r--r-- 1 root root    0 Dec 29 21:14 t2.sh
-rw-r--r-- 1 root root  515 Dec 29 20:23 t.sh
[root@l-closeloopdb1.h.cn2 /export/db_shell/test]# 

```



```
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  vacuuming "public.crawler_wrapper_rtdesc"
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "hecheng_wrapper_rtdesc_pkey" now contains 3438239 row versions in 31073 pages
DETAIL:  0 index row versions were removed.
1 index pages have been deleted, 0 are currently reusable.
CPU 0.07s/0.02u sec elapsed 0.09 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "idx_hechenng_rtdesc_treeid" now contains 3438239 row versions in 29531 pages
DETAIL:  0 index row versions were removed.
4714 index pages have been deleted, 0 are currently reusable.
CPU 0.06s/0.02u sec elapsed 0.08 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "idx_hechenng_rtdesc_hotelseq_status" now contains 3438239 row versions in 47206 pages
DETAIL:  0 index row versions were removed.
4860 index pages have been deleted, 0 are currently reusable.
CPU 0.10s/0.03u sec elapsed 0.14 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "idx_crawler_wrapper_rtdesc_crawldate" now contains 3438239 row versions in 157959 pages
DETAIL:  0 index row versions were removed.
24262 index pages have been deleted, 16553 are currently reusable.
CPU 0.37s/0.16u sec elapsed 0.53 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "crawler_wrapper_rtdesc_wrapper_id_crawl_date_idx" now contains 3438239 row versions in 139861 pages
DETAIL:  0 index row versions were removed.
46139 index pages have been deleted, 12668 are currently reusable.
CPU 0.41s/0.25u sec elapsed 0.66 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "idx_crawler_wrapper_rtdesc_wrapperid_operstatus_updated" now contains 3438235 row versions in 120584 pages
DETAIL:  0 index row versions were removed.
72050 index pages have been deleted, 39128 are currently reusable.
CPU 0.35s/0.18u sec elapsed 0.53 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "crawler_wrapper_rtdesc_updated_idx" now contains 3438239 row versions in 96239 pages
DETAIL:  0 index row versions were removed.
58646 index pages have been deleted, 57634 are currently reusable.
CPU 0.30s/0.12u sec elapsed 0.42 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  "crawler_wrapper_rtdesc": found 0 removable, 3423997 nonremovable row versions in 386481 out of 391333 pages
DETAIL:  0 dead row versions cannot be removed yet.
There were 17553292 unused item pointers.
0 pages are entirely empty.
CPU 3.03s/2.44u sec elapsed 5.48 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  vacuuming "pg_toast.pg_toast_16940642"
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  index "pg_toast_16940642_index" now contains 6576652 row versions in 140655 pages
DETAIL:  0 index row versions were removed.
9153 index pages have been deleted, 3576 are currently reusable.
CPU 0.24s/0.26u sec elapsed 0.50 sec.
psql:vacuum_crawler_wrapper_rtdesc.sql:1: INFO:  "pg_toast_16940642": found 0 removable, 885292 nonremovable row versions in 364902 out of 2508399 pages
DETAIL:  0 dead row versions cannot be removed yet.
There were 1093490 unused item pointers.
0 pages are entirely empty.
CPU 1.76s/1.31u sec elapsed 3.07 sec.
VACUUM

```







Meanwhile, `max_standby_streaming_delay` and `max_standby_archive_delay` parameters on the standby server might need be 



# Investigation





