# MySQL or PostgreSQL?

# Steinar H. Gunderson Leaving MySQL

About one month ago, [Steinar H. Gunderson](https://www.sesse.net/) wrote a [blog](https://blog.sesse.net/blog/tech/2021-12-05-16-41_leaving_mysql) **rubbishing MySQL and recommending PostgreSQL**. 

Here is the original content, followed by a wonderful discussion reorganized from more than 300 comments.

Note: Following blog was visible 24 hours ago, but not now, only available for a month?

## Sun, 05 Dec 2021 - Leaving MySQL

Today was my last day at Oracle, and thus also in the MySQL team.

When a decision comes to switch workplaces, there's always the question of “why”, but that question always has multiple answers, and perhaps the simplest one is that I found another opportunity, and and as a whole, it was obvious it was time to move on when that arrived.

But it doesn't really explain why I did go looking for that somewhere else in the first place. The reasons for that are again complex, and it's not possible to reduce to a single thing. But nevertheless, let me point out something that **I've been saying both internally and externally for the last five years** (although never on a stage—which explains why I've been staying away from stages talking about MySQL): **MySQL is a pretty poor database, and you should strongly consider using Postgres instead.**(**1**)

Coming to MySQL was like stepping into a parallel universe, where there were lots of people genuinely believing that MySQL was a state-of-the-art product. At the same time, I was attending orientation and told how the optimizer worked internally, and I genuinely needed shock pauses to take in how primitive nearly everything was. It felt bizarre, but I guess you soon get used to it. In a sense, it didn't bother me that much; lots of bad code means there's plenty of room for opportunity for improvement, and management was strongly supportive of large refactors. More jarring were the people who insisted everything was OK (it seems most MySQL users and developers don't really use other databases); even obviously crazy things like the executor, where everything was one big lump and everything interacted with everything else(**2**), was hailed as “efficient” (it wasn't).

