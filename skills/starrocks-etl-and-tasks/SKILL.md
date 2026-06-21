---
name: starrocks-etl-and-tasks
description: Build StarRocks ETL pipelines and async/scheduled tasks — daily batch loads, incremental load with deduplication, real-time aggregation pipelines, SCD Type 2, and SUBMIT TASK for background/scheduled INSERT/CTAS execution with monitoring. Use when designing StarRocks ETL workflows, running long INSERT...SELECT in the background, scheduling recurring transformations, or backfilling data.
license: Apache-2.0
metadata:
    author: "Tiansu Yu"
    version: "1.0"
---

# StarRocks ETL and Tasks

Design ETL pipelines in StarRocks and run them asynchronously or on a schedule with `SUBMIT TASK`.

## When to Use

- Designing batch, incremental, or real-time ETL pipelines
- Running long INSERT/CTAS in the background (avoid client timeout)
- Scheduling recurring transformations natively in StarRocks
- Backfilling or migrating data
- Implementing SCD Type 2 dimensions

Related skills: [starrocks-data-loading] for the underlying load methods; [starrocks-materialized-views] as an alternative to scheduled aggregation tasks; [starrocks-table-design] for staging/target schemas.

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

## SUBMIT TASK (Async Execution)

`SUBMIT TASK` runs ETL statements asynchronously in the background. Available since v2.5, with scheduled execution since v3.3.

### When to Use SUBMIT TASK

| Scenario | Why SUBMIT TASK | Alternative |
|----------|----------------|-------------|
| Long-running INSERT INTO ... SELECT | Avoids client timeout; runs in background | Synchronous INSERT (blocks session) |
| Scheduled recurring ETL (e.g. hourly aggregation) | Built-in scheduler with EVERY interval | External scheduler (Airflow, cron) |
| CTAS for large result sets | Prevents session timeout on big datasets | Manual CREATE + INSERT |
| INSERT OVERWRITE partition refresh | Fire-and-forget partition rebuilds | Synchronous OVERWRITE (blocks) |
| Backfill / data migration | Submit multiple tasks and monitor via metadata | Sequential INSERT statements |

**Use SUBMIT TASK when:**
- The operation would exceed your session's `query_timeout`
- You want StarRocks-native scheduling without external orchestrators
- You need to fire-and-forget and check status later
- You're running multiple independent ETL steps in parallel

**Don't use SUBMIT TASK when:**
- You need the result immediately in the same session
- The operation finishes in seconds (overhead not worth it)
- You already have Airflow/dbt orchestrating the pipeline (use those instead)

### Syntax

```sql
SUBMIT TASK <task_name>
[SCHEDULE [START(<schedule_start>)] EVERY(INTERVAL <schedule_interval>)]
[PROPERTIES("key" = "value" [, ...])]
AS <etl_statement>
```

Supported ETL statements:
- `CREATE TABLE AS SELECT` (v3.0+)
- `INSERT INTO / INSERT OVERWRITE` (v3.0+)
- `CACHE SELECT` (v3.3+)

### One-off Background Task

```sql
-- Background INSERT that won't block your session
SUBMIT TASK backfill_2024
AS
INSERT INTO target_table
SELECT * FROM source_table
WHERE dt >= '2024-01-01' AND dt < '2025-01-01';
```

### Scheduled Recurring Task

```sql
-- Refresh aggregation table every 5 minutes
SUBMIT TASK refresh_5min_agg
SCHEDULE EVERY(INTERVAL 5 MINUTE)
AS
INSERT OVERWRITE hourly_metrics
SELECT
    date_trunc('hour', event_time) AS hour,
    event_type,
    COUNT(*) AS cnt,
    SUM(amount) AS total
FROM raw_events
WHERE event_time >= DATE_SUB(NOW(), INTERVAL 2 HOUR)
GROUP BY date_trunc('hour', event_time), event_type;
```

### Scheduled Task with Start Time

```sql
-- Start at midnight, run daily
SUBMIT TASK daily_summary
SCHEDULE START('2024-01-01 00:00:00') EVERY(INTERVAL 1 DAY)
AS
INSERT OVERWRITE daily_revenue PARTITION (dt)
SELECT
    DATE(order_time) AS dt,
    store_id,
    SUM(amount) AS revenue
FROM orders
WHERE order_time >= DATE_SUB(CURDATE(), INTERVAL 1 DAY)
  AND order_time < CURDATE()
GROUP BY DATE(order_time), store_id;
```

### Task with Session Properties

```sql
-- Extend timeout and enable profiling for a heavy task
SUBMIT TASK heavy_etl
PROPERTIES(
    "session.insert_timeout" = "36000",
    "session.enable_profile" = "true"
)
AS
INSERT INTO warehouse.fact_orders
SELECT * FROM staging.raw_orders;
```

### CTAS as Background Task

```sql
-- Create a snapshot table without blocking
SUBMIT TASK create_snapshot
AS
CREATE TABLE snapshot_20240101
AS SELECT * FROM production_table
WHERE dt = '2024-01-01';
```

### Monitoring Tasks

```sql
-- List all registered tasks
SELECT task_name, `schedule`, create_time, definition
FROM information_schema.tasks;

-- Check execution history for a specific task
SELECT task_name, state, create_time, finish_time, error_message
FROM information_schema.task_runs
WHERE task_name = 'daily_summary'
ORDER BY create_time DESC
LIMIT 10;

-- Find failed task runs
SELECT task_name, state, error_message, create_time
FROM information_schema.task_runs
WHERE state = 'FAILED'
ORDER BY create_time DESC;
```

### Drop a Task

```sql
-- Stop and remove a scheduled task
DROP TASK daily_summary;
```

### FE Configuration Tuning

| Parameter | Default | Description |
|-----------|---------|-------------|
| `task_runs_concurrency` | 4 | Max parallel TaskRuns |
| `task_runs_queue_length` | 500 | Max pending TaskRuns |
| `task_runs_ttl_second` | 86400 | TTL for TaskRun records (seconds) |
| `task_ttl_second` | 86400 | TTL for one-off Task templates (seconds) |
| `task_min_schedule_interval_s` | 10 | Minimum schedule interval (seconds) |
