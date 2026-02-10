# StarRocks Infrastructure Configuration and Management

This guide covers cluster setup, resource management, and maintenance operations for StarRocks.

**For table design, partitioning, bucketing, and indexes, see [TABLE.md](TABLE.md)**

## Table of Contents
- [Cluster Architecture](#cluster-architecture)
- [Resource Management](#resource-management)
- [Monitoring and Maintenance](#monitoring-and-maintenance)

---

## Cluster Architecture

### Components

**Frontend (FE):**
- Query planning and scheduling
- Metadata management
- Client connection handling
- Recommend: 3+ nodes for HA

**Backend (BE):**
- Data storage and processing
- Query execution
- Recommend: 3+ nodes minimum

**Broker (Optional):**
- Used for loading data from external storage (S3, HDFS)
- Lightweight, can share with FE nodes

### Cluster Configuration Files

**fe.conf (Frontend)**
```properties
# Metadata directory
meta_dir = /path/to/meta

# HTTP server port
http_port = 8030

# RPC port
rpc_port = 9020

# Query port
query_port = 9030

# Enable authentication
enable_auth_check = true

# Max connections per FE
qe_max_connection = 4096
```

**be.conf (Backend)**
```properties
# Storage directories (use multiple for better I/O)
storage_root_path = /data1/starrocks;/data2/starrocks

# Heartbeat service port
heartbeat_service_port = 9050

# BE server port
be_port = 9060

# BE HTTP port
be_http_port = 8040

# Number of scanner threads
scanner_thread_pool_thread_num = 48

# Number of workers per CPU core
pipeline_exec_thread_pool_thread_num = 0  # 0 means auto
```

### Cluster Management Commands

```sql
-- Add Backend
ALTER SYSTEM ADD BACKEND "host:port";

-- Remove Backend (data will be migrated)
ALTER SYSTEM DECOMMISSION BACKEND "host:port";

-- Add Frontend (Observer for read replicas)
ALTER SYSTEM ADD FOLLOWER "host:edit_log_port";
ALTER SYSTEM ADD OBSERVER "host:edit_log_port";

-- Check cluster status
SHOW PROC '/backends';
SHOW PROC '/frontends';

-- Check cluster load
SHOW PROC '/statistic';
```


---

## Resource Management

### Session-Level Resource Control

```sql
-- Set query timeout
SET query_timeout = 300;

-- Set memory limit per query
SET query_mem_limit = 8589934592;  -- 8GB in bytes

-- Enable spilling to disk for large operations
SET enable_spill = true;
SET spill_mem_limit_threshold = 0.8;

-- Control parallelism
SET pipeline_dop = 16;  -- Degree of parallelism
SET parallel_fragment_exec_instance_num = 8;
```

### Resource Groups (Global)

```sql
-- Create resource group for ETL jobs
CREATE RESOURCE GROUP etl_group
WITH (
    "cpu_core_limit" = "10",
    "mem_limit" = "50%",
    "concurrency_limit" = "10",
    "type" = "normal"
);

-- Create high-priority group for dashboards
CREATE RESOURCE GROUP dashboard_group
WITH (
    "cpu_core_limit" = "20",
    "mem_limit" = "30%",
    "concurrency_limit" = "20",
    "type" = "short_query"
);

-- Assign user to group
SET PROPERTY FOR 'etl_user' 'resource_group' = 'etl_group';

-- View resource groups
SHOW RESOURCE GROUPS ALL;
```

### Storage Volume Management

```sql
-- Create storage volume (for remote storage)
CREATE STORAGE VOLUME my_s3_volume
TYPE = S3
LOCATIONS = ("s3://my-bucket/starrocks/")
PROPERTIES (
    "aws.s3.region" = "us-east-1",
    "aws.s3.access_key" = "your_key",
    "aws.s3.secret_key" = "your_secret"
);

-- Set table to use remote storage
ALTER TABLE large_table
SET ("storage_volume" = "my_s3_volume");
```

---

## Monitoring and Maintenance

### Key Metrics to Monitor

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
-- Recent load jobs
SHOW LOAD ORDER BY CreateTime DESC LIMIT 10;

-- Failed loads
SHOW LOAD WHERE State = 'CANCELLED';

-- Stream load statistics
SHOW STREAM LOAD;
```

### Compaction Management

```sql
-- Check compaction status
SHOW PROC '/compactions';

-- Trigger manual compaction (if needed)
ALTER TABLE table_name COMPACT;

-- Configure compaction thresholds
ALTER TABLE table_name
SET (
    "compaction_policy" = "size_based",
    "min_cumulative_compaction_num_singleton_deltas" = "5",
    "max_cumulative_compaction_num_singleton_deltas" = "1000"
);
```

### Tablet Repair and Rebalancing

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

### Backup and Recovery

```sql
-- Create backup repository
CREATE REPOSITORY backup_repo
WITH BROKER
ON LOCATION "s3://backup-bucket/starrocks/"
PROPERTIES (
    "aws.s3.access_key" = "your_key",
    "aws.s3.secret_key" = "your_secret",
    "aws.s3.region" = "us-east-1"
);

-- Backup database
BACKUP SNAPSHOT db_name.snapshot_label
TO backup_repo
ON (table1, table2)
PROPERTIES ("type" = "full");

-- Check backup status
SHOW BACKUP FROM db_name;

-- Restore from backup
RESTORE SNAPSHOT db_name.snapshot_label
FROM backup_repo
ON (table1, table2)
PROPERTIES ("backup_timestamp" = "2024-01-01-00-00-00");
```

### Routine Maintenance Tasks

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

### Statistics Collection (for CBO)

```sql
-- Collect statistics for better query planning
ANALYZE TABLE table_name;

-- Collect for specific columns
ANALYZE TABLE table_name (col1, col2);

-- Auto collection
ALTER TABLE table_name
SET ("auto_analyze" = "true");

-- Check statistics
SHOW STATS table_name;
```

### Troubleshooting Commands

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

### Performance Tuning Checklist

**Table design (see [TABLE.md](TABLE.md)):**
- [ ] Partition with `date_trunc()` on time column
- [ ] Bucket count = 2-8x BE cores per partition
- [ ] Hash key has high cardinality
- [ ] Bloom filter on high-cardinality string columns
- [ ] Bitmap index on low-cardinality filter columns
- [ ] Colocate frequently joined tables
- [ ] Enable persistent index for Primary Key tables
- [ ] Set appropriate replication_num (3 for production)
- [ ] Configure partition TTL for automatic cleanup

**Cluster operations:**
- [ ] Monitor BE/FE health regularly
- [ ] Review slow queries weekly
- [ ] Check compaction status
- [ ] Verify tablet distribution balance
- [ ] Set up backup schedules
- [ ] Configure resource groups for workload isolation
