+++
date = '2026-03-20T22:47:05+01:00'
draft = true
title = 'Your PostgreSQL database is not slow, it's cold'
+++

## Introduction
intro here, why PG can be a fast in mem db. break the myth that RDBMS = slow disk I/O.

### 1. The Shared Buffer Cache Explained  
from disk to memory  
how a query reads: check buffer cache first, only go to disk on a miss  
clock sweep algorithm  

![clock-sweep](/assets/images/clock-sweep.svg)

### 2. The OS Cache  
the OS page cache (filesystem cache) sits below shared_buffers  
A page evicted from shared_buffers often still lives in OS cache — so "disk read" ≠ actual spinning rust / SSD access  
Double caching: why the common advice is shared_buffers = 25% of RAM — the rest is left for the OS cache  

### 3. Demonstration  
Docker examples

### 4. The Working Set (if workload fit entirely in memory or when it does not)
Most production DBs don't fit entirely in RAM — and don't need to. The working set (hot data queried frequently) is usually a fraction of total size  
How to measure your cache hit ratio: heap_blks_hit vs heap_blks_read from pg_stat_user_tables  
The punchline: at 99%+ hit ratio, Postgres is already serving from memory with full ACID — no Redis layer needed for most read workloads  
show pg_prewarm

### 5. Takeaways
Practical tuning: shared_buffers, effective_cache_size, pg_prewarm
cycle back to introduction




