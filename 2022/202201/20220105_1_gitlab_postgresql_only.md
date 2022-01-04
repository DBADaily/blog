# GitLab supports only PostgreSQL now

Several months ago I worked on migrating packaged PostgreSQL server included with Omnibus GitLab to external PostgreSQL server so that we can deploy HA, PITR backup, monitor system, etc.. 

During the weekend I managed to set up GitLab in a test environment to prepare for the migration. Since current GitLab maintainer was a new fellow and was not familiar with GitLab yet, I wrote all the steps including GitLab parts to migrate PostgreSQL. After some migration tests, we did it on production and it was successful.

During the process, I noticed that GitLab supports only PostgreSQL now!

Then I searched through the document, blogs and  GitLab commit comments. Here I would like to share them here so that people can understand deeply why GitLab made the decision. 

Most contents below are just copied and then reorganized, the references are attached at the end of this article.

# [GitLab supports only PostgreSQL](https://docs.gitlab.com/omnibus/settings/database.html#using-a-mysql-database-management-server-enterprise-edition-only)

[PostgreSQL is the only supported database](https://docs.gitlab.com/ee/install/requirements.html#database), which is bundled with the Omnibus GitLab package. You can also use an external PostgreSQL database. 

Support for MySQL was removed in GitLab 12.1(released on July 22nd, 2019). 

We **highly** [recommend the use of PostgreSQL](https://gitlab.com/gitlab-org/gitlab-foss/-/blob/09f1530f84e2c738199967c5553cf581d8203dfc/doc/install/requirements.md#database) instead of MySQL/MariaDB as not all features of GitLab may work with MySQL/MariaDB. For example, MySQL does not have the right features to support nested groups in an efficient manner;.

## Deprecating support for MySQL in July 2017

In July of 2017 [GitLab documented that we would be deprecating support for MySQL](https://gitlab.com/gitlab-org/omnibus-gitlab/merge_requests/1756),

> Using MySQL with the Omnibus GitLab package is considered deprecated. Although
>
> GitLab Enterprise Edition will still work when MySQL is used, there will be some
>
> limitations as outlined in the [database requirements document].

## [Removal of MySQL in July 2019](https://about.gitlab.com/releases/2019/07/22/gitlab-12-1-released/)

In GitLab 12.1, mysql-client is removed from the GitLab packages, in conjunction with the removal of support for MySQL.

Planned removal date: **July 22nd, 2019.**

# [Why we're ending support for MySQL](https://about.gitlab.com/blog/2019/06/27/removing-mysql-support/)

It wasn't an easy decision, but we wanted to share with you why we did it.

## It wasn't great for our use case

There are lots of great use cases for MySQL, our specific needs just didn't seem to be a good fit. [Our use of MySQL had a number of limitations](https://gitlab.com/gitlab-org/gitlab-ce/issues/51173#issues). 

### We can't [support nested groups with MySQL in a performant way](https://gitlab.com/gitlab-org/gitlab-ce/issues/30472#note_27747600)

While for PostgreSQL we have a solution, for MySQL things will be very difficult. MySQL doesn't support `WITH` until version 8, stored procedures can only return 1 row (requiring the use of temporary tables, which are annoying and slow to use) as far as I can tell, stored procedures by default can not recurse (you need to set either a global or session local stack size), and simply slapping 20 `LEFT JOIN` statements in the query probably won't scale (and make the output very hard to work with).

This means we have a few options for MySQL, **all equally messy**:

1. We get all the namespaces recursively in Ruby. This will be slow, load the IDs into memory, and in general will suck.
2. For MySQL we keep using the current routes approach, which will be slow. This may also suggest developers it's OK to use the code else where, and I'd rather remove this as soon as possible instead.
3. **We ditch support for MySQL, something that has a 99.99% chance of being rejected from taking place.**

Without nested groups this would not be an issue, but alas it was introduced before the above problems were considered.

1. Using these raw queries means they'll be called tens of thousands of times per minute (e.g. we have around 30k Rails requests per minute). Creating a temporary table every time is something MySQL probably won't enjoy.

### We have to use hacks to increase limits on columns and this can lead to [MySQL refusing to store data](https://gitlab.com/gitlab-org/gitlab-ce/issues/49583)

#### 500 error when I create a pull request in which  some big text files are committed.

##### Summary

When I create a pull request in which I commit some very large text files, the gitlab returns 500 error.

##### Steps to reproduce

1. I have two branches, `A001` and `master`.
2. In branch `A001`, we have made many changes, including a very large ( > 200kb ) `.json` file.
3. Commit a pull request from `A001` to `master`.
4. gitlab return 500.

##### Workaround

The limit of type `TEXT` is 65535 byte. see [mysql storage requirement](https://dev.mysql.com/doc/refman/8.0/en/storage-requirements.html). `LONGTEXT` is better for this situation.

```
> use gitlabhq_production;
> alter table `merge_request_diff_files` modify column `diff` LONGTEXT;
```

### MySQL [can't add](https://gitlab.com/gitlab-org/gitlab-ce/issues/40168) `TEXT` type column without `length` specified

#### Uploads with long filenames on MySQL fail

##### Summary

Uploads with long filenames on MySQL fail.

##### Steps to reproduce

On a GitLab instance >=  v9.0, using MySQL, upload a file with a 255 character long filename to a Markdown field (issue, MR, comment).

An insert to the uploads table fails because `uploads.path` is `varchar(255)` for Rails `string` fields by default, and the path is the filename plus some directories.

I think this also affects  appearance logo and header logo uploads but not avatars since avatar  filenames get renamed to 'avatar' by the frontend.

##### PostgreSQL

Changing the type from `string` to `text` IIRC doesn't really take long on PostgreSQL since under the hoods both use the same disk representation. This means that PostgreSQL just needs to drop the length constraint.

##### MySQL

One problem is that we index `uploads.path` and MySQL doesn't allow indexes on TEXT without a length, this means we'd have to keep using `varchar` and just increase the limit.

Closing this since the `uploads.path` column length was increased for MySQL in 15270 (merged).

Note that some MySQL installations can run into errors saying the column length is too long. This can generally be resolved by modifying the MySQL setup, but in particular this is a problem at the moment for our Pivotal tile.

And reducing the column size is one of the options on the table.

### MySQL doesn't [support partial indexes](https://dba.stackexchange.com/questions/106589/to-have-postgresql-like-partial-index-in-mysql-5-5)

Neither MySQL nor the siblings (MariaDB, Drizzle, etc) have implemented partial indexes.

What you can do, with this restriction in mind:

- a) make a simple (not partial) index on `(is_active, measurement_id)`. It will be used in queries where the partial index would. Of course if the `is_active` column is 3% True and 97% false, this index will be much bigger (than a partial index). But still smaller than the table and useful for these  queries.
   Another limitation is the index cannot be `UNIQUE` with this solution so the constraint is not enforced. If the index is created with `UNIQUE`, the uniqueness will be enforced for rows with `is_active = FALSE` as well. I assume you don't want that:

  ```sql
  CREATE INDEX dir_events
      ON events (is_active, measurement_id)
      USING btree ;
  ```

- b1) (the simple variation of b):  add another table in your design, with only the primary key columns of `events` and a foreign key to `events`. This table should only have rows where the `is_active` is true in the original table (this will be enforced by your application/procedures). Queries with `is_active = TRUE` would be changed to join to that table (instead of the `WHERE` condition.)
   The `UNIQUE` is not enforced either with this solution but  the queries would only do a simple join (to a much smaller index) and  should be quite efficient:

  ```sql
  CREATE TABLE events_active
  ( event_id INT NOT NULL,         -- assuming an INT primary key on events
    PRIMARY KEY (event_id),
    FOREIGN KEY (event_id)
      REFERENCES events (event_id)
  ) ;
  
  INSERT INTO events_active 
    (event_id)
  SELECT event_id
  FROM events
  WHERE is_active = TRUE ;
  ```

- b2) a more complex solution: add another table in your design, with only the primary key columns of the table **and `measurement_id`**. As in previous suggestion, this table should only have rows where the `is_active` is true in the original table (this will be enforced by your  application/procedures, too). Then use this table only instead for  queries that have `WHERE is_active = TRUE` and need only the `measurement_id` column. If more columns are needed from `events`, you'll have to `join`, as before.
   The `UNIQUE` constraint can be enforced with this solution. The duplication of `measurement_id` column can also be secured to be consistent (with an extra unique constraint on `events` and a composite foreign key):

  ```sql
  ALTER TABLE events
    ADD UNIQUE (event_id, measurement_id) ;
  
  CREATE TABLE events_active
  ( event_id INT NOT NULL,
    measurement_id INT NOT NULL.
    PRIMARY KEY (event_id, measurement_id),
    UNIQUE (measurement_id),
    FOREIGN KEY (event_id, measurement_id)
      REFERENCES events (event_id, measurement_id)
  ) ;
  
  INSERT INTO events_active 
    (event_id, measurement_id)
  SELECT event_id, measurement_id
  FROM events
  WHERE is_active = TRUE ;
  ```

- c) **maybe the simplest of all: use PostgreSQL.** I'm sure there are  packages for your Linux distribution. They may not be the latest version of **Postgres but partial indexes were added in 7.0 (or earlier?)** so you  shouldn't have a problem. Plus, I'm confident that you could install the latest version in almost any Linux distribution - even with a little  hassle. You only need to install it once. 

