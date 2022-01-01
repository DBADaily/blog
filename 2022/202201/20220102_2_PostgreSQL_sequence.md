# 前言

本文是 sequence 系列继[三大数据库 sequence 之华山论剑](20220102_1_sequence.md) (Oracle PostgreSQL MySQL sequence 十年经验总结) 之后的第二篇，主要分享一下 PostgreSQL 中关于 sequence 的一些经验。

# 测试环境准备

以下测试是在 PostgreSQL 11 中进行。

通过以下 SQL 创建：

测试用户: alvin，普通用户，非 superuser

测试数据库: alvindb，owner 是 alvin

测试 schema: alvin，owner 也是 alvin

这里采用的是 user 与 schema 同名，结合默认的 `search_path`("$user", public)，这样操作对象(table, sequence, etc.)时就不需要加 schema 前缀了。

```
postgres=# CREATE USER alvin WITH PASSWORD 'alvin';
CREATE ROLE
postgres=# CREATE DATABASE alvindb OWNER alvin;
CREATE DATABASE
postgres=# \c alvindb
You are now connected to database "alvindb" as user "postgres".
alvindb=# CREATE SCHEMA alvin AUTHORIZATION alvin;
CREATE SCHEMA
alvindb=# \c alvindb alvin
You are now connected to database "alvindb" as user "alvin".
alvindb=> SHOW search_path;
   search_path   
-----------------
 "$user", public
(1 row)
```

# 创建 sequence 的两种方式

sequence 常规用途是用作主键序列的生成。下面通过通过创建 sequence 及表来讨论 sequence 创建方式。

## 创建 sequence 方式一 直接创建

下面是一种简单方式直接创建 sequence 及表。

```
alvindb=> CREATE SEQUENCE tb_test_sequence_test_id_seq;
CREATE SEQUENCE
alvindb=> 
CREATE TABLE tb_test_sequence (
    test_id INTEGER DEFAULT nextval('alvin.tb_test_sequence_test_id_seq') PRIMARY KEY,
    create_time TIMESTAMP DEFAULT clock_timestamp()
);
CREATE TABLE
```

查看已创建的对象

```
alvindb=> \d
                    List of relations
 Schema |             Name             |   Type   | Owner 
--------+------------------------------+----------+-------
 alvin  | tb_test_sequence_test_id_seq | sequence | alvin
 alvin  | tb_test_sequence             | table    | alvin
(2 rows)
```

查看已创建对象的结构

```
alvindb=> \d tb_test_sequence
                                            Table "alvin.tb_test_sequence"
   Column    |            Type             | Collation | Nullable |                      Default                      
-------------+-----------------------------+-----------+----------+---------------------------------------------------
 test_id     | integer                     |           | not null | nextval('tb_test_sequence_test_id_seq'::regclass)
 create_time | timestamp without time zone |           |          | clock_timestamp()
Indexes:
    "tb_test_sequence_pkey" PRIMARY KEY, btree (test_id)

alvindb=> \d tb_test_sequence_test_id_seq
                Sequence "alvin.tb_test_sequence_test_id_seq"
  Type  | Start | Minimum |       Maximum       | Increment | Cycles? | Cache 
--------+-------+---------+---------------------+-----------+---------+-------
 bigint |     1 |       1 | 9223372036854775807 |         1 | no      |     1

alvindb=>
```

此时，我们会注意到，**问题一，列 `tb_test_sequence.test_id` 的类型是 integer，而创建的 sequence 默认类型是 bigint。**

这样没有问题，但如果类型一致的话会更好。

接下来，我们 drop sequence 的话，会发现，由于表依赖 sequence，所以不能单独 drop sequence。

```
alvindb=> DROP SEQUENCE tb_test_sequence_test_id_seq;
ERROR:  cannot drop sequence tb_test_sequence_test_id_seq because other objects depend on it
DETAIL:  default value for column test_id of table tb_test_sequence depends on sequence tb_test_sequence_test_id_seq
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
alvindb=> 
```

下面我们 drop 掉表 `tb_test_sequence`。

```
alvindb=> DROP TABLE tb_test_sequence;
DROP TABLE
alvindb=> \d
                    List of relations
 Schema |             Name             |   Type   | Owner 
--------+------------------------------+----------+-------
 alvin  | tb_test_sequence_test_id_seq | sequence | alvin
(1 row)
```

可以看到，**问题二，虽然表 drop 了，但 sequence 还在。**

这样会有什么问题呢？

**在一个大型的数据库系统中，我们可能会发现有好多孤立的 sequence，因为我们 drop 表时可能会忘记 drop 掉其对应的 sequence。**

现在先手动 drop 掉 sequence。

```
alvindb=> DROP SEQUENCE tb_test_sequence_test_id_seq;
DROP SEQUENCE
alvindb=> \d
Did not find any relations.
alvindb=> 
```

我们优化一下 SQL 来解决上述两个问题：

