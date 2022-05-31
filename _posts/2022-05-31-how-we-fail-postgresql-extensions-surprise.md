---
title: "How we fail #1 - PostgreSQL extensions surprise"
date: 2022-05-31T13:35:53+02:00
categories:
  - how-we-fail
tags:
  - fail
  - postgresql
  - uuid
toc: true
---

As everyone - we **fail** - a lot, but the failure itself is not the issue, the important part is to be able to learn from our failures and mistakes.

Yes making a fatal mistake on **production** is serious and can affect our clients - luckily we have been able to catch all critical issues in our testing environments so no damage has been done 😮‍💨

> Our production environment is thoroughly backed-up in two separate places and we have been testing recovery periodically.

But as I said - mistakes happen and today I like to talk about the *small* mistake which happened during our internal alpha release which pretty much broke the whole environment for a few hours before we found out what was the roof of the cause 😨
<!--more-->

# What happened?
One of our primary databases is **PostgreSQL** with a few notable extensions like `unaccent`, `pgcrypto`, and most importantly the `uuid-ossp` extension for generating UUID especially the V4.

And another tool that I need to mention is a cool and useful system the [SkyWalking](https://skywalking.apache.org) - which is pretty much a "do everything" **APM** system with features like - centralized logging (including client-side), topology, tracing, alarms, and last but not least - performance monitoring.

## Upgrading the SkyWalking
We have used an older version of SkyWalking for some time as we encountered some issues previously with the latest version in our container cluster.

But generally, we like to be "on the edge" and use the latest and greatest version of tools, packages, and others. So we did the upgrade to the latest **9.0 version** and we were able to overcome issues that stopped us in our previous attempt.

So far so good, right? Well - the first issue is - we currently use SkyWalking with **PostgreSQL** and this works quite well it still has some limitations like SkyWalking will use the `public` schema by default and in the older version, it was impossible to force different one.

> It is **not** best practice to put huge systems like SkyWalking into a `public` schema as it will get messy very soon.

And to make a proper upgrade we had to clean and re-create (we were okay to lose historical data) all tables used by SkyWalking in the `public` schema - in our case, the SkyWalking is the only tool that resides inside that schema. So *natural* choice was to drop the schema itself and create a new one 😬

# Now it gets messy
I - personally - did the schema drop using the following simple SQL query.

```sql
DROP SCHEMA public CASCADE;
CREATE SCHEMA public;
GRANT CREATE, USAGE ON SCHEMA public TO postgres;
GRANT CREATE, USAGE ON SCHEMA public TO public;
GRANT CREATE, USAGE ON SCHEMA public TO skywalking;
```

And everything started from this point.

![So it begins](https://i.pinimg.com/originals/af/68/d2/af68d2d9baebbdf26c2cea10e087c048.gif)

## Extensions are lost
Dropping the `public` schema also drops the every created extension inside it. And as we use `public` schema to store all global extensions for all services using `search_path = "service-ID", "public"` that means ALL services now lost access to most important functions like `uuid_generate_v4()` to generate UUIDs for our items in the database.

> We use UUID v4 (GUID if you want to) as the primary globally unique key for records that needs ID. It still has some cons and prons - but leave this for another post 🙂

Well, panic intensified - we needed to create our extensions again, which again is a pretty simple SQL query for example like this.

```sql
CREATE EXTENSION IF NOT EXISTS "unaccent" SCHEMA public;
CREATE EXTENSION IF NOT EXISTS "pgcrypto" SCHEMA public;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp" SCHEMA public;
```

And yet - everything was in flames still.

![This is fine](https://media.tenor.co/images/0d1329f5ff7d31712e3d12ce160df6ec/raw)

## What next?
Another issue that arose was - errors about setting `NULL` into `NOT NULL` columns. This was weird as that particular column used the `uuid_generate_v4()` function and it is impossible for that function to return just `NULL`.

First, we thought the services which were running still kept "old reference" to the `public` schema. Because the `public` destruction happened during API services lifetime and only SkyWalking was down at that moment.

So we have tried the all-time best solution - did you try to turn it off and on again? Yet still - nothing 🥲

Now we have started comparing table structures from other environments with this one and we found something really weird. We have found out the `default expression` is missing for some columns (and those columns can't be `NULL`), not only to just "some" to all columns which used `uuid_generate_v4()` function as the default value.

# Gotcha!
Finally, we had the root cause of this weird issue - dropping extensions removed all default values which used functions from those extensions.

Okay now we have our offender guilty as charged but how to recover from this state?

> As I mentioned before this issue appeared in the testing environment and we do not back up these environments as they are only for internal bleeding-edge testing purposes.

Luckily PostgreSQL has a default view `information_schema` that can be used to introspect tables and their structures - so with some SQL query, we can introspect tables in a different environment and get all columns/tables which has the default value set correctly

For example, to get all tables that use generated UUIDv4 as the default value with "full path" format and their column name, you can use the following SQL query.

```sql
SELECT
  table_catalog | | '.' | | table_schema | | '."' | | table_name | | '"' AS "Path",
  column_name AS "Column"
FROM
  information_schema.columns
WHERE
  data_type = 'uuid'
  AND column_default = 'uuid_generate_v4()';
```

Now we have "full path" to the table and its column name which use the generated UUID. Now simply we can save the result as CSV and write a very simple CSV parse to generate `ALTER` queries to set a default value, like this.

```sql
ALTER TABLE root.service_schema."Table" ALTER COLUMN "Id" SET DEFAULT uuid_generate_v4();
```

And **voilà**! Everything was back on track - API queries to PostgreSQL did not fail and our automatic tests were *green* again as they should be.

# Takeaways
I started this post with - "be able to learn from our failures" - and so we did. Important things we will have to keep in mind in the future are:
 - Do not drop the whole `public` schema if not necessary
 - Move SkyWalking tables to the different schema (this will be solved differently - we will move SkyWalking to separated DB - most likely OpenSearch)
 - Try to not create "global" extensions and move them into per-service schemas if possible  


_This post is part of the "How we fail" series, hope you liked it, and stay tuned for more!_
