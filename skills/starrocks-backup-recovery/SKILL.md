---
name: starrocks-backup-recovery
description: Back up and restore StarRocks data — create a backup repository (S3 via broker), take full snapshots of tables/databases, check backup status, and restore from a snapshot. Use when setting up StarRocks backups, snapshotting tables before risky changes, configuring a backup repository, or recovering data from a snapshot.
license: Apache-2.0
metadata:
    author: "Tiansu Yu"
    version: "1.0"
---

# StarRocks Backup and Recovery

Protect StarRocks data with repository-based snapshots: backup repository setup, taking snapshots, checking status, and restoring.

## When to Use

- Setting up a backup repository (e.g. S3)
- Snapshotting tables or databases (scheduled or before risky changes)
- Checking backup job status
- Restoring data from a snapshot

Related skills: [starrocks-monitoring] for routine maintenance scheduling; [starrocks-cluster-setup] for the broker used by the repository.

## Backup and Recovery

```sql
-- Create backup repository
CREATE REPOSITORY backup_repo
WITH BROKER
ON LOCATION "s3a://backup-bucket/starrocks/"   -- backup repos require the s3a:// prefix
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