```
alvindb=> CREATE SEQUENCE tb_test_sequence_test_id_seq AS INTEGER;
CREATE SEQUENCE
alvindb=> 
CREATE TABLE tb_test_sequence (
    test_id INTEGER DEFAULT nextval('alvin.tb_test_sequence_test_id_seq') PRIMARY KEY,
    create_time TIMESTAMP DEFAULT clock_timestamp()
);
CREATE TABLE
alvindb=> ALTER SEQUENCE tb_test_sequence_test_id_seq OWNED BY tb_test_sequence.test_id;
ALTER SEQUENCE
```

上述 SQL 的作用是:

1. **创建 sequence 时指定类型，使列与 sequence 的类型保持一致**

2. **关联表的列与 sequence，使 drop 表或列时会自动 drop 与其关联的 sequence**

查看表结构，

```
alvindb=> \d tb_test_sequence
                                            Table "alvin.tb_test_sequence"
   Column    |            Type             | Collation | Nullable |                      Default                      
-------------+-----------------------------+-----------+----------+---------------------------------------------------
 test_id     | integer                     |           | not null | nextval('tb_test_sequence_test_id_seq'::regclass)
 create_time | timestamp without time zone |           |          | clock_timestamp()
Indexes:
    "tb_test_sequence_pkey" PRIMARY KEY, btree (test_id)

alvindb=> \d tb_test_sequence_test_id_seq
            Sequence "alvin.tb_test_sequence_test_id_seq"
  Type   | Start | Minimum |  Maximum   | Increment | Cycles? | Cache 
---------+-------+---------+------------+-----------+---------+-------
 integer |     1 |       1 | 2147483647 |         1 | no      |     1
Owned by: alvin.tb_test_sequence.test_id
```

可以看到，

1. 列 `tb_test_sequence.test_id` 与 sequence 的类型均为 integer
2. sequence 下方多了 'Owned by'，表示列与 sequence 已关联了。

下面 drop 表后，可以看到，sequence 也已被 drop 了。

```
alvindb=> DROP TABLE tb_test_sequence;
DROP TABLE
alvindb=> \d
Did not find any relations.
```

实际上，**如果 drop 掉列 test_id，其关联的 sequence 也会被 drop**。

```
alvindb=> ALTER TABLE tb_test_sequence DROP COLUMN test_id;
ALTER TABLE
alvindb=> \d tb_test_sequence
                            Table "alvin.tb_test_sequence"
   Column    |            Type             | Collation | Nullable |      Default      
-------------+-----------------------------+-----------+----------+-------------------
 create_time | timestamp without time zone |           |          | clock_timestamp()
 alvindb=> \d
             List of relations
 Schema |       Name       | Type  | Owner 
--------+------------------+-------+-------
 alvin  | tb_test_sequence | table | alvin
(1 row)
```

## 创建 sequence 方式二 通过 serial 创建

下面通过一个 SQL 来实现与上面完全相同的效果。

```
alvindb=> 
CREATE TABLE tb_test_sequence (
    test_id SERIAL PRIMARY KEY,
    create_time TIMESTAMP DEFAULT clock_timestamp()
);
CREATE TABLE
```

查看表结构，与方式一中完全一样。

```
alvindb=> \d
                    List of relations
 Schema |             Name             |   Type   | Owner 
--------+------------------------------+----------+-------
 alvin  | tb_test_sequence             | table    | alvin
 alvin  | tb_test_sequence_test_id_seq | sequence | alvin
(2 rows)
alvindb=> \d tb_test_sequence
                                            Table "alvin.tb_test_sequence"
   Column    |            Type             | Collation | Nullable |                      Default                      
-------------+-----------------------------+-----------+----------+---------------------------------------------------
 test_id     | integer                     |           | not null | nextval('tb_test_sequence_test_id_seq'::regclass)
 create_time | timestamp without time zone |           |          | clock_timestamp()
Indexes:
    "tb_test_sequence_pkey" PRIMARY KEY, btree (test_id)

alvindb=> \d tb_test_sequence_test_id_seq
            Sequence "alvin.tb_test_sequence_test_id_seq"
  Type   | Start | Minimum |  Maximum   | Increment | Cycles? | Cache 
---------+-------+---------+------------+-----------+---------+-------
 integer |     1 |       1 | 2147483647 |         1 | no      |     1
Owned by: alvin.tb_test_sequence.test_id
```

**这里总结一下一个单词 SERIAL 做了什么事情：**

1. **根据规则 `tablename_colname_seq` 创建 sequence，并设置 DEFAULT**
2. **增加 NOT NULL 约束**
3. **关联列与 sequence，使表或关联的列 drop 时，关联的 sequence 也会被 drop 掉**

注：这里 SERIAL 和 PRIMARY KEY 之一都会默认增加 NOT NULL 约束

**用 SERIAL 的确省了不少事，但它有什么问题吗？使用它会不会又引入了新的问题？**

1. **SERIAL 对应的数据类型是 integer，作为主键的数据类型，integer 足够吗？**
2. **关联列与 sequence 后，drop 时是方便了，但同时会不会给运维带来新的问题？比如 rename 表，列或 sequence？**
3. **在复制表或迁移表时，又该对 sequence 作何操作呢？**

接下来，我们从这几个问题出发进一步探讨。

# serial 与 bigserial

