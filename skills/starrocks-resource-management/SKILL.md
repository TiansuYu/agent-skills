---
name: starrocks-resource-management
description: Manage StarRocks resources — session-level controls (query timeout, memory limit, parallelism, disk spill), resource groups for workload isolation (ETL vs dashboards), and storage volumes for remote/tiered storage. Use when isolating workloads, capping query memory/CPU, controlling parallelism, configuring resource groups, or setting up S3-backed storage volumes.
license: Apache-2.0
metadata:
    author: "Tiansu Yu"
    version: "1.0"
---

# StarRocks Resource Management

Control how StarRocks consumes compute and storage: session limits, resource groups, and storage volumes.

## When to Use

- Isolating workloads (ETL vs interactive dashboards)
- Capping per-query memory, CPU, or concurrency
- Controlling query parallelism or enabling disk spill
- Setting up remote/tiered storage with storage volumes

Related skills: [starrocks-cluster-setup] for the underlying node topology; [starrocks-query-optimization] for tuning parallelism on specific queries; [starrocks-monitoring] for observing resource usage.

## Session-Level Resource Control

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

## Resource Groups (Global)

Users/roles are mapped to a group via **classifiers** in the `TO (...)` clause — there is no `SET PROPERTY ... 'resource_group'`. CPU share is set with `cpu_weight` (relative weight) or `exclusive_cpu_cores` (dedicated cores); the old `cpu_core_limit`/`"type"` properties are deprecated (since v3.3.5).

```sql
-- Create resource group for ETL jobs, mapped to a user via a classifier
CREATE RESOURCE GROUP etl_group
TO (user = 'etl_user')
WITH (
    "cpu_weight" = "10",
    "mem_limit" = "50%",
    "concurrency_limit" = "10"
);

-- Create high-priority group for dashboards with dedicated CPU cores
CREATE RESOURCE GROUP dashboard_group
TO (user = 'dashboard_user', query_type in ('select'))
WITH (
    "exclusive_cpu_cores" = "20",
    "mem_limit" = "30%",
    "concurrency_limit" = "20"
);

-- View resource groups
SHOW RESOURCE GROUPS ALL;
```

## Storage Volume Management

```sql
-- Create storage volume (for remote storage)
CREATE STORAGE VOLUME my_s3_volume
TYPE = S3
LOCATIONS = ("s3://my-bucket/starrocks/")   -- bucket/path only (use s3:// here, unlike backup repos which use s3a://)
PROPERTIES (
    "enabled" = "true",                       -- required to activate the volume
    "aws.s3.region" = "us-east-1",
    "aws.s3.access_key" = "your_key",
    "aws.s3.secret_key" = "your_secret"
);

-- Set table to use remote storage
ALTER TABLE large_table
SET ("storage_volume" = "my_s3_volume");
```