### bugs with case [(in)sensitivity](https://dev.mysql.com/doc/refman/5.7/en/case-sensitivity.html).

Bugs relating to case sensitivity:

- https://bugs.mysql.com/bug.php?id=65830
- https://bugs.mysql.com/bug.php?id=50909
- https://bugs.mysql.com/bug.php?id=65830
- https://bugs.mysql.com/bug.php?id=63164

Actual result are not case sensitive:

```
select * from collationtest where text collate utf8_general_cs like 'test';
+------+
| text |
+------+
| test |
| Test |
| TEST |
+------+
3 rows in set (0.00 sec)
```

Expected results should be case sensitive, like the utf8_bin collation:

```
select * from collationtest where text collate utf8_bin like 'test';
+------+
| text |
+------+
| test |
+------+
1 row in set (0.00 sec)
```

A accent and case sensitive collation for ut8mb4 will be made available in 8.0.

### Places where MySQL already is not supported

#### Geo

GitLab Geo also does not support MySQL. Existing users using GitLab with MySQL/MariaDB are advised to migrate to PostgreSQL instead.

#### Disaster Recover

We already don't support MySQL for Geo, which means no Disaster Recovery solution available when using MySQL.

#### Helm Charts

#### [Zero downtime migrations]() do not work with MySQL

