---
title: "Mixing it up: Using several different databases for our platform"
date: 2023-07-17T12:00:00+02:00
categories:
  - backend
tags:
  - backend
  - database
  - postgresql
  - mongodb
  - opensearch
  - cassandra
  - memcached
  - cache
toc: true
---

…right now around 6 databases and we are not stopping there 😄

# The Big Bang

Let’s start from the beginning as every system - our platform needs some database to store its data, *obviously,* because I’m not sure if anyone would use and subscribe to a system where any data are stored in-memory and lost on every service restart/deploy 😄

And when we initially started writing our platform code there was several discussion about which database to use at first, because this very often shapes initial data design and also “roots” to codebase in a way that changing it would be very expensive in the future (yet not impossible).

We knew we will not most likely settle with only one - which is not generally bad idea for many projects, but our plans and use-cases would benefit from using several databases with different data access/storage strategies. I’m, personally, huge advocate of this idea - using several databases with **specific strengths to solve specific use-cases**, but starting with many different databases at first would **complicated things heavily** and we needed to start writing code fast!

So initial requirements were simple - battle-tested, should be RDBMS-based for simpler data design at first and no need to learn anything new for most people, with performance in mind - because the ultimate question or idea was - “If this database will be our first and last - will it sustain everything we plan?”.

The candidates were - **MySQL** (or any fork like **MariaDB** or **Percona**), **Microsoft SQL Server** (as some team members had good experience with it) or **PostgreSQL**.

**MySQL** itself was discarded due its complicated legal stuff with Oracle in the past and its forks - especially the **MariaDB** due its new storage engines and performance - were considered as a serious candidate. **Microsoft SQL Server** was discarded as we wanted to build whole platform on open-source technologies and in the future open-source our libraries/modifications as well - **Microsoft SQL Server** licensing would complicate this idea and make any possible modifications kinda impossible.

So the winner (winner, chicken dinner) was **PostgreSQL** simply due its flexibility (for example - native UUID support, storing and querying JSON structures, and many other extensions), performance and overal reliability as it is widely battle-tested in huge projects.

And even now **PostgreSQL** (version 15 currently) is doing the heavy lifting in our platform and keeps most of our mission critical data and it never let us down (not counting situations when *we* actually break things 😅).

## Cache Line Departure

Even though cache system is not often considered as a proper database - our need-for-speed and giving more breathing space to PostgreSQL - we knew that some caching system is necessary.

The system must be simple, language agnostic, distributed (to be used across several service instances etc) - and of course very fast.

One of our first ideas were to go with **Redis** which is full fledged database and could be used as cache at first and its usage could be expanded down the line in the future. But we could not justify using it as we really needed only fast key-value store without any fancy features - just gimme value for this key and do not ask any questions… and of course - do it fast!

The choice was **Memcached** - fast in-memory distributed key-value storage - love at first sight, right? Right!

Memcached is key backbone for our platform and many requests and operations are using it to speed up data access and limit frequent (not only) PostgreSQL queries for data which rarely changes - for example every service has ACL roles and its permissions loaded and stored inside Memcached under some specific key. And for example - when someone edits role permissions using web UI - the ACL service will dispatch MQ broadcast with new permission-set and every service will have latest data. Or if cache just expires the service will ask ACL service for latest permission-set when needed and writes it to the cache. 

# Strong Foundation, Where To Next?

Now we have strong and reliable RDBMS backing our system and cache system - now we have technological opportunities and business reasons to start thinking about adding a new one.

What are the requirements? Initial business driver was the new CRM module (even though the codename is `Party` - this service wasn't created at party 🙁 😄 ) which allows pretty wild data structures behind the scene - and yes it could be fitted to PostgreSQL if we really wanted to, but no-SQL database without schema felt as a better solution for this.

So the requirements were simple - PostgreSQL but without schema. During our decision process two systems were considered - **CouchDB** and **MongoDB**.

Both has own strengths but **MongoDB** won this fight especially because we liked the JSON-based query language, transactions support and strong consistency and replication support.

Currently not only `Party` service but few more services are using **MongoDB** to store its data reliably. As this is no-SQL schema-less database there are no “migrations” in a classic sense - and we wanted to use this fact, because we have anticipated the old data will be very rarely requested, so there is no need to **pay the performance and risk** **price** for whole table alternation and all data migration - so we implemented versioning on entity-level and when old entity is requested - it is updated to latest format, written-back and served to upper layers after that. We also support whole collection migration if its critical and really necessary but we are trying not use this much.

And yes - this system caused many headaches 😄 As the implementation was tricky and caused some data inconsistencies during initial development cycles. But that’s not MongoDB fault, it is ours and MongoDB itself is good partner for our platform.

# Logs and Audit Events

Every platform must produce **logs** - especially for having more information on “non-production” nodes during QA phases and also logs for fatal issues on production nodes which are rare but if they occurs we can not be blind and need some information in order to replicate the issue on “non-production” nodes where more logging levels are allowed - and also store **audit** events - for example when user changes some critical setting inside the system for your tenant, you really want to know “*who*”, “*when*” and “*what*” changed.

All these information could be easily stored in already mentioned databases but we wanted better ability to search data and also use the database not only for logs or audit events in the future.

In this case there wasn’t much discussion as we agreed to go with **OpenSearch** (fully open-source fork of ElasticSearch before licensing changes) for its known amazing search capabilities, strong support for real-time data analysis and more.

Even though right know we do not utilize the **OpenSearch** in its full strengths - there are plans to put more data especially those which will benefit from full-text search capabilities.

# Data Warehouse

Data warehouse is **huge** topic, so this will be really just a small sneak peak what is going on behind it.

Data warehouse its one of our latest, yet not fully released for public, service to support big data (with real-time support) storage with processing, real-time analysis/alerts/monitoring and visualization.

The warehouse system consist from more than one service and also more than one database. Some of them are re-used from those mentioned before, so let’s focus more on those which are currently the new ones.

For sake of this article lets simplify the warehouse to two layers - **raw data storage** (could be referred as *data lake*) and **processed data storage**.

Because the raw data storage will processes many writes per second of raw unprocessed data - we have to store it fast and move to another. For this raw data storage we have chosen the **CassandraDB**. As its replication, data partitioning and generally amazing write capabilities are key features for us in this use case, it was great and amazing solution.

When data are processed by scriptable pipelines they could end inside different databases based on configuration and other. But our business needs were mostly for monitoring and working/visualizing time-series data we have added another database - the **InfluxDB**. Again its core strengths were something which greatly fits to our needs for processed data.

And as I mentioned the warehouse system is complex and there are more exciting stuff which we will share with you in future articles.

# Conclusion

As you can see - we use **a lot different database systems**, we are strive to use their strengths for our data needs were its appropriate (it is no shame to put data into good old PostgreSQL!). And most likely we are not stopping here - right now we have several discussions and plans to add more specific databases to support new features of our platform.

And also this article was meant as a general overview what and why we use it - we will do some specific use-cases and deep dives in the future - of course if you are interested!

See you around another table (lame, I know) at another time, cheers!