serial 对应的是 integer，是 4 个字节，最大值是 2 147 483 647，即 21 亿左右。

作为大表主键的 sequence，21 亿真的够吗？按全球人口 70 亿算，一人一个数都不够。

为解决这个问题，可以用 bigserial，即 bigint，8 个字节，最大值是 9 223 372 036 854 775 807，即 922亿个亿左右。这对于绝大多数场景是足够了，这也是 PostgreSQL 中 sequence 的最大值。

使用 bigserial 创建表：

```
alvindb=> 
CREATE TABLE tb_test_bigserial (
    test_id BIGSERIAL PRIMARY KEY,
    create_time TIMESTAMP DEFAULT clock_timestamp()
);
CREATE TABLE
```

查看表结构，

```
alvindb=> \d tb_test_bigserial
                                            Table "alvin.tb_test_bigserial"
   Column    |            Type             | Collation | Nullable |                      Default                       
-------------+-----------------------------+-----------+----------+----------------------------------------------------
 test_id     | bigint                      |           | not null | nextval('tb_test_bigserial_test_id_seq'::regclass)
 create_time | timestamp without time zone |           |          | clock_timestamp()
Indexes:
    "tb_test_bigserial_pkey" PRIMARY KEY, btree (test_id)

alvindb=> \d tb_test_bigserial_test_id_seq
                Sequence "alvin.tb_test_bigserial_test_id_seq"
  Type  | Start | Minimum |       Maximum       | Increment | Cycles? | Cache 
--------+-------+---------+---------------------+-----------+---------+-------
 bigint |     1 |       1 | 9223372036854775807 |         1 | no      |     1
Owned by: alvin.tb_test_bigserial.test_id
```

可以看到，列 `test_id` 和 sequence 的 Type 都是 bigint。这样，sequence 的类型问题就解决了。

# rename 对 sequence 的影响

关联列与 sequence 后，即 sequence 属于该列后，drop 表或列时会自动 drop 相关 sequence。

但如果对表或列 rename 后，甚至 rename sequence后，会发生什么呢？

我们来做一下实验。

创建测试表 `tb_test_sequence_rename`

```
alvindb=>
CREATE TABLE tb_test_sequence_rename (
    test_id BIGSERIAL PRIMARY KEY,
    create_time TIMESTAMP DEFAULT clock_timestamp()
);
CREATE TABLE
```

查看表与 sequence：

```
alvindb=> \d tb_test_sequence_rename
                                            Table "alvin.tb_test_sequence_rename"
   Column    |            Type             | Collation | Nullable |                         Default                          
-------------+-----------------------------+-----------+----------+----------------------------------------------------------
 test_id     | bigint                      |           | not null | nextval('tb_test_sequence_rename_test_id_seq'::regclass)
 create_time | timestamp without time zone |           |          | clock_timestamp()
Indexes:
    "tb_test_sequence_rename_pkey" PRIMARY KEY, btree (test_id)

alvindb=> \d tb_test_sequence_rename_test_id_seq
             Sequence "alvin.tb_test_sequence_rename_test_id_seq"
  Type  | Start | Minimum |       Maximum       | Increment | Cycles? | Cache 
--------+-------+---------+---------------------+-----------+---------+-------
 bigint |     1 |       1 | 9223372036854775807 |         1 | no      |     1
Owned by: alvin.tb_test_sequence_rename.test_id

alvindb=> 
```

对表进行 rename ：

```
alvindb=> ALTER TABLE tb_test_sequence_rename RENAME TO tb_test_sequence_rename2;
ALTER TABLE
```

通过如下结果，我们可以看到， **rename 表后 'Owned by' 也会随之自动变化**。

```
alvindb=> \d tb_test_sequence_rename2
                                           Table "alvin.tb_test_sequence_rename2"
   Column    |            Type             | Collation | Nullable |                         Default                          
-------------+-----------------------------+-----------+----------+----------------------------------------------------------
 test_id     | bigint                      |           | not null | nextval('tb_test_sequence_rename_test_id_seq'::regclass)
 create_time | timestamp without time zone |           |          | clock_timestamp()
Indexes:
    "tb_test_sequence_rename_pkey" PRIMARY KEY, btree (test_id)

alvindb=> \d tb_test_sequence_rename_test_id_seq
             Sequence "alvin.tb_test_sequence_rename_test_id_seq"
  Type  | Start | Minimum |       Maximum       | Increment | Cycles? | Cache 
--------+-------+---------+---------------------+-----------+---------+-------
 bigint |     1 |       1 | 9223372036854775807 |         1 | no      |     1
Owned by: alvin.tb_test_sequence_rename2.test_id

alvindb=> 
```

我们再 rename 列试一下，

```
alvindb=> ALTER TABLE tb_test_sequence_rename2 RENAME test_id TO test_id2;
ALTER TABLE
```

通过如下结果，我们可以看到， **rename 列后 'Owned by' 也会随之自动变化**。

