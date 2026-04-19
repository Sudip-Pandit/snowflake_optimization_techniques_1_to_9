# snowflake_optimization_techniques_1_to_9
# 32 Snowflake Optimizations That Cut Compute Costs in Production — Part 1: Query & Storage (Techniques 1–9)

*The patterns that silently drain your Snowflake bill — and the exact rewrites that fix them.*

---

## Table of Contents

1. [Who This Is For](#who-this-is-for)
2. [How to Use This Guide](#how-to-use-this-guide)
3. [The Cost Reality](#the-cost-reality)
4. [Technique 1 — Result Cache: Zero Credits for Repeated Queries](#technique-1--result-cache-zero-credits-for-repeated-queries)
5. [Technique 2 — Column Pruning: Never Write SELECT *](#technique-2--column-pruning-never-write-select-)
6. [Technique 3 — Predicate Pushdown: Filter Early, Filter Raw](#technique-3--predicate-pushdown-filter-early-filter-raw)
7. [Technique 4 — LIMIT with ORDER BY for Top-N Queries](#technique-4--limit-with-order-by-for-top-n-queries)
8. [Technique 5 — QUALIFY Instead of Nested Window Subqueries](#technique-5--qualify-instead-of-nested-window-subqueries)
9. [Technique 6 — Clustering Keys on High-Cardinality Filter Columns](#technique-6--clustering-keys-on-high-cardinality-filter-columns)
10. [Technique 7 — Monitor Clustering Depth and Reclustering Cost](#technique-7--monitor-clustering-depth-and-reclustering-cost)
11. [Technique 8 — Search Optimization Service for Point Lookups](#technique-8--search-optimization-service-for-point-lookups)
12. [Technique 9 — Flatten JSON Early: Avoid VARIANT in Hot Query Paths](#technique-9--flatten-json-early-avoid-variant-in-hot-query-paths)
13. [Part 1 Summary: Your Quick-Win Checklist](#part-1-summary-your-quick-win-checklist)
14. [What's Coming in Part 2](#whats-coming-in-part-2)
15. [Author's Note](#authors-note)

---

## Who This Is For

This guide is for analytics engineers, data platform engineers, and data leads who are running Snowflake in production and starting to feel the bill. You don't need to be a Snowflake expert — but you should be comfortable reading SQL and have access to your account's `SNOWFLAKE.ACCOUNT_USAGE` views.

If you're still in a free trial or running on toy datasets, bookmark this and come back when you're in production. These optimizations matter most when your tables have tens of millions of rows and your warehouse is running for hours every day.

---

## How to Use This Guide

This is Part 1 of a four-part series covering 32 Snowflake optimization techniques drawn from production deployments. Each technique follows the same structure: the anti-pattern, why it's expensive, the rewrite, and a realistic cost or performance impact.

**The series:**
- **Part 1 (this article)** — Query optimization and storage fundamentals (Techniques 1–9)
- **Part 2** — Warehouse tuning and join strategies (Techniques 10–16)
- **Part 3** — Advanced features and data loading (Techniques 17–24)
- **Part 4** — Table design, anti-patterns, and semi-structured data (Techniques 25–32)

**If you only have 10 minutes:** Read Techniques 1, 2, and 3. Result cache misses, `SELECT *`, and function-wrapped predicates account for the majority of avoidable compute waste I see across production Snowflake accounts. Fix those three first.

> **Effort key used throughout this guide:**
> `⚡ 5 min` = one-line change · `🔧 1 hour` = query rewrite · `🏗️ 1 sprint` = pipeline redesign

---

## The Cost Reality

I've reviewed Snowflake bills across data teams at companies of all sizes. The pattern is almost always the same.

A handful of anti-patterns — most of them invisible during development, all of them expensive in production — account for the majority of avoidable spend. Teams optimize the obvious things: they right-size warehouses, they set auto-suspend. But they miss the subtler ones: the `SELECT *` buried in a CTE that feeds three downstream joins, the `WHERE DATE_TRUNC(...)` that silently disables partition pruning on a billion-row table, the `UNION` that's sorting data that was already distinct.

This guide is about those patterns. Not the obvious ones. The ones that hide in plain sight until your bill arrives.

All performance figures in this article are based on production experience. Your results will vary by workload, warehouse size, and data distribution — but the directional impact is consistent.

---

## Section 1 — Query Optimization (Techniques 1–5)

The first five techniques live entirely inside your SQL. No infrastructure changes, no pipeline redesign — just rewrites. Some of these are five-minute fixes that can halve your query costs on frequently-run jobs.

---

## Technique 1 — Result Cache: Zero Credits for Repeated Queries

`Impact: HIGH` · `Effort: ⚡ 5 min`

Snowflake caches the output of every query you run for 24 hours. If you run the exact same query again and the underlying data hasn't changed, Snowflake returns the cached result instantly — zero compute credits burned, zero warehouse time.

This sounds obvious, but most teams unknowingly break the cache on every single run.

The culprit is non-deterministic functions. `CURRENT_TIMESTAMP()`, `RANDOM()`, `UUID_STRING()` — any function that returns a different value on each execution tells Snowflake the query is new, even if everything else is identical. The cache is bypassed. The warehouse spins up. Credits are consumed.

For dashboards and analyst workflows that re-run the same logic repeatedly, this is often the single biggest quick win available.

### What to avoid

```sql
-- WRONG: CURRENT_TIMESTAMP() changes every run — cache never reused
SELECT order_id, amount, CURRENT_TIMESTAMP() AS exported_at
FROM orders
WHERE status = 'PENDING';
```

### The fix

```sql
-- RIGHT: Static date literals — identical runs hit cache → 0 credits
SELECT region, SUM(revenue) AS total_revenue
FROM sales
WHERE sale_date BETWEEN '2024-01-01' AND '2024-03-31'
GROUP BY region;
```

### Rule

Replace dynamic timestamp calls in `WHERE` clauses with static date literals. Move any session-specific volatile values outside the query text — compute them in your application layer and pass them as parameters.

> **Watch for:** `CURRENT_TIMESTAMP()`, `CURRENT_DATE()`, `RANDOM()`, `UUID_STRING()`, `SYSDATE()` — all of these guarantee a cache miss on every execution.

---

## Technique 2 — Column Pruning: Never Write SELECT *

`Impact: HIGH` · `Effort: ⚡ 5 min`

This one is simple, it's well-known, and it's still everywhere in production codebases.

Snowflake stores data in columnar format. Each column lives in its own set of micro-partition files, independently of the others. When you write `SELECT *`, Snowflake must open and read every column's files from remote storage — including every column your query never uses.

On a wide table with 60 columns where you only need 4, that's roughly 93% unnecessary I/O. On a table that gets queried hundreds of times a day by a BI tool, the cost compounds fast.

The more insidious version: `SELECT *` buried inside a CTE or subquery that feeds a downstream join. The outer query might only reference 3 columns, but the inner `SELECT *` is already pulling all 60 from disk before the outer filter ever runs.

### What to avoid

```sql
-- WRONG: Reads all 60 columns from storage
SELECT * FROM orders WHERE status = 'SHIPPED';
```

### The fix

```sql
-- RIGHT: Reads only 4 columns — ~93% less storage I/O
SELECT order_id, customer_id, order_date, total_amount
FROM orders
WHERE status = 'SHIPPED';
```

### Impact at scale

| Pattern | Columns scanned | Storage I/O | Recommendation |
|---|---|---|---|
| `SELECT *` | All 60 | Full column scan | Never in production |
| Explicit column list | 4 (needed) | ~93% reduction | Always preferred |

> **Audit tip:** Search your codebase for `SELECT *` inside CTEs and subqueries — these are the ones that feed downstream joins and carry the hidden I/O tax furthest downstream.

---

## Technique 3 — Predicate Pushdown: Filter Early, Filter Raw

`Impact: HIGH` · `Effort: ⚡ 5 min`

This is the technique I see misunderstood most often, and it's responsible for some of the largest unnecessary full-table scans I've encountered in production.

Here's how Snowflake's partition pruning works: every micro-partition stores metadata containing the minimum and maximum value of each column it contains. When your `WHERE` clause is a plain range or equality comparison on a raw column — `WHERE event_ts >= '2024-01-01'` — Snowflake reads the metadata and skips any partition whose min/max range can't possibly contain matching rows. Thousands of partitions, never opened.

Pruning breaks the moment you wrap a column in a function.

`WHERE DATE_TRUNC('month', event_ts) = '2024-01-01'` — Snowflake cannot use the raw `event_ts` min/max metadata to evaluate this. The `DATE_TRUNC` is a computed value, not something stored in the partition metadata. So Snowflake falls back to scanning every partition and computing the function row by row.

One function call. Full table scan. Every time.

### What to avoid

```sql
-- WRONG: Function wraps the column — pruning disabled, full scan
WHERE DATE_TRUNC('month', event_ts) = '2024-01-01'
WHERE YEAR(order_date) = 2024
WHERE TO_CHAR(created_at, 'YYYY-MM') = '2024-01'
-- All three force a full scan — no partition pruning possible
```

### The fix

```sql
-- RIGHT: Direct range filter — pruning fires on every partition
WHERE event_ts  >= '2024-01-01' AND event_ts  < '2024-02-01'
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01'
```

### Rule to memorize

The raw column must appear alone on the left of the comparison operator. The moment you apply any function to it — `YEAR()`, `MONTH()`, `TO_CHAR()`, `UPPER()`, `LOWER()`, `CAST()`, `DATE_TRUNC()` — you've disabled pruning for that predicate.

> **⚠️ Watch for:** This pattern is especially common in WHERE clauses generated by BI tools and ORM layers. Check your query profile for high `partitions_scanned / partitions_total` ratios — that's the signal that pruning isn't firing.

---

## Technique 4 — LIMIT with ORDER BY for Top-N Queries

`Impact: MEDIUM` · `Effort: ⚡ 5 min`

When you need only the top N rows from a large result set, Snowflake uses a heap sort algorithm that keeps only the N best-seen rows in memory as it scans. It doesn't sort the entire dataset and then truncate — it stops accumulating once N rows are identified.

This is dramatically cheaper than sorting millions of rows and discarding all but the first hundred. But it only works when `ORDER BY` and `LIMIT` appear together in the same query.

- `LIMIT` without `ORDER BY`: non-deterministic — Snowflake may still scan the full table
- `ORDER BY` without `LIMIT`: forces a full sort of the entire result set — expensive

### The correct pattern

```sql
-- RIGHT: Heap sort stops after 100 rows found — efficient on any table size
SELECT customer_id, SUM(order_value) AS lifetime_value
FROM orders
GROUP BY customer_id
ORDER BY lifetime_value DESC
LIMIT 100;
```

### For deduplication use cases

Don't reach for `ORDER BY` + `LIMIT` when you want one row per group. Use `QUALIFY` with a window function instead — it avoids materializing a full sorted intermediate result:

```sql
-- Deduplicate: one row per customer, most recent order
SELECT *
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) = 1;
```

---

## Technique 5 — QUALIFY Instead of Nested Window Subqueries

`Impact: MEDIUM` · `Effort: ⚡ 5 min`

This is one of the most common SQL patterns in data engineering — and one of the most unnecessarily expensive versions of it is used constantly:

```sql
-- WRONG: Subquery wrapper materialises the full ranked result set
SELECT * FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
  FROM orders
) sub
WHERE rn = 1;
```

What's happening here: Snowflake computes the window function for every row in `orders`, writes the entire result including the `rn` column to an intermediate dataset in memory, and then filters it. If `orders` has 500 million rows, all 500 million are materialized before the filter runs.

Snowflake's `QUALIFY` clause was built for exactly this pattern. It filters the window function result inline — same query, same scan pass, no intermediate materialization:

```sql
-- RIGHT: Single scan, no subquery, no materialisation
SELECT *
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) = 1;
```

Identical result. Fraction of the memory. Shorter query plan.

`QUALIFY` works with any window function: `RANK()`, `DENSE_RANK()`, `SUM() OVER()`, `LAG()`, `LEAD()` — if you're filtering on a window function result anywhere in your codebase, `QUALIFY` is the right tool.

---

## Section 2 — Clustering & Storage (Techniques 6–9)

The next four techniques control how data is physically organized on disk. They determine how many micro-partitions Snowflake must read for any given query — and getting this right can mean the difference between a query that reads 2% of a table versus 100% of it.

---

## Technique 6 — Clustering Keys on High-Cardinality Filter Columns

`Impact: HIGH` · `Effort: 🔧 1 hour`

By default, Snowflake stores data in the order it was inserted. If your most common query filter is `WHERE order_date = '2024-01-15'` but rows landed in random date order, every micro-partition potentially contains rows from every date. Snowflake has no choice but to scan all of them.

A Clustering Key tells Snowflake to reorganize micro-partitions so rows with similar key values live together. Once clustered, a query filtering on `order_date` can skip the vast majority of partitions — Snowflake reads the metadata, sees that a partition's date range doesn't overlap the filter, and moves on. Snowflake's Automatic Clustering service maintains this continuously in the background as new data arrives.

### The right candidates

| Good candidate | Poor candidate | Reason |
|---|---|---|
| Date / timestamp columns | Boolean columns | Too few distinct values to cluster usefully |
| Region / country | UUID / surrogate keys | Too many distinct values, no range benefit |
| Status codes (hot filter) | Rarely queried columns | No queries benefit from the clustering |

### How to set it up

```sql
-- Define clustering key
ALTER TABLE orders CLUSTER BY (order_date);

-- Verify clustering health (target: average_depth < 6)
SELECT SYSTEM$CLUSTERING_INFORMATION('orders', '(order_date)');

-- Check pruning ratio in query history
SELECT
  query_id,
  partitions_scanned,
  partitions_total,
  ROUND(partitions_scanned / partitions_total * 100, 1) AS pct_scanned
FROM snowflake.account_usage.query_history
WHERE query_text ILIKE '%orders%'
ORDER BY start_time DESC
LIMIT 10;
```

> **⚠️ Cost warning:** Automatic Clustering consumes credits continuously to maintain organization. Only enable it on large tables (100 GB+) with high-frequency, selective range queries. Always check clustering depth before enabling — if your data already arrives in roughly sorted order (see Technique 23 in Part 3), you may not need it at all.

---

## Technique 7 — Monitor Clustering Depth and Reclustering Cost

`Impact: MEDIUM` · `Effort: 🔧 1 hour`

Clustering keys aren't set-and-forget. Heavy DML operations — bulk inserts, large `MERGE` statements, frequent deletes — deposit new rows without regard for clustering order. Over time this increases the *clustering depth*: how many micro-partitions, on average, Snowflake must check to find a single value.

A depth above 6 signals significant degradation. A table that was performing well six months ago may be scanning 5x more partitions today simply because it hasn't been re-examined.

### How to check

```sql
-- Returns JSON: average_depth, total_partition_count, etc.
SELECT PARSE_JSON(
  SYSTEM$CLUSTERING_INFORMATION('events', '(event_date, region)')
) AS cluster_info;

-- Monitor partition pruning ratio over recent queries
SELECT
  query_id,
  partitions_scanned,
  partitions_total,
  ROUND(partitions_scanned / partitions_total * 100, 1) AS pct_scanned
FROM snowflake.account_usage.query_history
WHERE query_text ILIKE '%events%'
  AND start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY pct_scanned DESC
LIMIT 20;
```

**Target metrics:** `average_depth` below 6 and `partitions_scanned / partitions_total` below 10% for well-clustered tables under typical query patterns.

If depth has risen significantly, weigh the reclustering credit cost against your query volume and scan ratios. Sometimes reducing DML frequency or switching to append-only ingestion is a better solution than paying for continuous reclustering.

---

## Technique 8 — Search Optimization Service for Point Lookups

`Impact: HIGH` · `Effort: 🔧 1 hour`

Clustering keys solve range queries. They do nothing for point lookups.

If your table is clustered by `order_date` and a customer service rep runs `WHERE user_id = 'abc123'` — a lookup on an unclustered column — Snowflake has no choice but to scan every partition to find that one user. On a table with billions of rows, that's a full scan every single time, even if it returns one row.

The Search Optimization Service builds and maintains a persistent search access path for equality lookups, substring searches, and geospatial queries. Once enabled on a column, point-lookup queries can skip 90% or more of micro-partitions — turning a full table scan into something that returns in seconds.

### When to use it

Customer service dashboards, order detail pages, audit log queries, any pattern where someone is looking up a specific entity by a non-clustered identifier.

```sql
-- Equality lookups on user_id and session_id
ALTER TABLE events
  ADD SEARCH OPTIMIZATION ON EQUALITY(user_id, session_id);

-- Substring / LIKE searches on event_name
ALTER TABLE events
  ADD SEARCH OPTIMIZATION ON SUBSTRING(event_name);

-- Inspect what is currently optimized
DESCRIBE SEARCH OPTIMIZATION ON events;
```

> **⚠️ Cost consideration:** The service charges for both storage (the access path) and compute (keeping the path current). Best ROI on large tables with more than 100 million rows where point-lookup queries run frequently. Audit your query profile for high-scan, low-result-count queries before enabling — those are your candidates.

---

## Technique 9 — Flatten JSON Early: Avoid VARIANT in Hot Query Paths

`Impact: HIGH` · `Effort: 🏗️ 1 sprint`

Snowflake's `VARIANT` type is genuinely useful for ingesting raw JSON. The problem is what happens when that raw JSON makes it into your query paths.

When you query a path like `raw_event:user_id::STRING` in a `WHERE` clause or `SELECT` list, Snowflake must deserialize the entire VARIANT blob at runtime for every row. It can't use columnar I/O. It can't use partition pruning — there's no min/max metadata for fields nested inside a VARIANT. And it's significantly slower than reading a native typed column.

The fix is to extract frequently queried fields into proper typed columns at the ingestion or transformation layer. Once those fields live in native columns, everything works: columnar I/O, partition pruning, result caching.

### What to avoid

```sql
-- WRONG: VARIANT path traversal — no pruning, full deserialisation per row
SELECT
  raw_event:user_id::STRING    AS user_id,
  raw_event:event_type::STRING AS event_type
FROM raw_events_variant
WHERE raw_event:event_type::STRING = 'click';
-- Cannot prune partitions on VARIANT path predicates
```

### The fix

```sql
-- RIGHT: Flatten to typed columns at ingestion or transformation
CREATE OR REPLACE TABLE events_flat AS
SELECT
  raw_event:user_id::STRING             AS user_id,
  raw_event:event_type::STRING          AS event_type,
  raw_event:properties:page_url::STRING AS page_url,
  TO_TIMESTAMP(raw_event:ts::INT)       AS event_ts
FROM raw_events_variant;

-- Now queries use native types → columnar I/O + partition pruning
```

### The right architecture

`VARIANT` is fine at the raw / bronze tier. Any table in your curated or serving layer that receives regular BI or analytics queries should have its hot fields extracted into typed columns. Keep the raw VARIANT if you need schema flexibility — but don't let it live in the hot query path.

---

## Part 1 Summary: Your Quick-Win Checklist

Work through this list in order — the earlier items deliver the highest return for the least effort:

| # | Technique | Effort | Impact | What to check |
|---|---|---|---|---|
| 1 | Result cache | ⚡ 5 min | HIGH | Search for `CURRENT_TIMESTAMP()`, `RANDOM()`, `UUID_STRING()` in repeated queries |
| 2 | Column pruning | ⚡ 5 min | HIGH | Search for `SELECT *` in CTEs and subqueries |
| 3 | Predicate pushdown | ⚡ 5 min | HIGH | Search for `DATE_TRUNC()`, `YEAR()`, `TO_CHAR()` in WHERE clauses |
| 4 | LIMIT + ORDER BY | ⚡ 5 min | MEDIUM | Check Top-N queries — do they have both clauses together? |
| 5 | QUALIFY | ⚡ 5 min | MEDIUM | Search for subquery patterns wrapping window functions |
| 6 | Clustering keys | 🔧 1 hour | HIGH | Run `SYSTEM$CLUSTERING_INFORMATION()` on large tables |
| 7 | Clustering depth | 🔧 1 hour | MEDIUM | Check `average_depth` — above 6 means degraded clustering |
| 8 | Search Optimization | 🔧 1 hour | HIGH | Look for high-scan, low-row-return queries in query history |
| 9 | Flatten VARIANT | 🏗️ 1 sprint | HIGH | Audit serving-layer tables for VARIANT columns in hot query paths |

---

## What's Coming in Part 2

Part 2 covers warehouse tuning and join strategies — where configuration choices and query structure determine whether you're spending 1x or 64x your minimum necessary compute:

- **Technique 10** — Right-sizing warehouses: larger is not always faster
- **Technique 11** — Multi-cluster warehouses for high concurrency
- **Technique 12** — Auto-suspend tuning to eliminate idle credit waste
- **Technique 13** — Workload isolation: separate warehouses by type
- **Technique 14** — Join order: large tables left, small tables right
- **Technique 15** — Semi-joins with EXISTS for existence checks
- **Technique 16** — Pre-aggregate before joining large tables

---

## Author's Note

I write weekly at the intersection of data engineering (Databricks, Snowflake, Delta Lake) and AI systems (LangChain, LangGraph, RAG, MCP). If this saved you some compute credits — or surfaced a pattern you recognise from your own codebase — follow, clap, or subscribe for Part 2. 🔔👏📩

---

*All performance figures are based on the author's production experience across multiple Snowflake deployments. Results will vary by workload, warehouse size, table size, and data distribution. Always validate optimizations in your own environment using Query Profile and `ACCOUNT_USAGE` views before applying at scale.*

*© 2026 Sudip P. All rights reserved.*