Don't get me wrong; I am genuinely proud of the work I have been doing, and **MySQL 8.0 (with its ever-increasing minor version number) is a much better product than 5.7 was**—and it will continue to improve. **But there is only so much you can do**; the changes others and I have been doing take the MySQL optimizer towards a fairly standard early-2000s design with some nice tweaks, but that's also where it ends. (Someone called it “catching up, one decade at a time”, and I'm not sure if it was meant positively or negatively, but I thought a bit of it as a badge of honor.) In the end, there's just not enough resources that I could see it turn into a competitive product, no matter how internal company communications tried to spin that Oracle is filled with geniuses and WE ARE WINNING IN THE CLOUD. And that's probably fine (and again, not really why I quit); **if you're using MySQL and it works for you, sure, go ahead**. But perhaps consider taking a look at the other side of that fence at some point, past the “**OMG vacuum**” memes.

## Blog Here

My new role will be in the Google Chrome team. It was probably about time; my T-shirt collection was getting a bit worn.

(**1**) Don't believe for a second that MariaDB is any better. Monty and his merry men left because they were unhappy about the new governance, not because they suddenly woke up one day and realized what a royal mess they had created in the code.

(**2**) For instance, the sorter literally had to care whether its input came from a table scan or a range scan, because there was no modularity. Anything that wasn't either of those two, including joins, required great contortions. Full outer joins were simply impossible to execute in the given design without rewriting the query (MySQL still doesn't support them, but at least now it's not hampered by the old we-can-do-left-deep-plans-only design). And don't even get me started on the “slice” system, which is perhaps the single craziest design I've ever seen in any real-world software.

# [The Discussion](https://news.ycombinator.com/item?id=29455852)

## leetrout

I only have anecdata of my usages of MySQL and Postgres but **I swear people that cut their teeth on MySQL and have never used Postgres just don't know what they are missing**. Yes Postgres can be slower out of the box and yes **Postgres has worse connection handling** that usually requires a pooler **but the actual engine and it's performance makes it worth it** in my opinion.

## ploxiln

The author was rather focused on the optimizer/scheduler. I can easily believe that it is much less advanced and robust, and messier, in MySQL. I really liked MySQL (and more recently MariaDB), and I have experienced some significant annoyances with Postgres recently, and it's for a completely different set of reasons. Logical Replication is the big one. MySQL has had logical (and "mixed") replication for a decade (or more?). **PostgreSQL** has gotten built-in **logical replication** only recently, and it's still very annoying. It **doesn't replicate schema change statements**, so those have to be applied independently by some external system, and of course it's not going to be timed exactly right so replication will get stuck. And somehow for very lightly active applications generating just a few MB of writes per day, the stuck replication log can back up 10s of GBs per day. And **does logical replication across versions for zero-downtime upgrades work in Postgres yet? I dunno.** **Connections** you mentioned already, **Postgres needs an external connection pooler/proxy**, MySQL not nearly so soon. **Vacuuming is a problem** I never had with MySQL/MariaDB of course. I'm an infrastructure dev, not an DBA or an advanced SQL user, I never used window functions, or wrote a query that filled half the screen, or have a thick web of foreign keys ... If you keep things damn simple so you have confidence in efficiency and performance, then MySQL works pretty damn well in my experience, and Postgres, for all its academic correctness and advanced query feature set, has been pretty damn annoying. And, you can tell MySQL to use a particular index for a query!

## ufmace

This is about where I'm at. I've used both DBs and adminned sites with them. **Sure PG has a ton of nice SQL features that are really handy if you want to do any kind of more advanced analysis.** I wouldn't be surprised either to learn that it's much more efficient on executing complex queries. But the bottom line is that **the great majority of production usage is basic CRUD queries using simple SQL, and a lot of applications will never need to use advanced SQL.** **The things MySQL does do better**, replication and connections like you mentioned, tend to be more suited to keeping big dumb basic sites up and running reliably. I'm usually gonna reach for Wordpress first along with MySQL if I need to set up a basic blog for non-technical people to contribute to. Maybe the DB isn't as cool as it could be, but it works fine, and I hardly ever have need to touch the database itself directly.

## mst

I've seen quite a few codebases where **the complexity of the problem did but they basically bodged around it in the app tier in order to keep the SQL within what MySQL could handle** - and it's -very- easy to not even notice you're doing that because it's a technical debt/death by a thousand cuts sort of experience when it does happen. Then again, depending on your scenario, introducing a bunch of extra complexity into your app tier code may be less annoying than introducing a bunch of extra complexity (connection poolers etc.) into your infrastructure. So it ends up that there's a bunch of workloads where mysql is obviously the better option, and a bunch of workloads where postgresql is obviously the better option, and also a bunch where they're mostly differently annoying and **the correct choice depends on a combination of your team composition and where your workload is likely to go in the next N years**.

**Having used both extensively**, I'd say that there's a lot of knowledge when using mysql that's basically "**introducing complexity into your application and query pattern to work around the optimizer being terrible** at doing the sensible thing" and then **there's lots of knowledge when administrating postgresql** that's "having to pick and choose lots of details about your topology and configuration because you get a giant bag of parts rather than something that's built to handle the common cases out of the box".So ... on the one hand, **I'm sufficiently spoiled by being able to use multi column keys, complex joins, sub queries, group by etc. and rarely have to even think about it because the query will perform just fine** that **I vastly prefer postgres by default as an apps developer these days**. On the other hand, I do very much remember the learning curve in terms of getting a deployment I was happy with and if I hadn't had a project that flat out required those things to be performant I might never have gotten around to trying it. So, **in summary: They've chosen very different trade-offs, and which set of downsides is more annoying is very dependent on the situation at hand.**

## da_chicken

This completely maps to my experience as well. I still believe Postgres' defaults being set to use an absolute minimum footprint, especially prior to PostgreSQL 9, significantly impacted it's adoption. It's better now, although IMO it could still do with an easier configuration where you could flag the install as dev/shared or prod/standalone to provide more sensible settings for the most common situations. Like, gosh, I'd like to be able to stand up an instance and easily tell it to use 90% of the system memory instead of 100 MB or whatever it was.

**But the MySQL issues were much worse**. **Outer join and sub query problems, the lack of functions like ROW_NUMBER(), a a default storage engine that lacked transactions and fsync(), foreign keys that don't do anything**, etc. **I've met so many people who think MySQL limitations are RDBMS limitations, or don't understand database fundamentals because MySQL just didn't implement them properly.** Then again, I also remember when the best reason to use MySQL was the existence of GROUP_CONCAT(), which everything else lacked for religious reasons. I also vividly remember prior to MySQL 5.5 when new users of PostgreSQL, Oracle, or MS SQL Server would discover that, instead of making a guess for what you wanted, the RDBMS would actually return error messages and expect you to understand and solve them deterministically. And somehow they would be angry about it! Old MySQL (mostly 3.3, but 4.0 and 4.1, too) used to silently truncate data, or allow things like February 31 in a date field, or **silently allow non-deterministic GROUP BY**, or only reporting warnings when you explicitly asked for them really undermined the perception of MySQL to database folks. This wasn't that long ago, either.

## brightball

**A lot of applications don’t use more advanced SQL simply because it’s not available**. Just as an example, there are so many people who assume they need a 3rd party search engine simply because what’s built into their database is so bad compared to the options you get with PG. You’re welcome to only use your database as a crud data store while you manage a mountain of other tools to work around all of the limitations…but you don’t have to.

## rtpg

We had tried to do a logical replication-based strategy for Postgres to achieve no-downtime version upgrades during business hours, but there were so many gotchas and things you have to do on the side (stuff like sequence tracking etc), that we just gave up and took the downtime to do it more safely.

I think **Postgres** is really cool and has a lot of good reliability, but I've always wondered what people do for high availability. And yeah, **pgbouncer being a thing bugs.**

It feels like we're getting closer and closer to "vanilla pg works perfectly for the common workflows" though, **so tough to complain too much**.

So to be honest I see that article (linked below) and I think "Yeah that's probably all right", but given this is for a Real System With Serious People's Data, when faced with "take the downtime and do the offline data upgrade that _basically everyone does_" and "try to use logical replication to get a new database in the right state, knowing that **logical replication doesn't copy everything** so I need to manually copy some stuff, and if I miss stuff I'm in a bad situation"... I trust the pg people that tools do what are advertised, but it's a very high risk proposition.

## skunkworker

IMO **logical replication is too slow for Production**. At scale you'll be doing physical replication and rarely using logical replication as it is quite slow in comparison still (As of PG 13.2). **Logical replication is helpful when doing a Postgres version upgrade primary/replica switch, but not useful many other places at the moment.**

You do need to use an external connection pooler (pgbouncer etc), though Postgres 14 has made some large improvements in gracefully handling large numbers of connections, though you can still run into problems.

As for the query planner/optimizer, so far in all of the optimizations and improvements I have worked on, I've only run into 1 or 2 that had a query plan that made my head scratch. There are some extreme edge cases that will prompt the cost function to prioritize the wrong index, but in production I have found that 99%+ of slow queries can be easily improved by a rather simple composite index. **One thing I do love about postgres is using partial indexes, which can significantly reduce the amount of space required and also make it extremely easy to create indexes to match predicates of a function, while indexing other columns.**

Other than one or two slow queries I've tracked down and worked on, I've never wished that I could "hint" to use a particular index over another.

Now some of the things that people like for Postgres over Mysql, in practice aren't that great at the moment. People like doing atomic DDL operations, when in reality locking the table schema can cause lots of problems, and in production you only add indexes etc concurrently.

You still get the issue of dead tuples in mysql and need to periodically clean them up using OPTIMZE TABLE et al, though postgres and innodb have different designs, but ultimately it needs to happen sometime. **Its just that postgres IMO requires more tuning to get the right balance.**

## bigiain

I seriously trialled and compared Postgres vs MySQL at the start of **a major major project**, and **MySQL had a few clear wins for us (mainly replication)** while the features Postgres had weren't in our current roadmap's requirements (the biggest regret that caused me was not having stored procedures). That was in 1998/1999. I now seem to be stuck on a career path where **everything is MySQL and switching to Postgres anywhere seems to have huge back pressure** from everyone I work with - **even though at least half of them know damned well (like I do) that MySQL hasn't been the right choice for a couple of decades almost**.

## cortesoft

Yeah, I am one of those people. Needed a database in 2005, and MySQL was the de facto choice. **Got used to it and never ran into problems** that couldn’t be solved by getting better at schema design and indexing. **I never felt limited by MySQL and I am very comfortable with it**, so never felt the need to try anything else. I might be missing something, but there is an opportunity cost in switching without a real motivating reason.

# Conclusion

For the database itself, PostgreSQL is a better choice. However, **people might need to consider much more things besides technology advantages**.

1. **PostgreSQL was born much earlier, but MySQL is more widely used for now**

2. There are **much more MySQL DBAs available**. 

   Yet **recruiting a decent PostgreSQL DBA** has taken us **five months already** and at least it would take **at least another one month**, i.e. more than **half a year** to get a PostgreSQL DBA! **OMG**!

3. MySQL teams have got quite a long time to do all the automation tools, yet PostgreSQL DBA might still use the old old-fashioned way. For example,to deploy a simple UPDATE statement, a MySQL developer might only need to fill in necessary information on a web UI, and the SQL will be run **automatically**. However, a PostgreSQL DBA might need to do it manually and there is a chance to produce a **major incident** if not careful enough.

If you want to take advantage of PostgreSQL and are good to **give PostgreSQL DBAs some time** to polish automation tools, **you will definitely be rewarded**!