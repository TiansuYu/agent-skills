---
name: starrocks-data-loading
description: Load data into StarRocks using Stream Load (HTTP/real-time), Broker Load (batch from S3/HDFS), Routine Load (Kafka continuous), and INSERT INTO. Use when ingesting data into StarRocks, building load pipelines, mapping/transforming columns on load, checking load status, or troubleshooting data load failures.
license: Apache-2.0
metadata:
    author: "Tiansu Yu"
    version: "1.0"
---

# StarRocks Data Loading

Load data into StarRocks tables via the four loading methods, with column mapping, status checks, and failure troubleshooting.

## When to Use

- Ingesting data from applications, object storage, or Kafka
- Choosing a loading method (Stream / Broker / Routine Load / INSERT)
- Mapping or transforming columns during load
- Checking load job status or canceling loads
- Diagnosing data load failures

**Method selection:**
- **Stream Load** — real-time, small/medium batches (<10GB), direct from applications
- **Broker Load** — large batch (>10GB) from S3/HDFS, scheduled ETL
- **Routine Load** — continuous streaming from Kafka
- **INSERT INTO** — small loads and ETL transformations within StarRocks

Related skills: [starrocks-table-design] for the target schema; [starrocks-etl-and-tasks] for ETL pipelines and async/scheduled execution.

## 1. Stream Load (HTTP-based, Real-time)

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

Stream Load returns its result synchronously in the HTTP response JSON. It is **not** queryable via `SHOW LOAD` (that covers Broker Load, INSERT, and Spark Load only). Use the response, or the `get_load_state` endpoint:
```bash
curl --location-trusted -u user:password \
    http://fe_host:8030/api/database/get_load_state?label=load_label_001
```

## 2. Broker Load (Batch from Object Storage)

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

## 3. Routine Load (Kafka Continuous)

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

## 4. INSERT INTO (Direct SQL)

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

> **Version/cluster caveats:** Explicit SQL transactions require **v3.5+** and support **INSERT only** on shared-nothing clusters. `UPDATE`/`DELETE` inside a transaction need a **shared-data cluster on v4.0+**, must target a **Primary Key table**, and must appear **before** any INSERT on the same table (one UPDATE/DELETE per table).
```sql
BEGIN;

-- DELETE/UPDATE must come before INSERT on the same table (PK table, shared-data v4.0+)
DELETE FROM staging_pk_table WHERE load_date < '2024-01-01';

INSERT INTO staging_pk_table (id, value)
SELECT id, value FROM source_table WHERE status = 'active';

COMMIT;
```

## Troubleshooting Data Load Failures

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

**Quick checks when a load fails:**
1. Check error message: `SHOW LOAD WHERE label = 'your_label'`
2. Verify data format matches column definitions
3. Check broker connectivity (for Broker Load)
4. Verify resource availability: `SHOW PROC '/backends'`

## Anti-Patterns to Avoid

❌ **No load labels** — always set a label so loads are trackable and duplicate-safe:
```sql
-- BAD: No label, cannot track or prevent duplicates
curl ... -T data.csv http://fe:8030/api/db/table/_stream_load

-- GOOD: With label
curl ... -H "label:load_20240101_001" -T data.csv http://...
```