**You use MySQL as the database for GitLab. Any upgrade to a new major/minor release will require downtime.** If a release includes any background migrations this could potentially lead to hours of downtime, depending on the size of your database. **To work around this you will have to use PostgreSQL** and meet the other online upgrade requirements mentioned above.

#### [Database load balancing](https://docs.gitlab.com/ee/administration/database_load_balancing.html) is supported only for PostgreSQL.

#### GitLab [optimizes the loading of dashboard events](https://gitlab.com/gitlab-org/gitlab-ce/issues/31806) using [PostgreSQL LATERAL JOINs](https://blog.heapanalytics.com/postgresqls-powerful-new-join-type-lateral/).

## It made us slower

In order to work around some of the pain points above, we ended of creating a lot of [MySQL](https://gitlab.com/gitlab-org/gitlab-ee/blob/7ef7c604729c2627914bcc0ece3355786a9a3413/ee/db/migrate/20180831134049_allow_many_prometheus_alerts.rb#L30)-[specific](https://gitlab.com/gitlab-org/gitlab-ee/blob/7ef7c604729c2627914bcc0ece3355786a9a3413/db/migrate/prometheus_metrics_limits_to_mysql.rb) code. In some cases this led to [merge requests that were twice as complex](https://gitlab.com/gitlab-org/gitlab-ce/merge_requests/16793) because they had to support a second database backend. Creating and  maintaining this code is a drain on our cycle time and velocity, and it  puts a dampener on our value of iteration.

It also made us slower because our CI system would run our test suites twice, once on each backend. Removing support for MySQL reduces the time for our CI jobs, and reduces the costs. [These costs ended up being considerable](https://about.gitlab.com/company/team/structure/working-groups/gitlab-com-cost/), and it was difficult to justify the expense with the small number of users choosing MySQL.

###  subqueries that work well in PostgreSQL may not be [performant in MySQL](https://dev.mysql.com/doc/refman/5.7/en/optimizing-subqueries.html)

In general, SQL optimized for PostgreSQL may run much slower in MySQL due to differences in query planners. For example, subqueries that work well in PostgreSQL may not be [performant in MySQL](https://dev.mysql.com/doc/refman/5.7/en/optimizing-subqueries.html).

### Binary column index length is limited to 20 bytes. This is accomplished with a hack.

### [The milestone filter runs slower queries on MySQL](https://gitlab.com/gitlab-org/gitlab-ce/issues/51173#note_99391731).

### [We cannot leverage PostgreSQL partitioning easily because we cannot use triggers to stay compatible with MySQL.](https://gitlab.com/gitlab-org/gitlab-foss/-/issues/51173#note_103352064)

With the growing amount of data, **PostgreSQL partitioning** is a great tool to partition tables on a single server to keep queries fast. This has been available in PostgreSQL for many releases but needs some code to manage partitions effectively. This is usually implemented with triggers, which we do not support today. I think this is due to maintaining a `db/schema.rb` instead of providing plain SQL schema because of the MySQL support.

### Cycle analytics uses an almost completely different implementation for MySQL. On PostgreSQL we use CTEs, but on MySQL we use temporary tables (which will not scale at all).

## We couldn't take advantage of either backend

By providing support for both database backends (PostgreSQL and MySQL) we were unable to truly take advantage of either. Where we wanted to utilize specific performance and realiability capabilities unique to a backend, we had to instead choose the lowest common denominator. As an example (there are [more](https://gitlab.com/gitlab-com/www-gitlab-com/merge_requests/24987#note_186506464)), we wanted to use PostgreSQL's `LATERAL JOIN` for optimizing dashboards events, but [couldn't because it was not available in MySQL](https://gitlab.com/gitlab-org/gitlab-ce/issues/31806#note_32117488).

## Most of our customers are on PostgreSQL

Using [Usage Ping](https://docs.gitlab.com/ee/user/admin_area/settings/usage_statistics.html#usage-ping) data we got a clear sense that the vast majority of our customers had  already moved to PostgreSQL. We've seen a steady migration to PostgreSQL and recently had less than  1200 GitLab instances reporting their usage of MySQL. At the time there were 110,000 instances using PostgreSQL. Of  those still using MySQL the majority were using 11.0 or earlier.

**Less than 1% of all instances use MySQL. Only about 0.25% of Starter/Premium/Ultimate customers use MySQL.**

This gave us a lot of confidence that GitLab users still on MySQL could migrate to the bundled PostgreSQL backend if they choose to upgrade to 12.1 and beyond, or remain on MySQL in older versions of GitLab.

As a side note – we don't use MySQL ourselves, and not doing so meant we weren't encountering issues BEFORE our users.

# Useful links

- [Migrating from MySQL to PostgreSQL](https://docs.gitlab.com/ee/update/mysql_to_postgresql.html)
- [pgdiff](https://github.com/joncrlsn/pgdiff)

# Conclusion

For GitLab, PostgreSQL is proven to be the right choice. Yet MySQL is still used more widely. You can suggested to continue to use MySQL as long as it meets your requirements and you are not suffering from it as GitLab.