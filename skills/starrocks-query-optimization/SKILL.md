---
name: starrocks-query-optimization
description: Optimize StarRocks queries — reading EXPLAIN/EXPLAIN VERBOSE, verifying partition pruning, choosing JOIN strategy (broadcast/shuffle/colocate), predicate pushdown, parallelism (pipeline_dop), index usage, and diagnosing slow queries via query profiles. Use when a StarRocks query is slow, when analyzing execution plans, or when tuning JOINs, pruning, or parallelism.
license: Apache-2.0
metadata:
    author: "Tiansu Yu"
    version: "1.0"
---

# StarRocks Query Optimization

Tune StarRocks query performance: execution plans, partition pruning, JOIN strategy, predicate pushdown, parallelism, index usage, and slow-query diagnosis.

## When to Use

- A StarRocks query is slow and needs diagnosis
- Reading and interpreting `EXPLAIN` / `EXPLAIN VERBOSE` / `EXPLAIN ANALYZE`
- Verifying partition pruning is active
- Choosing or hinting a JOIN strategy
- Tuning parallelism or index usage
- Analyzing a query profile to find the bottleneck

Related skills: [starrocks-table-design] for partitioning/bucketing/indexes that drive these plans; [starrocks-materialized-views] for pre-aggregation; [starrocks-monitoring] for processlist and killing queries.

## Understanding EXPLAIN

```sql
-- Basic explain (logical plan)
EXPLAIN SELECT * FROM table WHERE id = 123;

-- Verbose explain (with statistics)
EXPLAIN VERBOSE SELECT ...;

-- Cost-based explain
EXPLAIN COSTS SELECT ...;

-- Analyze actual execution
EXPLAIN ANALYZE SELECT ...;
```

**Key things to look for in EXPLAIN:**
- `partitions=X/Y` - Are partitions being pruned?
- `BROADCAST JOIN` vs `SHUFFLE JOIN` - Is join strategy optimal?
- `predicates` - Are filters being pushed down?
- `cardinality` - Are statistics accurate?

## Partition Pruning

**✅ Good - Partition pruning works:**
```sql
-- Partitioned by date_col
SELECT * FROM events
WHERE date_col >= '2024-01-01' AND date_col < '2024-01-07';
-- Result: partitions=7/365
```

**❌ Bad - No partition pruning:**
```sql
SELECT * FROM events
WHERE DATE(timestamp_col) = '2024-01-01';  -- Function on column
-- Result: partitions=365/365

-- Fix: materialize the date column
ALTER TABLE events ADD COLUMN date_col DATE AS DATE(timestamp_col);
-- Then partition by date_col
```

## JOIN Optimization

### 1. Broadcast JOIN (Small Table)

**Use when:** One table fits in memory (<1GB), broadcast to all nodes

```sql
-- Small dimension table (users) joined with large fact table (events)
SELECT e.*, u.username
FROM events e
INNER JOIN [broadcast] users u ON e.user_id = u.user_id
WHERE e.event_date = '2024-01-01';
```

StarRocks usually chooses broadcast automatically, but you can hint it.

### 2. Shuffle JOIN (Large Tables)

**Use when:** Both tables are large, redistribute data by join key

```sql
-- Both tables are large
SELECT o.*, i.product_name
FROM orders o
INNER JOIN [shuffle] order_items i ON o.order_id = i.order_id
WHERE o.order_date >= '2024-01-01';
```

### 3. Colocate JOIN (No Network)

**Use when:** Tables are in same colocate group, local join

```sql
-- Tables colocated on user_id
SELECT o.*, u.username
FROM orders o
INNER JOIN users u ON o.user_id = u.user_id;
-- No network shuffle needed
```

Set up colocate groups via table design — see [starrocks-table-design].

### 4. JOIN Order Optimization

**Put small table on right side for broadcast join:**

```sql
-- ✅ Good
SELECT *
FROM large_fact_table f
JOIN small_dimension_table d ON f.dim_id = d.id;

-- ❌ Bad (less efficient)
SELECT *
FROM small_dimension_table d
JOIN large_fact_table f ON d.id = f.dim_id;
```

## Predicate Pushdown

**✅ Good - Filters pushed to scan:**
```sql
SELECT user_id, SUM(amount) as total
FROM (
    SELECT user_id, amount
    FROM transactions
    WHERE date >= '2024-01-01'  -- Pushed to scan
)
GROUP BY user_id;
```

**❌ Bad - Filter after aggregation:**
```sql
SELECT user_id, total
FROM (
    SELECT user_id, SUM(amount) as total
    FROM transactions
    GROUP BY user_id
)
WHERE total > 1000;  -- Cannot push down
```

## Aggregation Optimization

**Use pre-aggregation when possible:**
```sql
-- If you frequently query daily sums, create aggregate table
CREATE TABLE daily_summary (
    date DATE,
    user_id BIGINT,
    total_amount DECIMAL(18, 2) SUM,
    event_count BIGINT SUM
)
AGGREGATE KEY (date, user_id)
PARTITION BY RANGE(date) ()
DISTRIBUTED BY HASH(user_id) BUCKETS 16;

-- Load with aggregation
INSERT INTO daily_summary
SELECT DATE(event_time), user_id, SUM(amount), COUNT(*)
FROM events
GROUP BY DATE(event_time), user_id;
```