```
alvindb=> \d tb_test_sequence_rename2
                                           Table "alvin.tb_test_sequence_rename2"
   Column    |            Type             | Collation | Nullable |                         Default                          
-------------+-----------------------------+-----------+----------+----------------------------------------------------------
 test_id2    | bigint                      |           | not null | nextval('tb_test_sequence_rename_test_id_seq'::regclass)
 create_time | timestamp without time zone |           |          | clock_timestamp()
Indexes:
    "tb_test_sequence_rename_pkey" PRIMARY KEY, btree (test_id2)

alvindb=> \d tb_test_sequence_rename_test_id_seq
             Sequence "alvin.tb_test_sequence_rename_test_id_seq"
  Type  | Start | Minimum |       Maximum       | Increment | Cycles? | Cache 
--------+-------+---------+---------------------+-----------+---------+-------
 bigint |     1 |       1 | 9223372036854775807 |         1 | no      |     1
Owned by: alvin.tb_test_sequence_rename2.test_id2

alvindb=> 
```

我们来 rename 一下 sequence，

```
alvindb=> ALTER SEQUENCE tb_test_sequence_rename_test_id_seq RENAME TO tb_test_sequence_rename_test_id_seq2;
ALTER SEQUENCE
```

通过如下结果，我们可以看到， **rename sequence 后 'Owned by' 也会随之自动变化，并且 Default 中的 sequence 也会随之变化**。

```
alvindb=> \d tb_test_sequence_rename2
                                            Table "alvin.tb_test_sequence_rename2"
   Column    |            Type             | Collation | Nullable |                          Default                          
-------------+-----------------------------+-----------+----------+-----------------------------------------------------------
 test_id2    | bigint                      |           | not null | nextval('tb_test_sequence_rename_test_id_seq2'::regclass)
 create_time | timestamp without time zone |           |          | clock_timestamp()
Indexes:
    "tb_test_sequence_rename_pkey" PRIMARY KEY, btree (test_id2)

alvindb=> \d tb_test_sequence_rename_test_id_seq2
            Sequence "alvin.tb_test_sequence_rename_test_id_seq2"
  Type  | Start | Minimum |       Maximum       | Increment | Cycles? | Cache 
--------+-------+---------+---------------------+-----------+---------+-------
 bigint |     1 |       1 | 9223372036854775807 |         1 | no      |     1
Owned by: alvin.tb_test_sequence_rename2.test_id2

alvindb=> 
```

通过以上三个 rename 实验，可以发现，**正常 rename 不会对 sequence 的使用产生影响**。

无论是 rename 表名，列名，还是 sequence 的名字，如我们所期望，PostgreSQL 都会智能地作出相应的修改。

# 复制表或迁移表时 sequence 的相关操作

以下表为例，

```
alvindb=> \d tb_test_bigserial
                                            Table "alvin.tb_test_bigserial"
   Column    |            Type             | Collation | Nullable |                      Default                       
-------------+-----------------------------+-----------+----------+----------------------------------------------------
 test_id     | bigint                      |           | not null | nextval('tb_test_bigserial_test_id_seq'::regclass)
 create_time | timestamp without time zone |           |          | clock_timestamp()
Indexes:
    "tb_test_bigserial_pkey" PRIMARY KEY, btree (test_id)
alvindb=> \d tb_test_bigserial_test_id_seq
                Sequence "alvin.tb_test_bigserial_test_id_seq"
  Type  | Start | Minimum |       Maximum       | Increment | Cycles? | Cache 
--------+-------+---------+---------------------+-----------+---------+-------
 bigint |     1 |       1 | 9223372036854775807 |         1 | no      |     1
Owned by: alvin.tb_test_bigserial.test_id
```

**在一些情况下，为避免 DDL 或大量 DML 对表 `tb_test_bigserial` 的影响，我们可以通过 RENAME 表的方式实现：**

1. 根据表 `tb_test_bigserial` 复制出相同表结构的表 `tb_test_bigserial_new`
2. 对 `tb_test_bigserial_new` 进行 DDL 或 大量 DML 
3. 将 `tb_test_bigserial_new` RENAME 回 `tb_test_bigserial`

根据表 `tb_test_bigserial` 复制出相同表结构的表 `tb_test_bigserial_new`：

```
alvindb=> CREATE TABLE tb_test_bigserial_new (LIKE tb_test_bigserial INCLUDING ALL);
CREATE TABLE
alvindb=> \d tb_test_bigserial_new
                                          Table "alvin.tb_test_bigserial_new"
   Column    |            Type             | Collation | Nullable |                      Default                       
-------------+-----------------------------+-----------+----------+----------------------------------------------------
 test_id     | bigint                      |           | not null | nextval('tb_test_bigserial_test_id_seq'::regclass)
 create_time | timestamp without time zone |           |          | clock_timestamp()
Indexes:
    "tb_test_bigserial_new_pkey" PRIMARY KEY, btree (test_id)
```

对 `tb_test_bigserial_new` 进行 DDL 或 大量 DML 后，将 `tb_test_bigserial_new` RENAME 回 `tb_test_bigserial`：

```
alvindb=> BEGIN;
BEGIN
alvindb=> ALTER TABLE tb_test_bigserial RENAME TO tb_test_bigserial_old;
ALTER TABLE
alvindb=> ALTER TABLE tb_test_bigserial_new RENAME TO tb_test_bigserial;
ALTER TABLE
alvindb=> END;
COMMIT
```

