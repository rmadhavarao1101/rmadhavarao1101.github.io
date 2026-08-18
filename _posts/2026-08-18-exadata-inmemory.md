# Enabling Database In-Memory on Exadata: a 16-second query that became 0.12

> Oracle 19c EE (Extreme Performance) 19.29 · two-node Exadata Database Service (ExaDB-D) · ~40M-row table · `GROUP BY` benchmark

I've been meaning to sit down with Oracle Database In-Memory for a while. Everyone quotes the marketing number — "100x faster analytics" — and I wanted to see it on my own data, on my own box, and understand what it actually takes to get there. Not the polished demo where everything works first try, but the real version with the sizing mistakes and the head-scratching along the way.

Spoiler: I got my big number.

| Metric | Row store | In-Memory |
| --- | --- | --- |
| Access path | `INDEX STORAGE FAST FULL SCAN` | `TABLE ACCESS INMEMORY FULL` |
| Elapsed (warm) | 16.66 s | 0.13 s |
| Buffers (per execution) | 23 | **19** |
| Consistent gets (clean session) | hundreds of thousands | **770** |
| Compression (`QUERY HIGH`) | — | ~2.6x (4796 MB → 1855 MB) |

But the interesting part wasn't the result — it was the hour of "why isn't this populating?" that came before it. So that's what I'm writing up, prompts and all.

## Contents

