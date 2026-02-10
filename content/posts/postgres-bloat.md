+++
date = '2026-02-04T22:47:05+01:00'
draft = true
title = 'Postgres Bloat'
+++

## Introduction

![bloat](/assets/images/bloat.jpg)

Your PostgreSQL database keeps growing, even though the number of rows stays roughly the same. Disk usage climbs, queries slow down, and you wonder what the heck is happening.

It’s not a bug - it’s called bloat and it's baked into the core design of Postgres.

To understand why it happens we have to follow the lifecycle of a row from the moment it’s written to a physical file to the moment it becomes "dead" space. This journey begins at the physical storage layer: **Pages and Tuples**.

### 1. The Physical Layer: Pages and Tuples
Postgres stores tables as files on disk, divided into fixed-size **pages** (8 KB blocks). Inside each page are **tuples** - the physical versions of your rows. See below for a diagram of a Page:

![bloat](/assets/images/page.jpg)

Think of a page like a box with limited capacity. When you insert a row, Postgres writes a tuple into the first page with available space. If that page is full, it allocates a new page and appends it to the table file. Each tuple contains not just your data, but also metadata that Postgres uses to determine which transactions can see it (more on this in the MVCC section).

This page-based architecture has critical implications for how bloat develops:

**Pages are the unit of I/O**. Postgres doesn't read individual rows—it reads entire 8 KB pages. Even if only one live tuple remains in a page, Postgres must load the full page into memory and scan past any dead tuples to find it

**Pages don't shrink**. Once a page is allocated to a table, it stays part of that table file. Deleting rows frees space *within* pages but doesn't return pages to the filesystem. Your table file can only grow or stay the same size through normal operations.

**Multiple versions coexist**. A single logical row in your application may correspond to several physical tuples spread across different pages. Each UPDATE creates a new tuple in a potentially different page, while the old tuple remains on disk.

Here's the key insight: your disk usage is driven by the total number of tuple versions, not the number of rows you can query. A table with 1 million rows might consume as much space as one with 10 million rows if it has enough dead tuples from updates and deletes.

But pages alone don't cause bloat. The real culprit is how Postgres handles concurrent users: MVCC.

### 2. MVCC and Why Old Data Sticks Around  

![bloat](/assets/images/mvcc.jpg)

Postgres uses **Multi-Version Concurrency Control** (MVCC) so readers never block writers and writers never block readers. The mechanism is simple: instead of overwriting data in place, Postgres creates new versions.

Here's what actually happens at the physical level:

**UPDATEs create, they don't modify**. When you update a row, Postgres doesn't find the existing tuple and change it. Instead, it writes a completely new tuple (often in a different page) containing the updated values. The old tuple stays exactly where it is on disk, but Postgres marks it as obsolete by setting its xmax field to the transaction ID that performed the update.
 
**DELETEs don't erase**. When you delete a row, Postgres doesn't remove the tuple from the page. It simply sets the tuple's `xmax` to indicate which transaction deleted it. The data remains on disk, taking up space.

The result is tuple accumulation. Every update adds storage, every delete leaves behind a dead tuple. A table with frequent updates can accumulate dozens of tuple versions for each logical row, even though only one version is visible to new queries.

But Why not just delete the old versions immediately? Because of concurrent transactions. When one transaction is updating a row, another transaction that started earlier might still be reading the old version. Postgres keeps old tuple versions around until it's certain no transaction needs them anymore. Only then can the space be reclaimed.

These old tuple versions serve a purpose initially—they allow long-running transactions to see a consistent snapshot of the data. But once all transactions that could possibly need them have finished, these tuples become dead tuples: invisible to all queries, yet still consuming space in your pages.

This is where bloat emerges. Even though dead tuples won't appear in query results, Postgres still has to:
* Load their pages from disk into memory
* Scan past them to find live tuples
* Keep them in your backup dumps

#### MVCC in Action
Let's watch bloat happen in real-time. We'll update a single row multiple times and see what Postgres actually does on disk.

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

From your application's perspective, you have one row. But Postgres now has **four tuple versions** of that row sitting on disk, the original plus three updates. 

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

Three dead tuples from three updates. In a production table with millions of updates per day, this is how real bloat accumulates.

Here's what actually happened on disk when we ran those updates:

```sql
Page 0, Tuple 1: [id=1, email='alice@example.com',     xmin=1000, xmax=1001] ← dead
Page 0, Tuple 2: [id=1, email='alice@newdomain.com',   xmin=1001, xmax=1002] ← dead
Page 0, Tuple 3: [id=1, email='alice@company.com',     xmin=1002, xmax=1003] ← dead
Page 0, Tuple 4: [id=1, email='alice@final.com',       xmin=1003, xmax=0]    ← current
```

Each UPDATE created a new tuple and marked the old one as superseded by setting its xmax to the updating transaction's ID. The old tuples aren't erased immediately—they're marked as dead, and the space they occupy can be reclaimed and reused for new data.

When you query the table, Postgres scans through the page and checks each tuple's xmin and xmax against your transaction's snapshot to determine which version you should see.

Dead tuples waste space and slow down queries. Even though they won't appear in your results, Postgres must:

* Load pages containing dead tuples from disk into memory
* Scan past dead tuples to find live ones
* Check each tuple's visibility (xmin/xmax) against the transaction snapshot

The more dead tuples per page, the more wasted I/O and CPU. A heavily bloated table might have pages that are 80% dead tuples—you're reading 5x more data than necessary. This is why bloat isn't just a disk space problem. It's a performance problem. And it's why understanding MVCC is essential to understanding Postgres behavior under write-heavy workloads.

At this point, you might be thinking: if every update creates dead tuples, won't databases just fill up with garbage? Yes - unless something cleans them up. Postgres answer is **VACUUM**, an automated background process that reclaims dead tuples space. But VACUUM comes with tradeoffs and quirks you need to understand. Let's see how it works an where it struggles.

So does Postgres really leave dead tuples on disk forever? No, Postgres has a solution: VACUUM. But it comes with tradeoffs and quirks you need to understand. Let's see it in action right away:

### 3. VACUUM  
In progress..

```sql
postgres=# vacuum users;
VACUUM
postgres=# SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'users';
 n_live_tup | n_dead_tup
------------+------------
          1 |          0
```

No more dead tuples!