这时，查看新表结构：

```
alvindb=> \d tb_test_bigserial
                                            Table "alvin.tb_test_bigserial"
   Column    |            Type             | Collation | Nullable |                      Default                       
-------------+-----------------------------+-----------+----------+----------------------------------------------------
 test_id     | bigint                      |           | not null | nextval('tb_test_bigserial_test_id_seq'::regclass)
 create_time | timestamp without time zone |           |          | clock_timestamp()
Indexes:
    "tb_test_bigserial_new_pkey" PRIMARY KEY, btree (test_id)
```

此处，我们只关注 sequence。上述的索引的名字可以根据需要决定是否需要 RENAME 回原来的名字。

查看 sequence，

```
alvindb=> \d tb_test_bigserial_test_id_seq
                Sequence "alvin.tb_test_bigserial_test_id_seq"
  Type  | Start | Minimum |       Maximum       | Increment | Cycles? | Cache 
--------+-------+---------+---------------------+-----------+---------+-------
 bigint |     1 |       1 | 9223372036854775807 |         1 | no      |     1
Owned by: alvin.tb_test_bigserial_old.test_id
```

从以上 'Owned by' 可以看出，sequence `tb_test_bigserial_test_id_seq`  还是归旧表 `tb_test_bigserial_old` 的列所有。

从上文“rename 对 sequence 的影响”我们知道，这是正常的。

此时 DROP 旧表，会提示新表 `tb_test_bigserial` 还在依赖 sequence `tb_test_bigserial_test_id_seq`。

```
alvindb=> DROP TABLE tb_test_bigserial_old;
ERROR:  cannot drop table tb_test_bigserial_old because other objects depend on it
DETAIL:  default value for column test_id of table tb_test_bigserial depends on sequence tb_test_bigserial_test_id_seq
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
alvindb=> 
```

**以下 RENAME 表时关键的一步：**

```
alvindb=> ALTER SEQUENCE tb_test_bigserial_test_id_seq OWNED BY tb_test_bigserial.test_id;
ALTER SEQUENCE
```

通过上述 SQL，sequence `tb_test_bigserial_test_id_seq` 就归新表的列所有了。

**在日常操作中，我们有可能忘记修改 sequence 的所属关系。以致后来 DROP 老表时加了 CASCADE，将 sequence 也一起 DROP 掉了，从而引发问题。**

此时 DROP 表就不报错了：

```
alvindb=> DROP TABLE tb_test_bigserial_old;
DROP TABLE
```

以下是 RENAME 后所期望的结果（注意 sequence 的 'Owned by'）：

```

alvindb=> \d tb_test_bigserial
                                            Table "alvin.tb_test_bigserial"
   Column    |            Type             | Collation | Nullable |                      Default                       
-------------+-----------------------------+-----------+----------+----------------------------------------------------
 test_id     | bigint                      |           | not null | nextval('tb_test_bigserial_test_id_seq'::regclass)
 create_time | timestamp without time zone |           |          | clock_timestamp()
Indexes:
    "tb_test_bigserial_new_pkey" PRIMARY KEY, btree (test_id)

alvindb=> \d tb_test_bigserial_test_id_seq
                Sequence "alvin.tb_test_bigserial_test_id_seq"
  Type  | Start | Minimum |       Maximum       | Increment | Cycles? | Cache 
--------+-------+---------+---------------------+-----------+---------+-------
 bigint |     1 |       1 | 9223372036854775807 |         1 | no      |     1
Owned by: alvin.tb_test_bigserial.test_id
```

# sequence cache

从上面可以看到，sequence 默认的 cache 是1。

**在高并发的系统中，为了提高性能，可以增大 cache，如 10，即一次将 sequence 的10个数值 load 到内存中。这样，只需要访问 sequence 1 次，而不是10次。**

下面，将 sequence `tb_test_bigserial_test_id_seq` 的 cache 改成 10：

```
alvindb=> SELECT pg_backend_pid();
 pg_backend_pid 
----------------
          16649
(1 row)
alvindb=> ALTER SEQUENCE tb_test_bigserial_test_id_seq CACHE 10;
ALTER SEQUENCE
alvindb=> \d tb_test_bigserial_test_id_seq
                Sequence "alvin.tb_test_bigserial_test_id_seq"
  Type  | Start | Minimum |       Maximum       | Increment | Cycles? | Cache 
--------+-------+---------+---------------------+-----------+---------+-------
 bigint |     1 |       1 | 9223372036854775807 |         1 | no      |    10
Owned by: alvin.tb_test_bigserial.test_id
```

在 session 1(pid 16649)中插入数据

```
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> SELECT * FROM tb_test_bigserial ORDER BY 2 DESC;
 test_id |        create_time         
---------+----------------------------
       2 | 2021-05-01 15:21:00.116979
       1 | 2021-05-01 15:20:56.405149
(2 rows)

alvindb=> SELECT * FROM tb_test_bigserial_test_id_seq;
 last_value | log_cnt | is_called 
------------+---------+-----------
         10 |      32 | t
(1 row)
alvindb=> SELECT lastval();
 lastval 
---------
       2
(1 row)
```

