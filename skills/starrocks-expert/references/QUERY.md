# StarRocks Query Optimization and Task Skills

This guide covers data loading patterns, query optimization, materialized views, and ETL pipeline best practices.

## Table of Contents
- [Data Loading Patterns](#data-loading-patterns)
- [Query Optimization](#query-optimization)
- [Materialized Views](#materialized-views)
- [ETL Pipeline Patterns](#etl-pipeline-patterns)
- [Performance Troubleshooting](#performance-troubleshooting)
- [Common Anti-Patterns](#common-anti-patterns)

---

## Data Loading Patterns

### 1. Stream Load (HTTP-based, Real-time)

**Use for:** Real-time ingestion, small to medium batches (<10GB), direct from applications

**Basic Stream Load:**
```bash
curl --location-trusted -u user:password \
    -H "label:load_$(date +%Y%m%d_%H%M%S)" \
    -H "column_separator:," \
    -T data.csv \
    http://fe_host:8030/api/database/table/_stream_load
```

**Stream Load with Column Mapping:**
```bash
curl --location-trusted -u user:password \
    -H "label:load_label_001" \
    -H "column_separator:," \
    -H "columns: col1, col2, col3, col4=col1*100, col5=now()" \
    -H "where: col1 > 0" \
    -T data.csv \
    http://fe_host:8030/api/database/table/_stream_load
```

**Stream Load JSON:**
```bash
curl --location-trusted -u user:password \
    -H "label:json_load_$(date +%s)" \
    -H "format:json" \
    -H "jsonpaths:[\"$.id\", \"$.name\", \"$.amount\"]" \
    -H "strip_outer_array:true" \
    -T data.json \
    http://fe_host:8030/api/database/table/_stream_load
```

**Stream Load Parquet:**
```bash
curl --location-trusted -u user:password \
    -H "label:parquet_load_001" \
    -H "format:parquet" \
    -T data.parquet \
    http://fe_host:8030/api/database/table/_stream_load
```

**Check Stream Load Status:**
```bash
# Get result from response
curl --location-trusted -u user:password \
    http://fe_host:8030/api/database/get_load_state?label=load_label_001

# Or query in SQL
SHOW LOAD WHERE label = 'load_label_001';
```

### 2. Broker Load (Batch from Object Storage)

**Use for:** Large batch loads (>10GB), data from S3/HDFS, scheduled ETL

**Load from S3 (Parquet):**
```sql
LOAD LABEL mydb.load_20240101_s3
(
    DATA INFILE("s3://bucket/path/data_*.parquet")
    INTO TABLE target_table
    FORMAT AS "parquet"
    (col1, col2, col3)
    SET (
        col4 = col1 * 100,
        col5 = FROM_UNIXTIME(col3)
    )
)
WITH BROKER
(
    "aws.s3.access_key" = "ACCESS_KEY",
    "aws.s3.secret_key" = "SECRET_KEY",
    "aws.s3.region" = "us-east-1",
    "aws.s3.use_instance_profile" = "false"
)
PROPERTIES (
    "timeout" = "3600",
    "max_filter_ratio" = "0.1",
    "strict_mode" = "false"
);
```

**Load from S3 (CSV with Partition):**
```sql
LOAD LABEL mydb.load_partition_20240101
(
    DATA INFILE("s3://bucket/year=2024/month=01/day=01/*.csv")
    INTO TABLE events
    COLUMNS TERMINATED BY ","
    FORMAT AS "csv"
    (event_id, user_id, event_type, created_at)
    PARTITION (p20240101)
)
WITH BROKER
(
    "aws.s3.access_key" = "ACCESS_KEY",
    "aws.s3.secret_key" = "SECRET_KEY",
    "aws.s3.region" = "us-east-1"
)
PROPERTIES (
    "timeout" = "7200"
);
```

**Check Broker Load Status:**
```sql
-- Check load job status
SHOW LOAD WHERE label = 'load_20240101_s3';

-- Cancel if needed
CANCEL LOAD FROM mydb WHERE label = 'load_20240101_s3';
```

### 3. Routine Load (Kafka Continuous)

**Use for:** Continuous streaming from Kafka, real-time pipelines

**Create Routine Load:**
```sql
CREATE ROUTINE LOAD mydb.routine_load_events ON events
COLUMNS(event_id, user_id, event_type, event_time, properties)
PROPERTIES (
    "desired_concurrent_number" = "3",
    "max_batch_interval" = "20",
    "max_batch_rows" = "250000",
    "format" = "json",
    "jsonpaths" = "[\"$.event_id\",\"$.user_id\",\"$.type\",\"$.timestamp\",\"$.data\"]"
)
FROM KAFKA (
    "kafka_broker_list" = "broker1:9092,broker2:9092,broker3:9092",
    "kafka_topic" = "events_topic",
    "kafka_partitions" = "0,1,2,3,4,5,6,7",
    "property.group.id" = "starrocks_consumer_group",
    "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);
```

**Manage Routine Load:**
```sql
-- Show routine load jobs
SHOW ROUTINE LOAD FOR mydb.routine_load_events;

-- Pause routine load
PAUSE ROUTINE LOAD FOR mydb.routine_load_events;

-- Resume routine load
RESUME ROUTINE LOAD FOR mydb.routine_load_events;

-- Stop routine load (cannot resume)
STOP ROUTINE LOAD FOR mydb.routine_load_events;

-- Show error details
SHOW ROUTINE LOAD TASK WHERE JobName = 'routine_load_events';
```

### 4. INSERT INTO (Direct SQL)

**Use for:** Small data loads, ETL transformations within StarRocks

**Basic INSERT:**
```sql
INSERT INTO target_table (col1, col2, col3)
VALUES (1, 'value1', 100.5),
       (2, 'value2', 200.3);
```

**INSERT from SELECT (ETL):**
```sql
INSERT INTO target_table (date, user_id, total_amount, event_count)
SELECT
    DATE(event_time) as date,
    user_id,
    SUM(amount) as total_amount,
    COUNT(*) as event_count
FROM source_table
WHERE event_time >= '2024-01-01'
GROUP BY DATE(event_time), user_id;
```

**INSERT OVERWRITE (Replace partition):**
```sql
INSERT OVERWRITE target_table PARTITION (p20240101)
SELECT col1, col2, col3
FROM source_table
WHERE date_col = '2024-01-01';
```

**Multi-statement Transaction:**
```sql
BEGIN;

DELETE FROM staging_table WHERE load_date < '2024-01-01';

INSERT INTO staging_table (id, value)
SELECT id, value FROM source_table WHERE status = 'active';

UPDATE metrics_table
SET last_updated = NOW()
WHERE table_name = 'staging_table';

COMMIT;
```

---

## Query Optimization

### Understanding EXPLAIN

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

### Partition Pruning

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

### JOIN Optimization

#### 1. Broadcast JOIN (Small Table)

**Use when:** One table fits in memory (<1GB), broadcast to all nodes

```sql
-- Small dimension table (users) joined with large fact table (events)
SELECT e.*, u.username
FROM events e
INNER JOIN [broadcast] users u ON e.user_id = u.user_id
WHERE e.event_date = '2024-01-01';
```

StarRocks usually chooses broadcast automatically, but you can hint it.

#### 2. Shuffle JOIN (Large Tables)

**Use when:** Both tables are large, redistribute data by join key

```sql
-- Both tables are large
SELECT o.*, i.product_name
FROM orders o
INNER JOIN [shuffle] order_items i ON o.order_id = i.order_id
WHERE o.order_date >= '2024-01-01';
```

#### 3. Colocate JOIN (No Network)

**Use when:** Tables are in same colocate group, local join

```sql
-- Tables colocated on user_id
SELECT o.*, u.username
FROM orders o
INNER JOIN users u ON o.user_id = u.user_id;
-- No network shuffle needed
```

#### 4. JOIN Order Optimization

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

### Predicate Pushdown

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

### Aggregation Optimization

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

### Parallel Execution Control

```sql
-- Control query parallelism
SET pipeline_dop = 16;  -- Degree of parallelism (0 = auto)

-- For complex queries, adjust fragment instance
SET parallel_fragment_exec_instance_num = 8;

-- Enable adaptive DOP (StarRocks 3.0+)
SET enable_adaptive_sink_dop = true;
```

### Index Usage

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

---

## Materialized Views

### Synchronous Rollup (Automatic)

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

### Asynchronous Materialized View (Scheduled)

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

-- Refresh specific partitions
REFRESH MATERIALIZED VIEW mv_daily_revenue PARTITION (p20240101, p20240102);

-- Drop MV
DROP MATERIALIZED VIEW mv_user_metrics;

-- Check MV status
SELECT * FROM information_schema.materialized_views
WHERE table_name = 'mv_user_metrics';
```

### MV Query Rewrite (Transparent Optimization)

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

**Enable query rewrite:**
```sql
SET enable_materialized_view_rewrite = true;
SET enable_materialized_view_union_rewrite = true;
```

---

## ETL Pipeline Patterns

### Pattern 1: Daily Batch Load from S3

```sql
-- Step 1: Load raw data to staging table
LOAD LABEL mydb.load_staging_{{ ds }}
(
    DATA INFILE("s3://bucket/data/{{ ds }}/*.parquet")
    INTO TABLE staging_events
    FORMAT AS "parquet"
)
WITH BROKER
(
    "aws.s3.access_key" = "KEY",
    "aws.s3.secret_key" = "SECRET",
    "aws.s3.region" = "us-east-1"
)
PROPERTIES (
    "timeout" = "7200"
);

-- Step 2: Wait for load to complete (check status)

-- Step 3: Transform and load to final table
INSERT INTO events PARTITION (p{{ ds_nodash }})
SELECT
    event_id,
    user_id,
    event_type,
    DATE(event_time) as event_date,
    event_time,
    CAST(properties['amount'] AS DECIMAL(18,2)) as amount
FROM staging_events
WHERE DATE(event_time) = '{{ ds }}';

-- Step 4: Clean up staging
TRUNCATE TABLE staging_events;
```

### Pattern 2: Incremental Load with Deduplication

```sql
-- Primary key table for automatic deduplication
CREATE TABLE user_profiles (
    user_id BIGINT NOT NULL,
    username VARCHAR(100),
    email VARCHAR(200),
    status VARCHAR(20),
    updated_at DATETIME
)
PRIMARY KEY (user_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 16
PROPERTIES (
    "enable_persistent_index" = "true"
);

-- Stream load with updates (automatic upsert)
curl --location-trusted -u user:password \
    -H "label:user_profile_update_$(date +%s)" \
    -H "format:json" \
    -T incremental_users.json \
    http://fe_host:8030/api/mydb/user_profiles/_stream_load

-- Only latest record per user_id is kept
```

### Pattern 3: Real-time Aggregation Pipeline

```sql
-- Step 1: Raw events table with hourly partitioning
CREATE TABLE raw_events (
    event_time DATETIME,
    event_type STRING,
    user_id BIGINT,
    processing_time INT
)
DUPLICATE KEY (event_time)
PARTITION BY date_trunc('hour', event_time)  -- Hourly for high frequency
DISTRIBUTED BY HASH(user_id) BUCKETS 64;

-- Step 2: Continuous loading via Routine Load from Kafka
CREATE ROUTINE LOAD mydb.routine_load_events ON raw_events
COLUMNS(event_time, event_type, user_id, processing_time)
PROPERTIES (
    "desired_concurrent_number" = "3",
    "max_batch_interval" = "20",
    "format" = "json"
)
FROM KAFKA (...);

-- Step 3: Async MV for 5-minute aggregation
CREATE MATERIALIZED VIEW mv_events_5min
REFRESH ASYNC START('2024-01-01 00:00:00') EVERY(INTERVAL 5 MINUTE)
AS
SELECT
    date_trunc('minute', event_time) as time_bucket,  -- Minute-level bucketing
    event_type,
    COUNT(*) as event_count,
    COUNT(DISTINCT user_id) as unique_users,
    AVG(processing_time) as avg_processing_time
FROM raw_events
WHERE event_time >= DATE_SUB(NOW(), INTERVAL 1 DAY)
GROUP BY date_trunc('minute', event_time), event_type;

-- Step 4: Query the MV for dashboard
SELECT * FROM mv_events_5min
WHERE time_bucket >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
ORDER BY time_bucket DESC;
```

### Pattern 4: SCD Type 2 (Slowly Changing Dimension)

```sql
-- Historical dimension table
CREATE TABLE dim_users_history (
    user_id BIGINT NOT NULL,
    username VARCHAR(100),
    email VARCHAR(200),
    status VARCHAR(20),
    valid_from DATETIME NOT NULL,
    valid_to DATETIME,
    is_current BOOLEAN
)
DUPLICATE KEY (user_id, valid_from)
DISTRIBUTED BY HASH(user_id) BUCKETS 16;

-- ETL to handle updates
BEGIN;

-- Step 1: Close existing records for updated users
UPDATE dim_users_history
SET valid_to = NOW(), is_current = false
WHERE user_id IN (SELECT user_id FROM staging_users)
  AND is_current = true;

-- Step 2: Insert new records
INSERT INTO dim_users_history
SELECT
    user_id,
    username,
    email,
    status,
    NOW() as valid_from,
    NULL as valid_to,
    true as is_current
FROM staging_users;

COMMIT;
```

---

## Performance Troubleshooting

### Slow Query Diagnosis

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

### Common Slow Query Fixes

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

### Data Load Failures

**Check error message:**
```sql
SHOW LOAD WHERE label = 'load_label' \G
-- Look at ErrorMsg field
```

**Common errors:**

| Error | Cause | Fix |
|-------|-------|-----|
| "too many filtered rows" | Data quality issues | Check `max_filter_ratio` |
| "timeout" | Large data, slow network | Increase `timeout` property |
| "replica not enough" | BE down | Check cluster status |
| "column not match" | Schema mismatch | Verify column mapping |

**Retry failed load:**
```sql
-- Check which files failed
SHOW LOAD WHERE label = 'load_label' \G

-- Resubmit with same label (idempotent)
LOAD LABEL mydb.load_label ...
```

---

## Common Anti-Patterns

### ❌ Anti-Pattern 1: Function on Partition Column Breaks Pruning

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

### ❌ Anti-Pattern 2: SELECT * in Production

```sql
-- BAD: Reads unnecessary data
SELECT * FROM large_table WHERE id = 123;

-- GOOD: Select only needed columns
SELECT id, name, amount FROM large_table WHERE id = 123;
```

### ❌ Anti-Pattern 3: Too Many Buckets

```sql
-- BAD: 10GB table with 256 buckets (40MB per bucket)
DISTRIBUTED BY HASH(id) BUCKETS 256;

-- GOOD: 10GB table with 8 buckets (~1.25GB per bucket)
DISTRIBUTED BY HASH(id) BUCKETS 8;
```

### ❌ Anti-Pattern 4: No Load Labels

```sql
-- BAD: No label, cannot track or prevent duplicates
curl ... -T data.csv http://fe:8030/api/db/table/_stream_load

-- GOOD: With label
curl ... -H "label:load_20240101_001" -T data.csv http://...
```

### ❌ Anti-Pattern 5: Wrong Time Column Type

```sql
-- ❌ BAD: Using VARCHAR for timestamps
CREATE TABLE events (
    event_date_str VARCHAR(20),  -- "2024-01-01 10:30:00"
    user_id BIGINT
)
DUPLICATE KEY (event_date_str)
PARTITION BY event_date_str  -- Cannot use date_trunc()!
DISTRIBUTED BY HASH(user_id) BUCKETS 16;

-- ✅ GOOD: Use proper DATETIME type
CREATE TABLE events (
    event_time DATETIME,
    user_id BIGINT
)
DUPLICATE KEY (event_time)
PARTITION BY date_trunc('day', event_time)
DISTRIBUTED BY HASH(user_id) BUCKETS 16;
```

### ❌ Anti-Pattern 6: Nested Subqueries

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

### ❌ Anti-Pattern 7: UNIQUE KEY for Deduplication Only

```sql
-- BAD: Using UNIQUE KEY just for deduplication
CREATE TABLE events (
    event_id BIGINT,
    event_time DATETIME,
    user_id BIGINT
)
UNIQUE KEY (event_id)  -- Overhead of maintaining uniqueness
...

-- GOOD: Use DUPLICATE KEY + GROUP BY in query
CREATE TABLE events (
    event_id BIGINT,
    event_time DATETIME,
    user_id BIGINT
)
DUPLICATE KEY (event_id, event_time)  -- No overhead
...

-- Deduplicate in query when needed
SELECT event_id, MAX(event_time)
FROM events
GROUP BY event_id;
```

### ❌ Anti-Pattern 8: Overusing Bloom Filters

```sql
-- BAD: Bloom filter on every column
ALTER TABLE events
SET ("bloom_filter_columns" = "id,date,type,status,region,country,city");
-- Wastes memory, slows down writes

-- GOOD: Only on high-cardinality columns with equality filters
ALTER TABLE events
SET ("bloom_filter_columns" = "id,email");
```

## Performance Checklist

Before deploying to production:

- [ ] Partitioning uses `date_trunc()` expression on DATETIME column
- [ ] Partition granularity matches query patterns (hour/day/month)
- [ ] Composite partitions kept under 100k total (tenants × time periods)
- [ ] Bucket count follows formula: (partition_size_GB / 2GB) × BE_count
- [ ] Hash key has high cardinality and even distribution
- [ ] Bloom filter only on high-cardinality string columns
- [ ] Bitmap index on low-cardinality filter columns
- [ ] Queries filter on partition column (verify pruning with EXPLAIN)
- [ ] Frequently joined tables are colocated
- [ ] Materialized views for repeated aggregations
- [ ] Load jobs use unique labels
- [ ] Partition TTL configured for automatic cleanup
- [ ] Resource groups configured for workload isolation
- [ ] Monitoring set up for load failures and slow queries