For repeated aggregations across queries, prefer a materialized view — see [starrocks-materialized-views].

## Parallel Execution Control

```sql
-- Control query parallelism
SET pipeline_dop = 16;  -- Degree of parallelism (0 = auto)

-- For complex queries, adjust fragment instance
SET parallel_fragment_exec_instance_num = 8;

-- Enable adaptive DOP (StarRocks 3.0+)
SET enable_adaptive_sink_dop = true;
```

## Index Usage

**Bloom filter for point queries:**
```sql
-- Create bloom filter on email
ALTER TABLE users SET ("bloom_filter_columns" = "email");

-- Now this query is fast
SELECT * FROM users WHERE email = 'user@example.com';
```

**Bitmap index for categorical filters:**
```sql
-- Create bitmap index on status
ALTER TABLE orders
SET ("indexes" = "idx_status ON status USING BITMAP");

-- Fast filtering
SELECT COUNT(*) FROM orders WHERE status IN ('completed', 'shipped');
```

See [starrocks-table-design] for full index selection guidance.

## Slow Query Diagnosis

**Step 1: Get query profile**
```sql
-- Run query and get query_id
SELECT ...;

-- Show profile
SHOW QUERY PROFILE '<query_id>';

-- Or for current session
SHOW QUERY PROFILE;
```

**Step 2: Analyze bottlenecks**

Look for:
- High `ScanTime` → Add indexes or partition pruning
- High `NetworkTime` → Check JOIN strategy (use broadcast)
- High `AggregateTime` → Pre-aggregate or use materialized views
- Large `RowsReturned` → Add filters earlier

**Step 3: Check execution plan**
```sql
EXPLAIN VERBOSE SELECT ...;
```

Look for:
- `partitions=365/365` → Partition pruning not working
- `SHUFFLE JOIN` on small table → Should be broadcast
- Missing predicates → Filters not pushed down

### Quick triage checklist

1. Run `EXPLAIN VERBOSE` to check execution plan
2. Verify partition pruning is active
3. Check for data skew in distribution
4. Review JOIN strategy (broadcast vs shuffle)
5. Consider creating materialized views

## Common Slow Query Fixes

**Problem: Full table scan**
```sql
-- Bad
SELECT * FROM large_table WHERE DATE(timestamp_col) = '2024-01-01';

-- Good (partition pruning)
SELECT * FROM large_table WHERE date_col = '2024-01-01';
```

**Problem: Inefficient JOIN**
```sql
-- Bad: shuffle join on large tables
SELECT * FROM orders o JOIN shipments s ON o.order_id = s.order_id;

-- Good: colocate tables
-- Create with same colocate group
ALTER TABLE orders SET ("colocate_with" = "order_shipment_group");
ALTER TABLE shipments SET ("colocate_with" = "order_shipment_group");
```

**Problem: Repeated complex calculation**
```sql
-- Bad: calculate every time
SELECT user_id, SUM(amount * exchange_rate * (1 - discount)) as total
FROM transactions
GROUP BY user_id;

-- Good: pre-calculate in MV
CREATE MATERIALIZED VIEW mv_user_totals
REFRESH ASYNC EVERY(INTERVAL 1 HOUR)
AS
SELECT user_id, SUM(amount * exchange_rate * (1 - discount)) as total
FROM transactions
GROUP BY user_id;
```

## Query Anti-Patterns

### ❌ Function on Partition Column Breaks Pruning

```sql
-- Table partitioned by: PARTITION BY date_trunc('day', event_time)

-- ❌ BAD: Different function than partition expression
SELECT * FROM events
WHERE DATE(event_time) = '2024-01-01';
-- Results in: partitions=365/365 (full scan!)

-- ✅ GOOD: Use same expression or direct comparison
SELECT * FROM events
WHERE event_time >= '2024-01-01' AND event_time < '2024-01-02';
-- Results in: partitions=1/365

-- ✅ ALSO GOOD: Match the partition expression
SELECT * FROM events
WHERE date_trunc('day', event_time) = '2024-01-01';
-- Results in: partitions=1/365
```

### ❌ SELECT * in Production

```sql
-- BAD: Reads unnecessary data
SELECT * FROM large_table WHERE id = 123;

-- GOOD: Select only needed columns
SELECT id, name, amount FROM large_table WHERE id = 123;
```

### ❌ Nested Subqueries

```sql
-- BAD: Multiple nested subqueries
SELECT a FROM (
    SELECT b as a FROM (
        SELECT c as b FROM (
            SELECT d as c FROM table
        )
    )
) WHERE a > 100;

-- GOOD: Flatten or use CTE
WITH base AS (
    SELECT d as value FROM table
)
SELECT value FROM base WHERE value > 100;
```

## Performance Checklist

Before deploying queries to production:

- [ ] Queries filter on partition column (verify pruning with EXPLAIN)
- [ ] Partition granularity matches query patterns (hour/day/month)
- [ ] Frequently joined tables are colocated
- [ ] JOIN order puts small table on the right (broadcast)
- [ ] Filters are pushed down (not applied after aggregation)
- [ ] Materialized views used for repeated aggregations
- [ ] Bloom filter on high-cardinality string equality filters
- [ ] Bitmap index on low-cardinality filter columns
- [ ] `pipeline_dop` tuned for heavy queries (0 = auto)
- [ ] Statistics collected for accurate CBO (`ANALYZE TABLE`)