从以上结果可以看到，此 session 中一次取了10个数据，`last_value` 是10，`is_called` 是 true，即新的 session 中从11开始。

lastval() 是2，即当前 session 中，上一次 nextval 返回的是2。需要注意，lastval() 并没有参数，即它返回最近任意 sequence 的 nextval 值。

如下，

```
alvindb=> SELECT nextval('tb_test_sequence_rename_test_id_seq2');
 nextval 
---------
       1
(1 row)

alvindb=> SELECT lastval();
 lastval 
---------
       1
(1 row)
```

在 session 2(pid 14287)中插入数据：

```
alvindb=> SELECT pg_backend_pid();
 pg_backend_pid 
----------------
          14287
(1 row)
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> SELECT * FROM tb_test_bigserial ORDER BY 2 DESC;
 test_id |        create_time         
---------+----------------------------
      12 | 2021-05-01 15:22:02.582036
      11 | 2021-05-01 15:22:01.313919
       2 | 2021-05-01 15:21:00.116979
       1 | 2021-05-01 15:20:56.405149
(4 rows)
alvindb=> SELECT * FROM tb_test_bigserial_test_id_seq;
 last_value | log_cnt | is_called 
------------+---------+-----------
         20 |      32 | t
(1 row)
alvindb=> SELECT lastval();
 lastval 
---------
      12
(1 row)
```

从以上结果可以看到，虽然在 session 1中，sequence `tb_test_bigserial_test_id_seq` 只取了两次值，但 session 2中是从11开始取值的。

因为 cache 是10，所以1-10是预留给了 session 1。11-20是预留给了 session 2。

在 session 2中，`last_value` 是20，lastval()是12。

如果现在再看 session 1呢？

```
alvindb=> SELECT pg_backend_pid();
 pg_backend_pid 
----------------
          16649
(1 row)

alvindb=> SELECT now();
              now              
-------------------------------
 2021-05-01 15:23:54.674943+08
(1 row)

alvindb=> SELECT * FROM tb_test_bigserial_test_id_seq;
 last_value | log_cnt | is_called 
------------+---------+-----------
         20 |      32 | t
(1 row)

alvindb=> SELECT lastval();
 lastval 
---------
       2
(1 row)
```

现在看到 session 1中的 `last_value` 变成了 20，而 lastval() 还是2。

对比 session 1 和 session 2，可以看出，`last_value` 是与 session 无关的。而 lastval() 是与 session 有关，但与是哪个 sequence 无关。

现在，在 session 3 中验证一下，

```
alvindb=> SELECT pg_backend_pid();
 pg_backend_pid 
----------------
          15809
(1 row)

alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> SELECT * FROM tb_test_bigserial ORDER BY 2 DESC;
 test_id |        create_time         
---------+----------------------------
      22 | 2021-05-01 15:25:08.643196
      21 | 2021-05-01 15:25:04.005421
      12 | 2021-05-01 15:22:02.582036
      11 | 2021-05-01 15:22:01.313919
       2 | 2021-05-01 15:21:00.116979
       1 | 2021-05-01 15:20:56.405149
(6 rows)
alvindb=> SELECT now();
              now              
-------------------------------
 2021-05-01 15:25:24.499995+08
(1 row)

alvindb=> SELECT * FROM tb_test_bigserial_test_id_seq;
 last_value | log_cnt | is_called 
------------+---------+-----------
         30 |      32 | t
(1 row)

alvindb=> SELECT lastval();
 lastval 
---------
      22
(1 row)
```

可以看出，以上推论是正确的。session 3 中 sequence 是从21开始的，插入两行数据后，`last_value` 变成 30，lastval() 在当前 session 中是22。

**现在有一个问题，后插入的数据的 `test_id` 一定比先插入的大吗？能否根据 `test_id` 排序来确定是哪一行数据先插入的呢？**

下面，我们回到 session 2 继续插入数据，

```
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> SELECT * FROM tb_test_bigserial ORDER BY 2 DESC;
 test_id |        create_time         
---------+----------------------------
      14 | 2021-05-01 15:28:16.500833
      13 | 2021-05-01 15:28:04.276826
      22 | 2021-05-01 15:25:08.643196
      21 | 2021-05-01 15:25:04.005421
      12 | 2021-05-01 15:22:02.582036
      11 | 2021-05-01 15:22:01.313919
       2 | 2021-05-01 15:21:00.116979
       1 | 2021-05-01 15:20:56.405149
(8 rows)
```

可以看出，在 session 2中，sequence 继续从 13 开始。这是因为 session 2中一次性 cache 的 10 个数值(11-20)还没有用完。

**所以，如果 cache 设置大于1，就不能以 sequence 的大小判断其插入的顺序。**

**可以用 `create_time` 来排序。**但需要注意，**在事务中，`now()` 返回的时间是一样的，而 `clock_timestamp()` 返回的则是实际的时间，**

这也是在创建表 `tb_test_bigserial` 时采用 `clock_timestamp()` 而不是 `now()` 的原因。

