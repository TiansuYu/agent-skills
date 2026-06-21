---
name: starrocks-materialized-views
description: Create and manage StarRocks materialized views — synchronous rollups (auto-maintained), asynchronous MVs (scheduled refresh, joins, partitioned), transparent query rewrite, and MV refresh/management. Use when pre-aggregating data in StarRocks, speeding up repeated aggregations or joins, setting up scheduled MV refresh, or enabling automatic query rewrite.
license: Apache-2.0
metadata:
    author: "Tiansu Yu"
    version: "1.0"
---

# StarRocks Materialized Views

Pre-aggregate and accelerate StarRocks queries with synchronous rollups and asynchronous materialized views, including transparent query rewrite.

## When to Use

- Repeated aggregations or joins that are expensive to recompute
- Building scheduled, refreshed summary tables
- Enabling transparent query rewrite so existing queries auto-use the MV
- Managing MV refresh and lifecycle

**Type selection:**
- **Synchronous Rollup** — simple pre-aggregations, automatically maintained
- **Asynchronous MV** — complex transformations, joins, scheduled refresh

Related skills: [starrocks-query-optimization] for when to reach for an MV vs other tuning; [starrocks-etl-and-tasks] for SUBMIT TASK-based refreshes.

## Synchronous Rollup (Automatic)

**Use for:** Simple pre-aggregations, automatically maintained

```sql
-- Create rollup (automatically synced)
CREATE MATERIALIZED VIEW order_daily_rollup AS
SELECT
    order_date,
    store_id,
    SUM(amount) as total_amount,
    COUNT(*) as order_count
FROM orders
GROUP BY order_date, store_id;

-- Query automatically uses rollup when applicable
SELECT order_date, SUM(amount)
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY order_date;
-- Automatically uses order_daily_rollup if beneficial
```

## Asynchronous Materialized View (Scheduled)

**Use for:** Complex transformations, joins, scheduled refresh

**Create async MV:**
```sql
CREATE MATERIALIZED VIEW mv_user_metrics
REFRESH ASYNC START('2024-01-01 00:00:00') EVERY(INTERVAL 1 HOUR)
AS
SELECT
    u.user_id,
    u.username,
    u.region,
    COUNT(DISTINCT o.order_id) as total_orders,
    SUM(o.amount) as total_spent,
    MAX(o.order_date) as last_order_date,
    COUNT(DISTINCT DATE(o.order_date)) as active_days
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id, u.username, u.region;
```

**Query async MV directly:**
```sql
-- Query the MV like a table
SELECT *
FROM mv_user_metrics
WHERE region = 'US'
  AND total_orders > 10
ORDER BY total_spent DESC
LIMIT 100;
```

**Async MV with partition:**
```sql
CREATE MATERIALIZED VIEW mv_daily_revenue
PARTITION BY date
REFRESH ASYNC START('2024-01-01 00:00:00') EVERY(INTERVAL 1 DAY)
AS
SELECT
    order_date as date,
    store_id,
    SUM(amount) as revenue,
    COUNT(*) as order_count
FROM orders
GROUP BY order_date, store_id;
```

**Manage materialized views:**
```sql
-- Show MVs
SHOW MATERIALIZED VIEWS FROM database_name;

-- Refresh manually
REFRESH MATERIALIZED VIEW mv_user_metrics;

-- Refresh a partition range (by partition-column value, not partition name)
REFRESH MATERIALIZED VIEW mv_daily_revenue PARTITION START ("2024-01-01") END ("2024-01-02");

-- Drop MV
DROP MATERIALIZED VIEW mv_user_metrics;

-- Check MV status
SELECT * FROM information_schema.materialized_views
WHERE table_name = 'mv_user_metrics';
```

## MV Query Rewrite (Transparent Optimization)

StarRocks can automatically rewrite queries to use MVs:

```sql
-- Original query
SELECT order_date, store_id, SUM(amount)
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY order_date, store_id;

-- If mv_daily_revenue exists, StarRocks automatically uses it
-- No query change needed!
```

For async MVs on the default catalog, query rewrite is **enabled by default** (`enable_materialized_view_rewrite` and `enable_materialized_view_union_rewrite` both default to `true` since v2.5). You'd only set these to re-enable after disabling:
```sql
SET enable_materialized_view_rewrite = true;
SET enable_materialized_view_union_rewrite = true;
```
