# StarRocks Table Design

This guide covers table design, partitioning, bucketing, and indexing strategies for StarRocks.

Reference: [StarRocks Partitioning Best Practices](https://docs.starrocks.io/docs/best_practices/partitioning/)

## Table of Contents
- [Table Types and DDL](#table-types-and-ddl)
- [Partitioning Strategies](#partitioning-strategies)
- [Bucketing and Distribution](#bucketing-and-distribution)
- [Indexes and Optimization](#indexes-and-optimization)

---

## Table Types and DDL

### Choosing the Right Table Type

#### 1. Duplicate Key Table
**Use when:** Raw event logging, append-only data, no deduplication needed

```sql
CREATE TABLE events_log (
    event_time DATETIME NOT NULL,
    user_id BIGINT,
    event_type VARCHAR(50),
    properties JSON
)
DUPLICATE KEY (event_time, user_id)
PARTITION BY date_trunc('day', event_time)
DISTRIBUTED BY HASH(user_id) BUCKETS 32;
```

**Pros:** Fastest ingestion, no overhead
**Cons:** No deduplication, largest storage

#### 2. Aggregate Key Table
**Use when:** Data can be pre-aggregated during ingestion

```sql
CREATE TABLE metrics_daily (
    metric_date DATE NOT NULL,
    metric_name VARCHAR(100) NOT NULL,
    total_count BIGINT SUM,
    total_amount DECIMAL(20, 2) SUM,
    max_value DECIMAL(20, 2) MAX,
    min_value DECIMAL(20, 2) MIN
)
AGGREGATE KEY (metric_date, metric_name)
PARTITION BY date_trunc('day', metric_date)
DISTRIBUTED BY HASH(metric_name) BUCKETS 16;
```

**Pros:** Reduced storage, faster queries
**Cons:** Limited aggregation functions (SUM, MAX, MIN, REPLACE)

#### 3. Unique Key Table
**Use when:** Need to enforce uniqueness, support updates

```sql
CREATE TABLE user_profiles (
    user_id BIGINT NOT NULL,
    username VARCHAR(100),
    email VARCHAR(200),
    status VARCHAR(20),
    updated_at DATETIME
)
UNIQUE KEY (user_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 16
PROPERTIES (
    "enable_persistent_index" = "true"
);
```

**Pros:** Supports upserts, deduplication
**Cons:** Higher write latency than DUPLICATE

#### 4. Primary Key Table
**Use when:** Frequent updates, real-time analytics with mutable data

```sql
CREATE TABLE orders_realtime (
    order_id BIGINT NOT NULL,
    order_time DATETIME NOT NULL,
    customer_id BIGINT,
    status VARCHAR(50),
    amount DECIMAL(18, 2),
    updated_at DATETIME
)
PRIMARY KEY (order_id)
PARTITION BY date_trunc('day', order_time)
DISTRIBUTED BY HASH(order_id) BUCKETS 32
PROPERTIES (
    "enable_persistent_index" = "true",
    "replicated_storage" = "true"
);
```

**Pros:** Best update performance, point query optimization
**Cons:** More memory usage for indexes

### Column Type Selection

**Best practices:**
- Use DATETIME for time columns (not VARCHAR)
- Use DECIMAL for monetary values (not DOUBLE)
- Use BIGINT for IDs (not VARCHAR when possible)
- Use JSON for semi-structured data
- Use ARRAY/MAP/STRUCT for nested data

**Example:**
```sql
CREATE TABLE transactions (
    -- Good: DATETIME for timestamps
    transaction_time DATETIME NOT NULL,
    
    -- Good: BIGINT for IDs
    transaction_id BIGINT NOT NULL,
    user_id BIGINT,
    
    -- Good: DECIMAL for money
    amount DECIMAL(18, 2),
    
    -- Good: VARCHAR with appropriate length
    currency VARCHAR(3),
    
    -- Good: JSON for flexible data
    metadata JSON,
    
    -- Good: ARRAY for multi-value
    tags ARRAY<VARCHAR(50)>
)
DUPLICATE KEY (transaction_time, transaction_id)
PARTITION BY date_trunc('day', transaction_time)
DISTRIBUTED BY HASH(transaction_id) BUCKETS 32;
```

---

## Partitioning Strategies

**Modern partitioning uses expression-based syntax with automatic partition creation.**

### Partitioning vs Bucketing - Different Jobs

| Aspect | Partitioning | Bucketing |
|--------|-------------|-----------|
| **Primary goal** | Coarse-grain data pruning and lifecycle control (TTL, archiving) | Fine-grain parallelism and data locality |
| **Planner visibility** | FE can skip entire partitions via predicates | Only equality predicates support bucket pruning |
| **Lifecycle ops** | DROP PARTITION is metadata-only (ideal for GDPR, TTL) | Buckets can't be dropped individually |
| **Typical count** | 10²-10⁴ per table (days, weeks, tenants) | 10-120 per partition |
| **Red flags** | >100k partitions causes FE memory issues | >200k tablets per BE; tablets >10GB slow compaction |

### When Should You Partition?

| Table Type | Partition? | Typical Key |
|------------|-----------|-------------|
| Fact/event stream | **Yes** | `date_trunc('day', event_time)` |
| Huge dimension (billions of rows) | Sometimes | Time or business key change date |
| Small dimension/lookup (<100M rows) | No | Rely on hash distribution only |

**Key question:** If 80%+ of queries include a time filter, partition on time.

### Expression-Based Partitioning (Recommended)

**Daily partitions (most common):**
```sql
CREATE TABLE click_stream (
    user_id BIGINT,
    event_time DATETIME,
    url STRING,
    page_views INT
)
DUPLICATE KEY (user_id, event_time)
PARTITION BY date_trunc('day', event_time)
DISTRIBUTED BY HASH(user_id) BUCKETS 32;
```

**Hourly partitions (high-frequency data):**
```sql
CREATE TABLE iot_sensors (
    sensor_id BIGINT,
    reading_time DATETIME,
    temperature DOUBLE
)
DUPLICATE KEY (sensor_id, reading_time)
PARTITION BY date_trunc('hour', reading_time)
DISTRIBUTED BY HASH(sensor_id) BUCKETS 64;
```

**Monthly partitions (long-term storage):**
```sql
CREATE TABLE historical_archive (
    report_month DATETIME,
    account_id BIGINT,
    metrics JSON
)
DUPLICATE KEY (report_month, account_id)
PARTITION BY date_trunc('month', report_month)
DISTRIBUTED BY HASH(account_id) BUCKETS 16;
```

### Composite Partitioning (Multi-Tenant)

**Pattern A: Time-first (recommended for most SaaS workloads)**

Prunes on time, keeps tenant rows co-located:
```sql
CREATE TABLE metrics (
    tenant_id INT,
    dt DATETIME,
    metric_name STRING,
    value DOUBLE
)
PRIMARY KEY (tenant_id, dt, metric_name)
PARTITION BY date_trunc('day', dt)
DISTRIBUTED BY HASH(tenant_id) BUCKETS 32;
```

**Pattern B: Tenant-first (for large tenants needing isolation)**

Enables tenant-specific operations but creates more partitions:
```sql
CREATE TABLE activity (
    tenant_id INT,
    dt DATETIME,
    id BIGINT,
    event_data JSON
)
DUPLICATE KEY (dt, id)
PARTITION BY tenant_id, date_trunc('month', dt)
DISTRIBUTED BY HASH(id) BUCKETS 32;

-- Warning: #tenants × #months partitions. Keep under 100k total.
```

### Choosing the Partition Key

**Priority order:**

1. **Time-first default** - If 80%+ of queries include a time filter: `PARTITION BY date_trunc('day', dt)`
2. **Tenant isolation** - Add tenant for tenant-level data management: `PARTITION BY tenant_id, date_trunc('day', dt)`
3. **Retention alignment** - Partition by the column you'll purge on (enables metadata-only DROP)
4. **Composite keys** - Creates `#tenants × #days` partitions - keep total under 100k

### Choosing Partition Granularity

| Granularity | Use When | Pros | Cons | Annual Count |
|-------------|----------|------|------|--------------|
| `date_trunc('hour', dt)` | >2 tablets/day needed, IoT bursts, hot-spot isolation | Fine-grained pruning, 24 partitions/day | 8,760 partitions/year | High |
| `date_trunc('day', dt)` | **Default for most BI/reporting (80% of use cases)** | Balanced: 365 partitions/year, simple TTL | Less precise for "last 3h" queries | Medium |
| `date_trunc('week', dt)` | Weekly aggregations, medium-term storage | Lower partition count | Coarser pruning | Low |
| `date_trunc('month', dt)` | Historical archive, long retention (>2 years) | Minimal metadata overhead | Very coarse pruning | Very Low |

**Sizing guidelines:**
- Each partition: ≤100 GB data
- Each partition: ≤20k tablets (across all replicas)
- Total partitions per table: <100k (critical for FE memory)
- Watch composite partitions: 1000 tenants × 365 days = 365k partitions ⚠️

**Mixed granularity (StarRocks 3.4+):**
You can merge historical partitions into coarser granularity for better efficiency.

### Partition Management

**With expression-based partitioning, partitions are created automatically on INSERT.**

```sql
-- View existing partitions
SHOW PARTITIONS FROM table_name;

-- Example output shows auto-generated partition names:
-- p20240101000000  (for date_trunc('day', ...))
-- p20240101120000  (for date_trunc('hour', ...))
-- p10001_20240101000000 (for composite: tenant_id, date_trunc('day', ...))

-- Drop old partitions by name
ALTER TABLE table_name
DROP PARTITION p20231201000000;

-- Drop multiple partitions
ALTER TABLE table_name
DROP PARTITION p20231201000000, p20231202000000;

-- Truncate specific partition
TRUNCATE TABLE table_name PARTITION (p20240101000000);
```

**Partition lifecycle management (TTL):**
```sql
-- Set automatic partition TTL (drops partitions older than N days)
ALTER TABLE events
SET ("partition_live_number" = "90");  -- Keep only last 90 partitions

-- Or use partition retention in days
ALTER TABLE events
SET ("partition_retention_time" = "90d");
```

### Partition Pruning Verification

```sql
-- ✅ Good: Filter on partitioned column directly
EXPLAIN VERBOSE
SELECT * FROM events
WHERE event_time >= '2024-01-01' AND event_time < '2024-01-07';
-- Look for: partitions=7/365 (only 7 partitions scanned)

-- ❌ Bad: Function on partitioned column breaks pruning
EXPLAIN VERBOSE
SELECT * FROM events
WHERE DATE(event_time) = '2024-01-01';
-- Results in: partitions=365/365 (full scan!)

-- ✅ Good: Use date_trunc in query matching partition definition
EXPLAIN VERBOSE
SELECT * FROM events
WHERE date_trunc('day', event_time) = '2024-01-01';
-- Results in: partitions=1/365
```

**Best practice:** Always filter using the same expression or direct comparison on the partitioned column.

---

## Bucketing and Distribution

### Determining Bucket Count

**Formula:**
```
bucket_count = (data_size_per_partition_GB / 2GB) × number_of_BE_nodes
Minimum: 1
Maximum: typically 128-256
```

**Examples:**

| Partition Size | BE Nodes | Recommended Buckets | Reasoning |
|----------------|----------|---------------------|-----------|
| 1 GB | 3 | 8 | Small dataset, min distribution |
| 10 GB | 3 | 16 | ~2GB per bucket |
| 100 GB | 10 | 64 | ~1.5GB per bucket |
| 1 TB | 20 | 128 | ~8GB per bucket |

### Distribution Strategies

#### 1. Hash Distribution (Most Common)

```sql
-- Single column hash
DISTRIBUTED BY HASH(user_id) BUCKETS 32;

-- Multi-column hash (for better distribution)
DISTRIBUTED BY HASH(user_id, event_date) BUCKETS 32;
```

**Choose hash column based on:**
- High cardinality (many unique values)
- Frequently used in WHERE/JOIN conditions
- Even distribution (avoid skew)

#### 2. Random Distribution

```sql
DISTRIBUTED BY RANDOM BUCKETS 16;
```

**Use when:**
- No good hash key available
- Data is already evenly distributed
- Table is small

### Colocate Groups (Advanced)

**Use for:** Frequently joined large tables

```sql
-- Create first table in group
CREATE TABLE orders (
    order_id BIGINT,
    user_id BIGINT,
    amount DECIMAL(18, 2)
)
DUPLICATE KEY (order_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 32
PROPERTIES (
    "colocate_with" = "user_order_group"
);

-- Create second table in same group
CREATE TABLE users (
    user_id BIGINT,
    username VARCHAR(100)
)
UNIQUE KEY (user_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 32
PROPERTIES (
    "colocate_with" = "user_order_group"
);

-- Now JOINs on user_id are local (no network shuffle)
SELECT o.*, u.username
FROM orders o
JOIN users u ON o.user_id = u.user_id;
```

**Requirements:**
- Same bucketing column
- Same bucket count
- Same replication strategy

---

## Indexes and Optimization

### 1. Prefix Index (Automatic)

StarRocks automatically creates a prefix index on the first 36 bytes of DUPLICATE/AGGREGATE/UNIQUE keys.

```sql
-- This table has automatic prefix index on (event_time, user_id)
CREATE TABLE events (
    event_time DATETIME NOT NULL,  -- 8 bytes
    user_id BIGINT NOT NULL,       -- 8 bytes
    event_type VARCHAR(50),        -- 50+ bytes (only partially indexed)
    data JSON
)
DUPLICATE KEY (event_time, user_id, event_type)
PARTITION BY date_trunc('day', event_time)
DISTRIBUTED BY HASH(user_id) BUCKETS 32;
```

**Best practice:** Put frequently filtered columns first in key definition

### 2. Bloom Filter Index

```sql
CREATE TABLE users (
    user_id BIGINT,
    email VARCHAR(200),
    phone VARCHAR(20),
    username VARCHAR(100)
)
UNIQUE KEY (user_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 16
PROPERTIES (
    "bloom_filter_columns" = "email,phone"
);

-- Or add to existing table
ALTER TABLE users
SET ("bloom_filter_columns" = "email,phone");
```

**Use for:**
- High-cardinality VARCHAR columns
- Equality filters (WHERE email = '...')
- NOT IN queries

**Don't use for:**
- Low-cardinality columns
- Range queries
- Numeric columns with range filters

### 3. Bitmap Index

```sql
ALTER TABLE events
SET ("indexes" = "idx_status ON status USING BITMAP");

-- Or in CREATE TABLE
CREATE TABLE events (
    event_time DATETIME,
    status VARCHAR(20) INDEX idx_status USING BITMAP,
    category VARCHAR(50)
)
DUPLICATE KEY (event_time)
PARTITION BY date_trunc('day', event_time)
DISTRIBUTED BY HASH(user_id) BUCKETS 32;
```

**Use for:**
- Low to medium cardinality columns (10-10000 distinct values)
- Frequent WHERE filters
- Common in OLAP dimensions (status, category, region)

### 4. NGram Bloom Filter (Full-text Search)

```sql
ALTER TABLE articles
SET ("indexes" = "idx_content ON content USING NGRAMBF(5)");
```

**Use for:**
- LIKE '%keyword%' searches
- Text search scenarios

---

## Table Design Checklist

Before creating production tables:

- [ ] Choose correct table type (Duplicate/Aggregate/Unique/Primary)
- [ ] Use DATETIME for time columns (not VARCHAR)
- [ ] Partition with `date_trunc()` expression matching query patterns
- [ ] Partition granularity matches use case (hour/day/month)
- [ ] Composite partitions kept under 100k total
- [ ] Bucket count follows formula: (partition_size_GB / 2GB) × BE_count
- [ ] Hash key has high cardinality and even distribution
- [ ] Frequently filtered columns in DUPLICATE KEY prefix (first 36 bytes)
- [ ] Bloom filter on high-cardinality string columns (email, phone)
- [ ] Bitmap index on low-cardinality filter columns (status, region)
- [ ] Colocate frequently joined tables in same group
- [ ] Enable persistent index for Primary Key tables
- [ ] Set replication_num = 3 for production
- [ ] Configure partition TTL for automatic cleanup