**现在如果调用 setval 会发生什么呢？**

我们在 session 1中调用 setval，

```
alvindb=> SELECT pg_backend_pid();
 pg_backend_pid 
----------------
          16649
(1 row)
alvindb=> SELECT setval('tb_test_bigserial_test_id_seq', 51);
 setval 
--------
     51
(1 row)
alvindb=> SELECT * FROM tb_test_bigserial_test_id_seq;
 last_value | log_cnt | is_called 
------------+---------+-----------
         51 |       0 | t
(1 row)
alvindb=> SELECT lastval();
 lastval 
---------
      51
(1 row)
```

上述 SQL 已将 `last_value` 设为 51，即下一个返回 52。

现在我们在 session 2中继续插入，

```
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> SELECT * FROM tb_test_bigserial ORDER BY 2 DESC;
 test_id |        create_time         
---------+----------------------------
      18 | 2021-05-01 15:28:22.692747
      17 | 2021-05-01 15:28:22.371691
      16 | 2021-05-01 15:28:22.021806
      15 | 2021-05-01 15:28:20.917822
      14 | 2021-05-01 15:28:16.500833
      13 | 2021-05-01 15:28:04.276826
      22 | 2021-05-01 15:25:08.643196
      21 | 2021-05-01 15:25:04.005421
      12 | 2021-05-01 15:22:02.582036
      11 | 2021-05-01 15:22:01.313919
       2 | 2021-05-01 15:21:00.116979
       1 | 2021-05-01 15:20:56.405149
(12 rows)
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> SELECT * FROM tb_test_bigserial ORDER BY 2 DESC;
 test_id |        create_time         
---------+----------------------------
      19 | 2021-05-01 15:28:26.485763
      18 | 2021-05-01 15:28:22.692747
      17 | 2021-05-01 15:28:22.371691
      16 | 2021-05-01 15:28:22.021806
      15 | 2021-05-01 15:28:20.917822
      14 | 2021-05-01 15:28:16.500833
      13 | 2021-05-01 15:28:04.276826
      22 | 2021-05-01 15:25:08.643196
      21 | 2021-05-01 15:25:04.005421
      12 | 2021-05-01 15:22:02.582036
      11 | 2021-05-01 15:22:01.313919
       2 | 2021-05-01 15:21:00.116979
       1 | 2021-05-01 15:20:56.405149
(13 rows)

alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> SELECT * FROM tb_test_bigserial ORDER BY 2 DESC;
 test_id |        create_time         
---------+----------------------------
      20 | 2021-05-01 15:28:29.747721
      19 | 2021-05-01 15:28:26.485763
      18 | 2021-05-01 15:28:22.692747
      17 | 2021-05-01 15:28:22.371691
      16 | 2021-05-01 15:28:22.021806
      15 | 2021-05-01 15:28:20.917822
      14 | 2021-05-01 15:28:16.500833
      13 | 2021-05-01 15:28:04.276826
      22 | 2021-05-01 15:25:08.643196
      21 | 2021-05-01 15:25:04.005421
      12 | 2021-05-01 15:22:02.582036
      11 | 2021-05-01 15:22:01.313919
       2 | 2021-05-01 15:21:00.116979
       1 | 2021-05-01 15:20:56.405149
(14 rows)

alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> SELECT * FROM tb_test_bigserial ORDER BY 2 DESC;
 test_id |        create_time         
---------+----------------------------
      52 | 2021-05-01 15:28:32.084771
      20 | 2021-05-01 15:28:29.747721
      19 | 2021-05-01 15:28:26.485763
      18 | 2021-05-01 15:28:22.692747
      17 | 2021-05-01 15:28:22.371691
      16 | 2021-05-01 15:28:22.021806
      15 | 2021-05-01 15:28:20.917822
      14 | 2021-05-01 15:28:16.500833
      13 | 2021-05-01 15:28:04.276826
      22 | 2021-05-01 15:25:08.643196
      21 | 2021-05-01 15:25:04.005421
      12 | 2021-05-01 15:22:02.582036
      11 | 2021-05-01 15:22:01.313919
       2 | 2021-05-01 15:21:00.116979
       1 | 2021-05-01 15:20:56.405149
(15 rows)

alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
alvindb=> SELECT * FROM tb_test_bigserial ORDER BY 2 DESC;
 test_id |        create_time         
---------+----------------------------
      53 | 2021-05-01 15:28:45.026188
      52 | 2021-05-01 15:28:32.084771
      20 | 2021-05-01 15:28:29.747721
      19 | 2021-05-01 15:28:26.485763
      18 | 2021-05-01 15:28:22.692747
      17 | 2021-05-01 15:28:22.371691
      16 | 2021-05-01 15:28:22.021806
      15 | 2021-05-01 15:28:20.917822
      14 | 2021-05-01 15:28:16.500833
      13 | 2021-05-01 15:28:04.276826
      22 | 2021-05-01 15:25:08.643196
      21 | 2021-05-01 15:25:04.005421
      12 | 2021-05-01 15:22:02.582036
      11 | 2021-05-01 15:22:01.313919
       2 | 2021-05-01 15:21:00.116979
       1 | 2021-05-01 15:20:56.405149
(16 rows)
```

