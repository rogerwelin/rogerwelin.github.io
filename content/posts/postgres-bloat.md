+++
date = '2026-02-04T22:47:05+01:00'
draft = true
title = 'Postgres Bloat'
+++

## Introduction

![bloat](/assets/images/bloat.jpg)

Your PostgreSQL database keeps growing, even though the number of rows stays roughly the same. Disk usage climbs, queries slow down, and you wonder what the heck is happening.

It’s not a bug - it’s called bloat and it's baked into Postgres design. 

To understand why it happens we have to follow the lifecycle of a row from the moment it’s written to a physical file to the moment it becomes "dead" space.

This journey begins at the physical storage layer: **Pages and Tuples**.

### 1. The Physical Layer: Pages and Tuples
PostgreSQL stores tables as files on disk, divided into fixed-size pages (8 KB blocks). Inside each page are tuples—the physical versions of your rows. See below for a diagram of a Page:

![bloat](/assets/images/page.jpg)

Think of a page like a box with limited capacity. When you insert a row, PostgreSQL writes a tuple into the first page with available space. If that page is full, it allocates a new page and appends it to the table file. Each tuple contains not just your data, but also metadata that PostgreSQL uses to determine which transactions can see it (more on this in the MVCC section).

This page-based architecture has critical implications for how bloat develops:

**Pages are the unit of I/O**. PostgreSQL doesn't read individual rows—it reads entire 8 KB pages. Even if only one live tuple remains in a page, PostgreSQL must load the full page into memory and scan past any dead tuples to find it

**Pages don't shrink**. Once a page is allocated to a table, it stays part of that table file. Deleting rows frees space *within* pages but doesn't return pages to the filesystem. Your table file can only grow or stay the same size through normal operations.

**Multiple versions coexist**. A single logical row in your application may correspond to several physical tuples spread across different pages. Each UPDATE creates a new tuple in a potentially different page, while the old tuple remains on disk.

Here's the key insight: your disk usage is driven by the total number of tuple versions, not the number of rows you can query. A table with 1 million rows might consume as much space as one with 10 million rows if it has enough dead tuples from updates and deletes.

This physical layout sets the stage for bloat, but the real culprit is how Postgres handles concurrent updates (MVCC).


### 2. MVCC and Why Old Data Sticks Around  

![bloat](/assets/images/mvcc.jpg)


PostgreSQL uses MVCC (Multi-Version Concurrency Control) to allow one person to read data while another is writing it.
Instead of overwriting an old row (which would lock the table and slow everyone down), Postgres simply leaves the old version there and creates a new version (tuple) next to it.

Here's how this (quite paradoxical) plays out physically:

**UPDATEs create, they don't modify**. When you update a row, PostgreSQL doesn't find the existing tuple and change it. Instead, it writes a completely new tuple (often in a different page) containing the updated values. The old tuple stays exactly where it is on disk, but PostgreSQL marks it as obsolete by setting its xmax field to the transaction ID that performed the update.
 
**DELETEs don't erase**. When you delete a row, PostgreSQL doesn't remove the tuple from the page. It simply sets the tuple's `xmax` to indicate which transaction deleted it. The data remains on disk, taking up space.

The result of these two operations is tuple accumulation! Every update adds more storage for that row.  A table with frequent updates to the same rows can accumulate dozens of tuple versions for each logical row, even though only one version is visible to new queries.

These old tuple versions serve a purpose initially—they allow long-running transactions to see a consistent snapshot of the data. But once all transactions that could possibly need them have finished, these tuples become dead tuples: invisible to all queries, yet still consuming space in your pages.

This is where bloat emerges. Even though dead tuples won't appear in query results, PostgreSQL still has to:
* Load their pages from disk into memory
* Scan past them to find live tuples
* Keep them in your backup dumps

#### MVCC in Action
Let's watch bloat happen in real-time. We'll update a single row multiple times and see what PostgreSQL actually does on disk.

We'll start with a simple table using Postgres in docker:

```sql
$ docker run --name bloat-postgres -p 5432:5432 -e POSTGRES_PASSWORD=mypass -d postgres:18
$ docker exec -it bloat-postgres psql -U postgres

postgres=# CREATE TABLE users (id INT PRIMARY KEY, name TEXT, email TEXT);
CREATE TABLE
postgres=# INSERT INTO users VALUES (1, 'Alice', 'alice@wonderland.com');
INSERT 0 1
```

Now let's update Alice's email three times:

```sql
UPDATE users SET email = 'alice@newdomain.com' WHERE id = 1;
UPDATE users SET email = 'alice@company.com' WHERE id = 1;
UPDATE users SET email = 'alice@final.com' WHERE id = 1;
```

From your application's perspective, you have one row. But PostgreSQL now has **four tuple versions** of that row sitting on disk, the original plus three updates. 

Every tuple in Postgres has hidden system columns that track its version:

- **ctid**: Physical location (page number, offset within page)
- **xmin**: Transaction ID that created this tuple
- **xmax**: Transaction ID that deleted/updated it (0 means it's still current)

Let's query these columns to see what's really there:

```sql
SELECT ctid, xmin, xmax, * FROM users;
-- you will see something similar to below:
postgres=# SELECT ctid, xmin, xmax, * FROM users;
 ctid  | xmin | xmax | id | name  |      email
-------+------+------+----+-------+-----------------
 (0,4) |  768 |    0 |  1 | Alice | alice@final.com
```

Look at the **ctid**: (0,4) means page 0, tuple offset 4. We inserted one row and did three updates—so why is this the *fourth* tuple? Because each UPDATE created a new tuple version. Tuples 1, 2, and 3 are still on disk with xmax values marking them as superseded, but your query doesn't see them; they're dead.

We can see dead tuple count if we check the table statistics:


```sql
postgres=# SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'users';
 n_live_tup | n_dead_tup
------------+------------
          1 |          3
 ```

Three dead tuples from just thee updates. This is how bloat accumulates.

Here's what actually happened on disk when we ran those updates:

```sql
Page 0, Tuple 1: [id=1, email='alice@example.com',     xmin=1000, xmax=1001] ← dead
Page 0, Tuple 2: [id=1, email='alice@newdomain.com',   xmin=1001, xmax=1002] ← dead
Page 0, Tuple 3: [id=1, email='alice@company.com',     xmin=1002, xmax=1003] ← dead
Page 0, Tuple 4: [id=1, email='alice@final.com',       xmin=1003, xmax=0]    ← current
```

Each UPDATE created a new tuple and marked the old one as superseded by setting its xmax to the updating transaction's ID. The old tuples aren't erased immediately—they're marked as dead, and the space they occupy can be reclaimed and reused for new data.

**Why Not Just Delete the Old Versions Immediately?**  
Because of MVCC. When transaction 1003 was updating Alice's email, another transaction (say, 1002) might have still been running. That transaction started before the update, so it should see the old email address. PostgreSQL keeps the old tuple versions around until it's certain no transaction needs them anymore.

When you query the table, PostgreSQL scans through the page and checks each tuple's xmin and xmax against your transaction's snapshot to determine which version you should see.


Dead tuples waste space and slow down queries. Even though they won't appear in your results, PostgreSQL must:

Load pages containing dead tuples from disk into memory
Scan past dead tuples to find live ones
Check each tuple's visibility (xmin/xmax) against the transaction snapshot

The more dead tuples per page, the more wasted I/O and CPU. A heavily bloated table might have pages that are 80% dead tuples—you're reading 5x more data than necessary. This is why bloat isn't just a disk space problem. It's a performance problem. And it's why understanding MVCC is essential to understanding PostgreSQL's behavior under write-heavy workloads.

So does PostgreSQL really leave dead tuples on disk forever? No, PostgreSQL has a solution: VACUUM. But it comes with tradeoffs and quirks you need to understand. Let's see it in action right away:

```sql
postgres=# vacuum users;
VACUUM
postgres=# SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'users';
 n_live_tup | n_dead_tup
------------+------------
          1 |          0
```

No more dead tuples!


### 3. VACUUM  




