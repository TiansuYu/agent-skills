---
name: starrocks-cluster-setup
description: Set up and manage StarRocks cluster topology — Frontend (FE), Backend (BE), and Broker roles, fe.conf/be.conf configuration, adding/removing/decommissioning nodes, and checking cluster status. Use when deploying or scaling a StarRocks cluster, configuring FE/BE nodes, adding followers/observers, or checking cluster health.
license: Apache-2.0
metadata:
    author: "Tiansu Yu"
    version: "1.0"
---

# StarRocks Cluster Setup

Deploy and manage the StarRocks cluster topology: FE/BE/Broker roles, node configuration, and node lifecycle operations.

## When to Use

- Deploying a new StarRocks cluster or planning topology
- Configuring `fe.conf` / `be.conf`
- Adding, removing, or decommissioning BE nodes
- Adding FE followers/observers for HA and read scaling
- Checking cluster status

Related skills: [starrocks-resource-management] for session limits, resource groups, and storage volumes; [starrocks-monitoring] for ongoing health checks and maintenance.

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
