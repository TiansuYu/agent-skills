---
name: starrocks-monitoring
description: Monitor and maintain a StarRocks cluster — health/metrics (backends, processlist, killing queries), compaction management, tablet repair/rebalancing, routine maintenance schedules, statistics collection for CBO, and troubleshooting commands. Use when checking StarRocks cluster health, diagnosing compaction or tablet issues, killing runaway queries, collecting statistics, or planning routine maintenance.
license: Apache-2.0
metadata:
    author: "Tiansu Yu"
    version: "1.0"
---

# StarRocks Monitoring and Maintenance

Observe and maintain a running StarRocks cluster: health metrics, compaction, tablet repair, statistics, and routine upkeep.

## When to Use

- Checking cluster/BE health, disk usage, or tablet distribution
- Inspecting running queries and killing runaway ones
- Diagnosing or triggering compaction
- Repairing or rebalancing tablets/replicas
- Collecting statistics for the cost-based optimizer (CBO)
- Planning daily/weekly/monthly maintenance

Related skills: [starrocks-cluster-setup] for node topology and config; [starrocks-resource-management] for workload isolation; [starrocks-backup-recovery] for snapshots; [starrocks-query-optimization] for query-level diagnosis.

## Key Metrics to Monitor

**Cluster Health:**
```sql
-- Backend status and load
SHOW PROC '/backends';

-- Disk usage per BE
SHOW PROC '/statistic';

-- Tablet distribution
SHOW PROC '/statistic/{be_id}';
```

**Query Performance:**
```sql
-- Recent queries
SELECT * FROM information_schema.loads ORDER BY StartTime DESC LIMIT 10;

-- Running queries
SHOW PROCESSLIST;

-- Kill slow query
KILL QUERY connection_id;

-- Query profile (after execution)
SHOW QUERY PROFILE '<query_id>';
```

**Data Loading:**
```sql
-- Recent load jobs (Broker Load / INSERT / Spark Load)
SHOW LOAD ORDER BY CreateTime DESC LIMIT 10;

-- Failed loads
SHOW LOAD WHERE State = 'CANCELLED';

-- Stream Load / Routine Load are not shown by SHOW LOAD; query information_schema instead
SELECT * FROM information_schema.loads ORDER BY create_time DESC LIMIT 10;
```

## Compaction Management

```sql
-- Check compaction status
SHOW PROC '/compactions';

-- Trigger manual compaction (v3.1+): whole table, a partition, or cumulative
ALTER TABLE table_name COMPACT;
ALTER TABLE table_name COMPACT partition_name;
ALTER TABLE table_name CUMULATIVE COMPACT;

-- Compaction thresholds are BE configuration, NOT table properties.
-- Tune them in be.conf or dynamically via the be_configs table, e.g.:
UPDATE information_schema.be_configs
SET value = "100"
WHERE name = "max_cumulative_compaction_num_singleton_deltas";
```

## Tablet Repair and Rebalancing

```sql
-- Check tablet health
ADMIN SHOW REPLICA STATUS FROM table_name;

-- Check tablet distribution
SHOW TABLET FROM table_name;

-- Trigger tablet repair (if replicas are corrupted)
ADMIN REPAIR TABLE table_name;

-- Manual tablet rebalance (usually automatic)
ADMIN REBALANCE DISK;
```

## Routine Maintenance Tasks

**Daily:**
- Monitor load job failures
- Check query latencies
- Review disk usage

**Weekly:**
- Analyze slow queries
- Review partition retention
- Check compaction lag

**Monthly:**
- Review and optimize bucket counts
- Analyze data skew
- Update statistics (if using CBO)

## Statistics Collection (for CBO)

```sql
-- Collect statistics for better query planning (manual, synchronous by default)
ANALYZE TABLE table_name;

-- Collect for specific columns
ANALYZE TABLE table_name (col1, col2);

-- Automatic full collection is ON by default (no per-table property needed);
-- it is governed by FE config (e.g. enable_collect_full_statistic,
-- statistic_auto_analyze_start_time / _end_time). Define a recurring custom
-- collection job with CREATE ANALYZE if you need finer control.

-- Check collection status and statistics metadata
SHOW ANALYZE STATUS;
SHOW META;
```

## Troubleshooting Commands

```sql
-- Check FE logs
ADMIN SHOW FRONTEND CONFIG LIKE 'log%';

-- Show system variables
SHOW VARIABLES;

-- Show session variables
SHOW SESSION VARIABLES;

-- Check metadata inconsistencies
ADMIN CHECK TABLET (tablet_id);

-- Show query plan and profile
EXPLAIN VERBOSE SELECT ...;
EXPLAIN COSTS SELECT ...;
```