**可以看出，setval 后，session 2中并没有立即生效，而是等 session 2中预分配的10个数值消耗完，才继续从 sequence 又 cache 了10个数值。**

另外，可以看出，上述数据中 `test_id` 列的数据并不是连续的。这也是使用 cache 的原因。

**所以，在高并发的系统中，为了提高 sequence 的性能，不仅仅是增大 cache，以上提到的问题都需要注意。**

# sequence 权限

创建用户 simon。

```
postgres=# CREATE USER simon WITH PASSWORD 'simon';
CREATE ROLE
```

现在将表 `tb_test_bigserial` 的读写权限授予 simon

```
alvindb=# \c alvindb alvin;
You are now connected to database "alvindb" as user "alvin".
alvindb=> \dp+ tb_test_bigserial
                                   Access privileges
 Schema |       Name        | Type  | Access privileges | Column privileges | Policies 
--------+-------------------+-------+-------------------+-------------------+----------
 alvin  | tb_test_bigserial | table |                   |                   | 
(1 row)

alvindb=> GRANT USAGE ON SCHEMA alvin TO "simon";
GRANT
alvindb=> GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE ON alvin.tb_test_bigserial TO "simon";
GRANT
alvindb=> \dp+ tb_test_bigserial
                                    Access privileges
 Schema |       Name        | Type  |  Access privileges  | Column privileges | Policies 
--------+-------------------+-------+---------------------+-------------------+----------
 alvin  | tb_test_bigserial | table | alvin=arwdDxt/alvin+|                   | 
        |                   |       | simon=arwdD/alvin   |                   | 
(1 row)
```

通过用户 simon 插入数据

```
alvindb=> \c alvindb simon 
You are now connected to database "alvindb" as user "simon".
alvindb=> SET search_path TO alvin;
SET
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
ERROR:  permission denied for sequence tb_test_bigserial_test_id_seq
```

此时插入数据提示没有 sequence `tb_test_bigserial_test_id_seq` 的权限。

现在将 sequence `tb_test_bigserial_test_id_seq` 也授权一下:

```
alvindb=> \c alvindb alvin
You are now connected to database "alvindb" as user "alvin".
alvindb=> GRANT ALL ON SEQUENCE tb_test_bigserial_test_id_seq TO "simon";
GRANT
alvindb=> \dp+ tb_test_bigserial
                                    Access privileges
 Schema |       Name        | Type  |  Access privileges  | Column privileges | Policies 
--------+-------------------+-------+---------------------+-------------------+----------
 alvin  | tb_test_bigserial | table | alvin=arwdDxt/alvin+|                   | 
        |                   |       | simon=arwdD/alvin   |                   | 
(1 row)

alvindb=> \dp+ tb_test_bigserial_test_id_seq
                                          Access privileges
 Schema |             Name              |   Type   | Access privileges | Column privileges | Policies 
--------+-------------------------------+----------+-------------------+-------------------+----------
 alvin  | tb_test_bigserial_test_id_seq | sequence | alvin=rwU/alvin  +|                   | 
        |                               |          | simon=rwU/alvin   |                   | 
(1 row)
alvindb=> INSERT INTO tb_test_bigserial(create_time) VALUES (DEFAULT);
INSERT 0 1
```

将 sequence 的读写权限授予 simon 后，simon 就可以成功向表 `tb_test_bigserial` 插入数据了。

可以看出，**要对其他用户授予读写权限，不仅要授予表的权限，还需要授予对应 sequence 的权限。**

# 总结

sequence 看起来很简单，但如果使用姿势不对，很可能造成 MI (Major Incident)。

对于以下常见的有关 sequence 的问题，需要多留意。

1. integer vs bigint

   对于交易型的表，log 表，history 表等较大的表，主键 sequence 需要用 bigint。否则可能遇到如下错误：

   ```
   ERROR:  nextval: reached maximum value of sequence "tb_test_sequence_test_id_seq" (2147483647)
   ```

   其实，不止数据库需要考虑这个问题，应用程序中也需要考虑。

   存储主键值的变量等需要用相应的类型，如 Java 中对应的需要使用 long 类型而不是 int 类型。

   开发和测试同学可能都需要排查这个问题，以防止将来生产出现不必要的 MI。

   开发可以直接看代码或查看表的DDL。测试的话，可以通过 setval 来修改当前 sequence 的值进行测试。

   甚至有强制主键 id 必须用 bigint 的数据库规范来避免这个问题。

2. sequence 'Owned by'

   通过 rename 方式复制表时，需要注意同时修改 sequence 的 'Owned by'。

   否则，新表主键 default 中的 sequence 在 drop 老表时有可能被级联 drop 掉。

3. sequence nextval

   当涉及到表或 sequence 的迁移，或需要重新创建 sequence，需要注意重新设置 sequence 值。

4. sequence cache

   当 sequence cache 大于1时，需要注意排序时不要用 sequence 的列，可以考虑根据时间的列排序。

5. sequence 权限

   在一些情况下，不但要授予相关用户表的权限，也需要注意 sequence 的权限。

