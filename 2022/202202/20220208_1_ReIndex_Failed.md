# ReIndex 失败原因调查
# 背景

有的读写较多的集群部署了 ReIndex 定时任务，每天凌晨对一些大表重新创建索引 (PostgreSQL 11, 先 create index, 再 drop 原来的 index)。

# 处理报警

早上收到 ReIndex 失败报警后，先把 create index 失败造成的 INVALID index drop 掉。

# 调查原因

近一个月同事已经处理过好几次这样的报警了，为什么还会持续报警呢？我决定调查一下原因。

查看 ReIndex shell 脚本日志，发现是死锁原因造成的。

```
CREATE INDEX CONCURRENTLY ON public.tb_transaction USING gist (transaction_id) TABLESPACE pg_default ;
psql:reindex.sql:1: ERROR:  deadlock detected
DETAIL:  Process 19875 waits for ShareLock on virtual transaction 375/448560; blocked by process 20825.
Process 20825 waits for ShareUpdateExclusiveLock on relation 1007178 of database 1005837; blocked by process 19875.
HINT:  See server log for query details.
```

从 PostgreSQL 日志中，根据造成死锁的两个 pid, 查到了这两个 session 所有相关日志。查到另一个造成死锁执行的 SQL 是 VACUUM VERBOSE。 查看服务器所有定时任务，确认这是另一个定时任务中的。根据 ReIndex 失败的时间，进一步分析日志，原来是 VACUUM 任务在 vacuum 一个表时，另一个 ReIndex 的任务恰好要对这个表重新创建索引，造成死锁。

# 解决方案一

将 VACUUM 定时任务提前或将 ReIndex 任务延后，避免两个任务时间上有交集。
此方案能较快缓解问题，但随着时间的推移，数据库会变得越来越大，后续还可能存在两个任务重叠的情况。

# 解决方案二

将维护任务都放到一个 shell 脚本里，这样它们串行执行，就不会冲突了。

但目前 ReIndex 和 VACUUM 是两个不同的脚本，并不是所有集群都需要。

此方案通过配置管理自动化 ( Ansible, Salt 等) 可以更好地实现，即只需要配置各集群需要哪些定时任务 (如下 yaml 配置所示)，就可轻松实现同一脚本中包含不同的任务。

```
CluserA:
  - reindex
  - vacuum
ClusterB:
  - reindex
ClusterC:
  - vaccum
```

# 总结

解决问题更优的方式是查到根本原因并作优化，而不是简单把表面的问题处理掉，不然问题还会反复出现。
