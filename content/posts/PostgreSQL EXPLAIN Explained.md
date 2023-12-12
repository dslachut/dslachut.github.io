---
title: PostgreSQL EXPLAIN Explained
date: 2023-12-12T17:20:00-06:00
draft: false
tags:
  - how-to
categories:
  - PostgreSQL
  - '"query optimization"'
keywords:
  - postgres
  - explain
publishDate: 2023-12-12T17:20:00-06:00
toc: true
---
*A basic overview of using `EXPLAIN` in PostgreSQL.*

Note: This post is based on [this post](https://www.cybertec-postgresql.com/en/how-to-interpret-postgresql-explain-analyze-output/). I'm producing this summary for myself, because [The Internet is not Forever](../the-internet-is-not-forever).

Up Front: EXPLAIN and friends
---

- `EXPLAIN` shows you what the database thinks about your query.
- `ANALYZE` executes the query and shows the actual run time alongside the plan.
    - Be careful doing this with anything that modifies data!
- `BUFFERS` shows cache usage
    - Have to use with `ANALYZE`
- `VERBOSE` helps with slow, expensive functions
- `SETTINGS` shows performance related values
- `WAL` show the Write Ahead Log usage for statements that modify data
- `FORMAT`
    - default TEXT
    - Also: XML, YAML, JSON

Example:
```SQL
EXPLAIN (ANALYZE, BUFFERS) SELECT a,b FROM c WHERE d > 100;
```

Problem: Slow queries
---

When your PostgreSQL database gets large or complex enough, your queries are going to start taking more time than you want to execute. There are some easy things you can do first. Write better queries and add indices.

First, you can look for obvious ways to speed up your query. Minimize joins and operations that cause loops or whole-table scans. Put as filters as possible in your query and---in complex queries---put those filters in as early as possible.

Second, add---or get your DBA to add---an index to columns on which you need to filter. An index takes up disk space and slows down any data changes, so there may be a good reason you can't create a certain index. If there is not a specific problem like this for you, then build the index. Some data types---e.g. PostGIS geometries---are near impossible to query in a reasonable time unless you have an index.

If you with databases long enough, then you'll get to situations where you can't see anything obviously wrong with your query and you should have all the indices you need, but the query is still too slow. This is where you'll need query analysis

EXPLAINing your query
---

SQL is a declarative programming language: you tell the computer what you want and the built-in optimizer compiles a procedure. If your query is too slow, you need to analyze this procedure to figure out if something is going wrong.

### EXPLAIN

`EXPLAIN` Is the basic tool for looking at the procedure, aka the Query Plan, to see where your query is slow. The four basic things I look for when EXPLAINing a query are:
- *Are my indices being used?* It's possible to have an index on a column, but your query is written so that the optimizer doesn't see how to use the index.
- *Is there a sequential scan?* This can be a big hint from the optimizer about where you need to put an index if you don't have one. If you do have an index on the column being scanned, that means your query is selecting everything in the column without filtering. 
- *Is there a loop?* Loops are slow because, even though they may do something fast, they do it a lot. The query plan will point you to what is causing the loop so you can think about how to restructure your query to avoid the loop.
- *What is the highest-cost block?* Often, there will be one specific part of your query that is slowing down everything. If you can't solve it some other way, your best bet might be to restructure your query to avoid doing that part of it.
`EXPLAIN` often doesn't give a complete enough picture to understand the slowdowns, but there are some parameters you can add to it for more information.

### ANALYZE

The first shortcoming of `EXPLAIN` is the `cost` estimate for each block of the query plan. As it says on the tin, it's just an estimate. If you want to get the real runtime of the query, you need the `ANALYZE` parameter.

``` SQL
EXPLAIN ANALYZE SELECT a,b FROM c WHERE d > 100;
```

**There is a big downside** to using `ANALYZE`: to get the real runtime of a query, it must execute the query for real. This can lead to a couple complications:
- If your query modifies data, `ANALYZE`-ing the query will modify your data. It will create tables, delete data, or whatever else you accidentally tell it to do. If you need to `ANALYZE` a data-modifying query, you need a solid test environment to prevent data loss.
- If your query is really slow, which it is if you have to `EXPLAIN` it, `ANALYZE`-ing the query will be really slow.

### BUFFERS

`BUFFERS` tells you how well your query uses PostgreSQL's built-in data caching. Cache hits are good. They mean some piece of data was already loaded into working memory. Cache reads and writes are bad. They indicate disk I/O which is much slower than reading from memory.

If you have a lot of reads and writes compared to hits, the first thing you want to try is executing the `EXPLAIN` with `BUFFERS` again. If the cache hits are still low compared to reads and writes, you should try to restructure your query to better take advantage of caching. On the other hand, the cache hits could suddenly be higher, indicating a cold-start problem that may be difficult to mitigate inside the query.

```SQL
EXPLAIN (ANALYZE, BUFFERS) SELECT a,b FROM c WHERE d > 100;
```

The downside to `BUFFERS` is the same as `ANALYZE`. To get real execution stats for a query, the query must be executed for real. In fact, the downside is the exact same, because `BUFFERS` can only be used with `ANALYZE`.

### WAL

`WAL` is another one that must be used with `ANALYZE`, because it's gathering runtime data. All the caveats of using `ANALYZE` apply, except more so because this parameter is specifically for SQL statements that modify data. WAL stands for "Write Ahead Log".

```SQL
EXPLAIN (ANALYZE, WAL) INSERT INTO c (a,b) VALUES (100, 200);
```

At a previous employer, we had a couple large data sets which we needed to transform inside PostgreSQL. These complex transformations on large datasets would heavily use (or sometimes fill-up, oops!) the Write Ahead Log. Using the `WAL` parameter will show you what parts of the query are heavily using the WAL.

### FORMAT

The `FORMAT` parameter lets you control the output format of your `EXPLAIN`s. The default is a `TEXT` format that's meant to be human-readable. It often is human-readable. There are external tools you can use to better dig into these query plans. For those you can have the query planner output to a machine-readable format: `XML`, `YAML`, or `JSON`. I've never used an external viewer, but it's good to know they exist.

### VERBOSE and SETTINGS

I've not used these a bunch.

[My source](https://www.cybertec-postgresql.com/en/how-to-interpret-postgresql-explain-analyze-output/) says that `VERBOSE` will add "output expressions" to the display of the Query Plan. He also says that in most cases it's just clutter. I have experience with those "most cases" so I don't really use it. He suggests it can be useful if your query executes stored functions as part of the process. I'll have to look into this in future.

He also mentions the `SETTINGS` parameter. I'm going to be honest here: I've never used this and I have no idea what it does. I'd bet that it could give useful information in some rare cases, especially if you're a DBA. I have it listed here for one day, when I'm at my whits' end and need something else to try.