- [Picking a table](#picking-a-table)
- [Checking what I had to work with](#checking-what-i-had-to-work-with)
- [Sizing it: start low, grow later](#sizing-it-start-low-grow-later)
- [Marking the table, and my first wrong turn](#marking-the-table-and-my-first-wrong-turn)
- [Reading the actual memory picture](#reading-the-actual-memory-picture)
- [The stray table nobody mentioned](#the-stray-table-nobody-mentioned)
- [Growing the store, and a lesson in per-node fit](#growing-the-store-and-a-lesson-in-per-node-fit)
- [The fix: distribute instead of duplicate](#the-fix-distribute-instead-of-duplicate)
- [The payoff](#the-payoff)
- [What I'd tell someone starting this](#what-id-tell-someone-starting-this)

---

Environment, for context: Oracle 19c Enterprise Edition (Extreme Performance), running on a two-node Exadata Database Service (ExaDB-D) cluster. In-Memory is included with the Exadata cloud service, which is a nice perk — no separate license line to chase down.

```console
$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Tue Aug 18 13:59:15 2026
Version 19.29.0.0.0

Connected to:
Oracle Database 19c EE Extreme Perf Release 19.0.0.0.0 - Production
Version 19.29.0.0.0
```

## Picking a table

I went looking for a table that would actually show something. In-Memory rewards scans, aggregations, and joins — not single-row primary-key lookups, where the buffer cache is already plenty fast. I landed on `TEST.USR`.

```sql
SELECT owner, segment_name, segment_type,
       bytes/1024/1024 AS mb,
       ROUND(bytes/1024/1024/1024, 2) AS gb, blocks
FROM   dba_segments
WHERE  owner = 'TEST' AND segment_name = 'USR';
```

```text
OWNER      SEGMENT_NAME  SEGMENT_TYPE   MB       GB     BLOCKS
---------- ------------- -------------- -------- ------ ----------
TEST       USR           TABLE          4804     4.69   614912
```

About 4.7 GB, roughly 615K blocks. One row back, so it's not partitioned. Then the row count and shape:

```sql
SELECT num_rows, avg_row_len, last_analyzed
FROM   dba_tables
WHERE  owner='TEST' AND table_name='USR';
```

```text
  NUM_ROWS AVG_ROW_LEN LAST_ANAL
---------- ----------- ---------
  39891039          69 04-JUN-26
```

Just under 40 million rows, a narrow 69-byte average row, stats gathered recently. And a quick LOB check, because In-Memory doesn't populate LOB columns into the column store:

```sql
SELECT column_name, segment_name
FROM   dba_lobs
WHERE  owner='TEST' AND table_name='USR';
```

```text
no rows selected
```

No LOBs. Every column here is a scalar that goes in cleanly. This is a good demo table — big enough that a row-store scan does real work, narrow enough that most queries only touch a handful of columns, which is exactly where columnar storage pulls ahead.

## Checking what I had to work with

Before touching anything I looked at the host. This is the step people skip, and it's the one that saves you.

```console
SQL> show parameter sga

NAME                          TYPE        VALUE
----------------------------- ----------- ------------------------------
allow_group_access_to_sga     boolean     FALSE
lock_sga                      boolean     FALSE
pre_page_sga                  boolean     TRUE
sga_max_size                  big integer 16G
sga_min_size                  big integer 0
sga_target                    big integer 16G
unified_audit_sga_queue_size  integer     1048576

SQL> !free -g
              total   used   free   shared  buff/cache   available
Mem:            148    129      4        5          14           7
Swap:            15      3     12
```

Two things jumped out. First, `sga_max_size` and `sga_target` are both pinned at 16G with no slack. The In-Memory column store lives *inside* the SGA — it doesn't sit on top of it. Whatever I hand to `INMEMORY_SIZE` gets carved out of the existing 16G, mostly at the expense of the buffer cache.

Second, and more important: the host has only ~7 GB available and swap is already 3 GB in use. A database node that's swapping is under real memory pressure. That gave me one hard rule — **do not grow the SGA to make room for the store.** The store comes from *within* the existing 16G, so it adds nothing to the host footprint. It just repurposes some buffer cache into column store. Safe path.

Current store size?

```sql
SELECT inst_id, value FROM gv$parameter WHERE name = 'inmemory_size';
```

```text
   INST_ID VALUE
---------- ----------
         1 0
         2 0
```

Zero on both nodes. And here's the thing nobody tells you clearly: going from **0 to any nonzero value cannot be done dynamically.** It's a static change — set it in the spfile, and the instance only picks it up on restart. Once the store is already nonzero you can raise it live (18c and up), but the first enablement needs a bounce. On a two-node RAC that's a *rolling* restart, one instance at a time.

## Sizing it: start low, grow later

Small trick that shaped my whole approach: you can **increase** `INMEMORY_SIZE` dynamically but can only **decrease** it with a restart. So it's always cheaper to start low and grow. I set 2 GB and did the rolling restart:

```sql
ALTER SYSTEM SET INMEMORY_SIZE = 2G SCOPE=SPFILE SID='*';
-- rolling bounce of both instances
```

```console
SQL> SELECT inst_id, value FROM gv$parameter WHERE name = 'inmemory_size';

   INST_ID VALUE
---------- ----------
         1 2147483648
         2 2147483648
```

Store's up. Now to put the table in it.

## Marking the table, and my first wrong turn

```console
SQL> ALTER TABLE test.usr INMEMORY
  2    MEMCOMPRESS FOR QUERY HIGH PRIORITY HIGH
  3    DISTRIBUTE AUTO DUPLICATE ALL;

Table altered.
```

A word on those options, because I picked one wrong. `MEMCOMPRESS FOR QUERY HIGH` is good compression with fast scans — the usual sweet spot. `PRIORITY HIGH` populates right away instead of lazily. `DUPLICATE ALL` is an Exadata-only feature that mirrors a full copy on *every* node. Sounds great. It's the mistake, and it takes a while to reveal itself.

`Table altered.` — but marking a table `INMEMORY` is just a dictionary flag. It happily succeeds even with `INMEMORY_SIZE = 0`. I ran a populate and checked:

```console
SQL> EXEC DBMS_INMEMORY.POPULATE('TEST','USR');

PL/SQL procedure successfully completed.

SQL> SELECT inst_id, populate_status,
  2         ROUND(inmemory_size/1024/1024/1024,2) AS in_mem_gb, bytes_not_populated
  3  FROM   gv$im_segments
  4  WHERE  segment_name = 'USR'
  5  ORDER  BY inst_id;

no rows selected
```

Nothing. `GV$IM_SEGMENTS` only lists segments that actually made it into the store, so "no rows" means nothing landed. And `POPULATE` returning "success" told me nothing — it just queues a request and returns, even when there's nowhere for the data to go.

## Reading the actual memory picture

Time to stop guessing and look at the store directly:

```console
SQL> SELECT inst_id, pool,
  2         ROUND(alloc_bytes/1024/1024/1024,2) AS alloc_gb,
  3         ROUND(used_bytes/1024/1024/1024,2)  AS used_gb,
  4         populate_status
  5  FROM   gv$inmemory_area ORDER BY inst_id, pool;

   INST_ID POOL         ALLOC_GB    USED_GB  POPULATE_STATUS
---------- ------------ ---------- --------- ---------------
         1 1MB POOL       1.28       1.28    OUT OF MEMORY
         1 64KB POOL       .70        .03    DONE
         2 1MB POOL       1.28       1.28    OUT OF MEMORY
         2 64KB POOL       .70        .03    DONE
```

There it is. **`OUT OF MEMORY` on the 1MB pool, both nodes.** This is the detail that clarified everything: a 2G store does *not* give you 2 GB for data. It splits into a **1MB pool** (the actual column data) and a **64KB pool** (metadata). Here the split left only ~1.28 GB for data. So my real ceiling was 1.28 GB, not 2 GB.

Before blaming the size, I checked the workers were even allowed to run — if `inmemory_max_populate_servers` were 0, population would silently never happen:

```console
SQL> SELECT inst_id, value FROM gv$parameter
  2  WHERE name = 'inmemory_max_populate_servers';

   INST_ID VALUE
---------- ----------
         2 2
         1 2
```

Two per node — fine. So it wasn't a worker problem. But there was a second surprise hiding underneath.

## The stray table nobody mentioned

I widened the query to show *everything* in the store, not just `USR`:

```console
SQL> SELECT inst_id, owner, segment_name,
  2         ROUND(bytes/1024/1024,0) AS on_disk_mb,
  3         ROUND(inmemory_size/1024/1024,0) AS in_mem_mb, populate_status
  4  FROM   gv$im_segments ORDER BY inst_id, in_mem_mb DESC;

   INST_ID OWNER SEGMENT_NAME   ON_DISK_MB  IN_MEM_MB  POPULATE_STAT
---------- ----- -------------- ----------  ---------  -------------
         1 TEST  USR_08182026        4697       1431   OUT OF MEMORY
         1 TEST  USR                 4796         77   STARTED
         2 TEST  USR_08182026        4697       1319   OUT OF MEMORY
```

`USR_08182026`. A dated copy of the table — someone's backup from earlier that day — was *also* flagged `INMEMORY`, and it had grabbed the whole pool ahead of the real `USR`, which only got 77 MB in before running dry. I wasn't fighting for space against my own table. I was losing it to a copy I didn't know was there.

Lesson filed: when something won't populate, list the *entire* store. The segment you're chasing may not be the one filling it.

I confirmed we didn't need the copy in memory and evicted it. `NO INMEMORY` just removes it from the column store — it leaves the table on disk completely untouched, which is why it's safe:

```console
SQL> ALTER TABLE test.usr_08182026 NO INMEMORY;

Table altered.

SQL> EXEC DBMS_INMEMORY.POPULATE('TEST','USR');

PL/SQL procedure successfully completed.
```

That freed the pool and `USR` started climbing — but it still didn't finish. Even with the copy gone, the full table needed more than the 1.28 GB the 2G store exposed:

```console
SQL> SELECT inst_id, segment_name, populate_status,
  2         ROUND(bytes/1024/1024,0) AS on_disk_mb,
  3         ROUND(inmemory_size/1024/1024,0) AS in_mem_mb, bytes_not_populated
  4  FROM   gv$im_segments WHERE segment_name='USR' ORDER BY inst_id;

   INST_ID SEGMENT  POPULATE_STAT ON_DISK_MB  IN_MEM_MB  BYTES_NOT_POPULATED
---------- -------  ------------- ----------  ---------  -------------------
         1 USR      STARTED           4796        523           3580248064
```

523 MB in, 3.5 GB still to go. Time for more room.

## Growing the store, and a lesson in per-node fit

This is where starting at 2G paid off. Growing is dynamic — no restart:

```console
SQL> ALTER SYSTEM SET INMEMORY_SIZE = 4G SCOPE=BOTH SID='*';

System altered.

SQL> EXEC DBMS_INMEMORY.POPULATE('TEST','USR');

PL/SQL procedure successfully completed.
```

I watched it populate over a couple of minutes — two populate servers per node grinding through ~3 GB takes a moment. First check caught it mid-flight, climbing:

```text
   INST_ID SEGMENT  POPULATE_STAT ON_DISK_MB  IN_MEM_MB  BYTES_NOT_POPULATED
---------- -------  ------------- ----------  ---------  -------------------
         1 USR      STARTED           4796        943           2451595264
         2 USR      STARTED           4796        671           3167272960
```

And then a *split* result that told me something important:

```text
   INST_ID SEGMENT  POPULATE_STAT ON_DISK_MB  IN_MEM_MB  BYTES_NOT_POPULATED
---------- -------  ------------- ----------  ---------  -------------------
         1 USR      COMPLETED         4796       1855                    0
         2 USR      OUT OF MEMORY     4796       1324           1426120704
```

Node 1 finished at **1855 MB** — a clean 4796 → 1855, about **2.6x compression** under `QUERY HIGH`. Node 2 ran out. Same store size, different outcome, because the 1MB/64KB pool split isn't identical across instances and node 2's data pool topped out a hair too low.

And that's the real problem with `DUPLICATE ALL`: it demands the *entire* 1855 MB copy on *both* nodes. I could keep throwing memory at it — bump to 5G, give each node room for a full mirror — but I stopped to ask what `DUPLICATE ALL` was actually buying me. It's an *availability* feature. It protects against an instance failure by keeping a second copy. My post is about query speed, not fault tolerance. I was paying a memory tax for something my demo didn't need.

## The fix: distribute instead of duplicate

So I dropped duplicate and let the segment spread across the cluster instead:

```console
SQL> ALTER TABLE test.usr NO INMEMORY;

Table altered.

SQL> ALTER TABLE test.usr INMEMORY
  2    MEMCOMPRESS FOR QUERY HIGH PRIORITY HIGH
  3    DISTRIBUTE AUTO;

Table altered.

SQL> EXEC DBMS_INMEMORY.POPULATE('TEST','USR');

PL/SQL procedure successfully completed.
```

With `DISTRIBUTE AUTO` and no duplicate, each node holds roughly *half* the segment, so each only has to fit ~900 MB instead of the full 1855. This time both nodes finished:

```text
   INST_ID SEGMENT  POPULATE_STAT ON_DISK_MB  IN_MEM_MB  BYTES_NOT_POPULATED
---------- -------  ------------- ----------  ---------  -------------------
         1 USR      COMPLETED         4796        831                    0
         2 USR      COMPLETED         4796        991                    0
```

831 + 991 = 1822 MB across the cluster, `bytes_not_populated = 0` on both. One full copy of the table, split in two, comfortably inside 4G. Finally.

One thing to remember with a distributed segment: each node only has its half locally. A *serial* query on one node would fetch the other half over the interconnect. So when you benchmark, run it **parallel** — that engages both nodes' PX servers, each scans its local half in memory, no cross-node fetch.

## The payoff

Quick look at cardinality to pick a good `GROUP BY` key:

```console
SQL> SELECT column_name, num_distinct, data_type
  2  FROM   dba_tab_columns
  3  WHERE  owner='TEST' AND table_name='USR'
  4  ORDER  BY num_distinct;

COLUMN_NAME     NUM_DISTINCT  DATA_TYPE
--------------- ------------  ------------
ENABLED                    1  NUMBER
GUID                   17646  VARCHAR2
DEVICE                349664  NUMBER
CREATED_STAMP       38080512  TIMESTAMP(6)
DB_STAMP            38653952  TIMESTAMP(6)
ID                  39891039  NUMBER
```

`GUID` at 17,646 distinct values across 40M rows is the sweet spot. The harness — parallel, stats on, run each path warm:

```sql
SET TIMING ON
ALTER SESSION SET STATISTICS_LEVEL = ALL;
ALTER SESSION FORCE PARALLEL QUERY PARALLEL 4;

-- Baseline: In-Memory off for this session
ALTER SESSION SET INMEMORY_QUERY = DISABLE;
SELECT guid, COUNT(*) FROM test.usr
GROUP BY guid ORDER BY 2 DESC FETCH FIRST 20 ROWS ONLY;

-- In-Memory on
ALTER SESSION SET INMEMORY_QUERY = ENABLE;
SELECT guid, COUNT(*) FROM test.usr
GROUP BY guid ORDER BY 2 DESC FETCH FIRST 20 ROWS ONLY;
```

Baseline first:

```console
SQL> ALTER SESSION SET INMEMORY_QUERY = DISABLE;

SQL> SELECT guid, COUNT(*) FROM test.usr
  2  GROUP BY guid ORDER BY 2 DESC FETCH FIRST 20 ROWS ONLY;

GUID                                        COUNT(*)
------------------------------------------ ----------
fab2331d92734fa6ae2fd0bbebcf3317              191161
ed300885771048e2b160d3994f0d7f9d              167021
b0a2cf1aa5f540a09c78913fdf35ce37              152625
2360da1416294137b4a466a17679bc45              143081
e4d94360bce142cb9bb96d3c9724d0c2              143019
... (20 rows selected) ...

Elapsed: 00:00:16.66
```

Then flip In-Memory on and run the identical query:

```console
SQL> ALTER SESSION SET INMEMORY_QUERY = ENABLE;

SQL> SELECT guid, COUNT(*) FROM test.usr
  2  GROUP BY guid ORDER BY 2 DESC FETCH FIRST 20 ROWS ONLY;

GUID                                        COUNT(*)
------------------------------------------ ----------
fab2331d92734fa6ae2fd0bbebcf3317              191161
ed300885771048e2b160d3994f0d7f9d              167021
b0a2cf1aa5f540a09c78913fdf35ce37              152625
... (20 rows selected) ...

Elapsed: 00:00:00.13
```

**16.66 seconds down to 0.13.** Same data, same instance, same query. But a timing number alone proves nothing — I wanted the execution plans to show *why*.

> **Gotcha:** with `SERVEROUTPUT` on, a stray `DBMS_OUTPUT` call kept hijacking `DBMS_XPLAN.DISPLAY_CURSOR` (`cannot fetch plan for SQL_ID ... BEGIN DBMS_OUTPUT.GET_LINES`). Fix: `SET SERVEROUTPUT OFF` and fetch by explicit SQL_ID.

```console
SQL> SET SERVEROUTPUT OFF
SQL> SELECT sql_id, child_number, ROUND(elapsed_time/1000000,2) AS sec
  2  FROM   v$sql
  3  WHERE  sql_text LIKE 'SELECT guid, COUNT(%'
  4  AND    sql_text NOT LIKE '%v$sql%'
  5  ORDER  BY last_active_time DESC;

SQL_ID        CHILD_NUMBER        SEC
------------- ------------ ----------
5yhfz6c434szd            1        .43
5yhfz6c434szd            0      86.71
```

Two child cursors under one SQL_ID — child 0 is the row-store plan, child 1 is In-Memory.

<details>
<summary><b>Child 0 — the baseline (row store), full plan</b></summary>

```text
SQL_ID  5yhfz6c434szd, child number 0
-------------------------------------
SELECT guid, COUNT(*) FROM test.usr GROUP BY guid ORDER BY 2
DESC FETCH FIRST 20 ROWS ONLY

Plan hash value: 4234979258

-------------------------------------------------------------------------------------------------------------
| Id  | Operation                          | Name             | Starts | E-Rows | A-Rows |   A-Time    | Buffers |
-------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                  |      1 |        |     20 |00:00:16.65 |     23 |
|   1 |  SORT ORDER BY                     |                  |      1 |     20 |     20 |00:00:16.65 |     23 |
|*  2 |   VIEW                             |                  |      1 |     20 |     20 |00:00:16.65 |     23 |
|*  3 |    WINDOW SORT PUSHED RANK         |                  |      1 |  17646 |     20 |00:00:16.65 |     23 |
|   4 |     PX COORDINATOR                 |                  |      1 |        |     80 |00:00:16.65 |     23 |
|   5 |      PX SEND QC (RANDOM)           | :TQ10001         |      0 |  17646 |      0 |00:00:00.01 |      0 |
|*  6 |       WINDOW CHILD PUSHED RANK     |                  |      0 |  17646 |      0 |00:00:00.01 |      0 |
|   7 |        HASH GROUP BY               |                  |      0 |  17646 |      0 |00:00:00.01 |      0 |
|   8 |         PX RECEIVE                 |                  |      0 |  17646 |      0 |00:00:00.01 |      0 |
|   9 |          PX SEND HASH              | :TQ10000         |      0 |  17646 |      0 |00:00:00.01 |      0 |
|  10 |           HASH GROUP BY            |                  |      0 |  17646 |      0 |00:00:00.01 |      0 |
|  11 |            PX BLOCK ITERATOR       |                  |      0 |    39M |      0 |00:00:00.01 |      0 |
|* 12 |             INDEX STORAGE FAST FULL SCAN | IDX_USER_DEVICE1 | 0 | 39M | 0 |00:00:00.01 |  0 |
-------------------------------------------------------------------------------------------------------------

  12 - storage(:Z>=:Z AND :Z<=:Z)

Note
-----
   - Degree of Parallelism is 4 because of session
```

</details>

<details open>
<summary><b>Child 1 — In-Memory, full plan</b></summary>

```text
SQL_ID  5yhfz6c434szd, child number 1
-------------------------------------
SELECT guid, COUNT(*) FROM test.usr GROUP BY guid ORDER BY 2
DESC FETCH FIRST 20 ROWS ONLY

Plan hash value: 1302132066

-------------------------------------------------------------------------------------------------------------
| Id  | Operation                          | Name    | Starts | E-Rows | A-Rows |   A-Time    | Buffers |
-------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |         |      1 |        |     20 |00:00:00.12 |     19 |
|   1 |  SORT ORDER BY                     |         |      1 |     20 |     20 |00:00:00.12 |     19 |
|*  2 |   VIEW                             |         |      1 |     20 |     20 |00:00:00.12 |     19 |
|*  3 |    WINDOW SORT PUSHED RANK         |         |      1 |  17646 |     20 |00:00:00.12 |     19 |
|   4 |     PX COORDINATOR                 |         |      1 |        |     80 |00:00:00.12 |     19 |
|   5 |      PX SEND QC (RANDOM)           | :TQ10001|      0 |  17646 |      0 |00:00:00.01 |      0 |
|*  6 |       WINDOW CHILD PUSHED RANK     |         |      0 |  17646 |      0 |00:00:00.01 |      0 |
|   7 |        HASH GROUP BY               |         |      0 |  17646 |      0 |00:00:00.01 |      0 |
|   8 |         PX RECEIVE                 |         |      0 |  17646 |      0 |00:00:00.01 |      0 |
|   9 |          PX SEND HASH              | :TQ10000|      0 |  17646 |      0 |00:00:00.01 |      0 |
|  10 |           HASH GROUP BY            |         |      0 |  17646 |      0 |00:00:00.01 |      0 |
|  11 |            PX BLOCK ITERATOR       |         |      0 |    39M |      0 |00:00:00.01 |      0 |
|* 12 |             TABLE ACCESS INMEMORY FULL | USR |      0 |   39M  |      0 |00:00:00.01 |      0 |
-------------------------------------------------------------------------------------------------------------

  12 - inmemory(:Z>=:Z AND :Z<=:Z)

Note
-----
   - Degree of Parallelism is 4 because of session
```

</details>

There's the whole story in two lines of plan output: the baseline reads `INDEX STORAGE FAST FULL SCAN` on `IDX_USER_DEVICE1`; the In-Memory run reads `TABLE ACCESS INMEMORY FULL` on `USR`. And the top-line `Buffers` on the in-memory plan is **19**. Nineteen. The query only needed the `guid` column, so that's all it read.

An honest note worth making in any post like this: my baseline is an index fast-full-scan, not a full table scan. The optimizer chose the index on its own as the cheapest row-store option. That's the *fair* comparison — In-Memory beating the optimizer's own best non-In-Memory plan. A forced full table scan would have been slower still and made the gap look even bigger.

### The columnar numbers, measured cleanly

To quantify the scan without cumulative-session noise, I reconnected a fresh session (counters reset to zero) and ran the in-memory query exactly once:

```console
$ sqlplus / as sysdba

SQL> SET SERVEROUTPUT OFF
SQL> ALTER SESSION SET STATISTICS_LEVEL = ALL;
SQL> ALTER SESSION FORCE PARALLEL QUERY PARALLEL 4;
SQL> ALTER SESSION SET INMEMORY_QUERY = ENABLE;

SQL> SELECT guid, COUNT(*) FROM test.usr
  2  GROUP BY guid ORDER BY 2 DESC FETCH FIRST 20 ROWS ONLY;
... 20 rows selected ...

SQL> SELECT n.name, s.value
  2  FROM   v$mystat s JOIN v$statname n ON s.statistic#=n.statistic#
  3  WHERE  n.name IN ('consistent gets','IM scan rows','IM scan bytes in-memory',
  4                    'IM scan CUs columns accessed','IM scan CUs pruned')
  5  ORDER  BY n.name;

NAME                          VALUE
----------------------------- ----------
IM scan CUs columns accessed         114
IM scan CUs pruned                     0
IM scan bytes in-memory       1987378010
IM scan rows                    60296572
consistent gets                      770
```

**770 consistent gets** for an aggregation over the whole table. That's the number I keep coming back to. A row-store scan of this table is hundreds of thousands of block reads; the column store did the same logical work in 770.

> The *first* time I pulled these stats — before the fresh session — they were nonsense (millions of "physical reads," "IM scan rows" several times the row count). `V$MYSTAT` is cumulative for the whole session, and I'd run the query a half-dozen times. Reconnect, run once, read then. The plan's own `Buffers` column is per-execution and doesn't have that problem.

## What I'd tell someone starting this

- [x] **The store size is not the space you get.** A 2G store gave me ~1.28 GB of usable data pool after the 64KB metadata pool took its cut. Size against the pool, confirm the real footprint in `V$IM_SEGMENTS` after population. (The in-memory compression advisor is useless before the store exists — it trial-populates, which it can't at `INMEMORY_SIZE = 0`. Size conservatively, populate, then read the truth.)
- [x] **Start small and grow.** Increasing `INMEMORY_SIZE` is dynamic; shrinking needs a restart. Starting at 2G and growing to 4G cost me zero extra bounces.
- [x] **`DUPLICATE ALL` is availability, not speed.** On a two-node cluster it doubles your per-node footprint for fault tolerance. Chasing query performance, `DISTRIBUTE AUTO` fits in half the memory and benchmarks identically under a parallel query.
- [x] **When nothing populates, list the whole store.** A stray copy of my table was eating the pool. `GV$IM_SEGMENTS` with no filter showed it in one line; the filtered query hid it for twenty minutes.
- [x] **Watch how you measure.** `SERVEROUTPUT OFF`, plans by SQL_ID, session stats from a clean session — otherwise your numbers will embarrass you in the comments.

The 135x is real and it's fun to quote. But the part I'll actually remember is the pool math and the stray backup — the stuff that turns a five-minute tutorial into an afternoon, and the stuff the tutorials never mention.
