# Snowflake Troubleshooting Performance Issues
## Advanced Certification Study Guide

---

## Table of Contents
1. [System Clustering Information](#system-clustering-information)
2. [Warehouse Monitoring](#warehouse-monitoring)
3. [Optimization Techniques](#optimization-techniques)
4. [Micro-Partition Pruning](#micro-partition-pruning)
5. [Monitoring and Alerting](#monitoring-and-alerting)
6. [Troubleshooting Workflow](#troubleshooting-workflow)
7. [Exam Tips and Practice Scenarios](#exam-tips-and-practice-scenarios)

---

## System Clustering Information

### Understanding System Clustering Metadata

**System Clustering**: Snowflake's automatic measurement and reporting of how well data is organized in micropartitions relative to defined clustering keys.

#### Clustering Depth Concept

**Definition**: A ratio (0-1) indicating how well data is clustered on defined clustering keys. Lower values = better clustering.

```
Clustering Depth Scale:
0.0 = Perfectly clustered (ideal)
0.3 = Good clustering (functional)
0.5 = Moderate clustering (acceptable)
0.8 = Poor clustering (consider remediation)
1.0 = Randomly ordered (no clustering benefit)
```

#### SYSTEM$CLUSTERING_DEPTH() Function

**Syntax**:
```sql
SYSTEM$CLUSTERING_DEPTH('fully_qualified_table_name', 'column_name');
SYSTEM$CLUSTERING_DEPTH('database.schema.table_name', 'column_name');
```

**Basic Usage**:
```sql
-- Check clustering depth for single column
SELECT SYSTEM$CLUSTERING_DEPTH('sales.public.orders', 'customer_id');
-- Returns: 0.25 (good clustering)

-- Check depth for multiple columns
SELECT SYSTEM$CLUSTERING_DEPTH('sales.public.orders', 'customer_id, order_date');
-- Returns: 0.15 (excellent clustering)

-- Monitor clustering across tables
SELECT 
    table_name,
    SYSTEM$CLUSTERING_DEPTH('sales.public.' || table_name, 'customer_id') as clustering_depth
FROM information_schema.tables 
WHERE table_schema = 'PUBLIC'
    AND table_type = 'BASE TABLE'
ORDER BY clustering_depth DESC;
```

#### Interpreting Clustering Depth Results

**Quality Assessment Framework**:

| Depth Value | Classification | Action Required | Performance Impact |
|------------|-----------------|-----------------|-------------------|
| 0.0 - 0.2 | Excellent | None | Optimal pruning, minimal scan |
| 0.2 - 0.4 | Good | Monitor | Good pruning, efficient scan |
| 0.4 - 0.6 | Acceptable | Review | Moderate pruning, acceptable |
| 0.6 - 0.8 | Degraded | Investigate | Limited pruning, investigate causes |
| 0.8 - 1.0 | Poor | Remediate | No pruning, critical issue |

#### Common Clustering Depth Issues and Solutions

**Issue 1: Clustering Depth Exceeds 0.8**

```sql
-- Diagnose problem
SELECT 
    table_name,
    SYSTEM$CLUSTERING_DEPTH('schema.table_name', 'clustering_column') as depth
FROM information_schema.tables
WHERE depth > 0.8;

-- Root cause analysis
-- 1. Auto-clustering disabled?
SHOW TABLES LIKE 'problem_table';
-- Look for CLUSTERING_KEY column

-- 2. Too many random inserts?
SELECT 
    DATE_TRUNC('day', last_altered) as load_date,
    COUNT(*) as inserts_per_day
FROM information_schema.table_history
WHERE table_name = 'problem_table'
GROUP BY load_date
ORDER BY load_date DESC
LIMIT 7;

-- 3. Clustering key no longer matches query patterns?
-- Review current queries vs. defined clustering key
```

**Issue 2: Clustering Depth Degradation Over Time**

```sql
-- Monitor clustering depth trend
-- Create monitoring table
CREATE OR REPLACE TABLE clustering_depth_history (
    check_date TIMESTAMP,
    table_name VARCHAR,
    clustering_column VARCHAR,
    clustering_depth FLOAT
);

-- Daily monitoring job
INSERT INTO clustering_depth_history
SELECT 
    CURRENT_TIMESTAMP(),
    table_name,
    'customer_id',
    SYSTEM$CLUSTERING_DEPTH('schema.' || table_name, 'customer_id')
FROM information_schema.tables
WHERE table_schema = 'ANALYTICS';

-- Trend analysis
SELECT 
    table_name,
    check_date,
    clustering_depth,
    LAG(clustering_depth) OVER (PARTITION BY table_name ORDER BY check_date) as prev_depth,
    clustering_depth - LAG(clustering_depth) OVER (PARTITION BY table_name ORDER BY check_date) as depth_change
FROM clustering_depth_history
WHERE check_date >= DATEADD(month, -1, CURRENT_DATE())
ORDER BY table_name, check_date DESC;

-- Alert on significant degradation
SELECT *
FROM clustering_depth_history
WHERE clustering_depth - LAG(clustering_depth) OVER (PARTITION BY table_name ORDER BY check_date) > 0.1
    AND clustering_depth > 0.6;
```

**Issue 3: Clustering Key No Longer Optimal**

```sql
-- Analyze query patterns vs. clustering key
WITH query_filters AS (
    SELECT 
        query_text,
        CASE 
            WHEN query_text ILIKE '%WHERE%customer_id%' THEN 'customer_id'
            WHEN query_text ILIKE '%WHERE%order_date%' THEN 'order_date'
            WHEN query_text ILIKE '%WHERE%region%' THEN 'region'
            ELSE 'other'
        END as filtered_column,
        COUNT(*) as query_count,
        AVG(execution_time) as avg_execution_ms
    FROM snowflake.account_usage.query_history
    WHERE database_name = 'SALES'
        AND table_name = 'ORDERS'
        AND start_time >= DATEADD(week, -4, CURRENT_DATE())
    GROUP BY filtered_column
)
SELECT 
    filtered_column,
    query_count,
    ROUND(100.0 * query_count / SUM(query_count) OVER (), 2) as pct_of_queries,
    ROUND(avg_execution_ms, 0) as avg_exec_ms
FROM query_filters
ORDER BY query_count DESC;

-- Decision: If dominant filter column differs from clustering key, consider re-clustering
```

### Managing Clustering Keys Based on System Information

#### Adjusting Clustering Keys

```sql
-- Drop existing clustering key
ALTER TABLE orders DROP CLUSTERING KEY;

-- Define new clustering key based on system analysis
ALTER TABLE orders CLUSTER BY (order_date, customer_id);

-- Enable auto-clustering for maintenance
-- (Assuming clustering key is defined)
-- Snowflake auto-maintains by default

-- Verify new clustering key
SHOW TABLES LIKE 'orders';
-- Check CLUSTERING_KEY column

-- Monitor impact
SELECT 
    SYSTEM$CLUSTERING_DEPTH('sales.public.orders', 'order_date'),
    SYSTEM$CLUSTERING_DEPTH('sales.public.orders', 'customer_id'),
    SYSTEM$CLUSTERING_DEPTH('sales.public.orders', 'order_date, customer_id')
AS clustering_depth_combined;
```

#### When to Reconsider Clustering Strategy

**Trigger Events**:
1. **Clustering depth exceeds 0.7** for > 7 days
2. **Query patterns changed** significantly
3. **Table growth > 10x** with same clustering key
4. **Auto-clustering cost exceeds benefit** (> 50% of query savings)
5. **New high-frequency queries** on different columns

#### Cost-Benefit Analysis for Reclustering

```sql
-- Calculate clustering impact
WITH clustering_cost AS (
    -- Auto-clustering credit cost
    SELECT 
        'AUTO_CLUSTERING_COST' as metric,
        SUM(credits_used) as value
    FROM snowflake.account_usage.metering_history
    WHERE service = 'AUTOMATIC_CLUSTERING'
        AND start_time >= DATEADD(month, -1, CURRENT_DATE())
),
query_benefit AS (
    -- Query performance improvement from clustering
    SELECT 
        'QUERY_BENEFIT' as metric,
        COUNT(DISTINCT query_id) * AVG(bytes_scanned) / (1024*1024*1024) * 0.1 as value
        -- Rough estimate: 10% of scanned bytes as saved credits
    FROM snowflake.account_usage.query_history
    WHERE table_name = 'ORDERS'
        AND start_time >= DATEADD(month, -1, CURRENT_DATE())
)
SELECT *
FROM clustering_cost
UNION ALL
SELECT *
FROM query_benefit;

-- Decision logic:
-- IF query_benefit > clustering_cost * 1.5 THEN keep clustering
-- ELSE review clustering strategy
```

### System Clustering Tables and Views

#### Key Metadata Views for Clustering

```sql
-- Table clustering information
SELECT 
    table_catalog,
    table_schema,
    table_name,
    clustering_key
FROM information_schema.tables
WHERE clustering_key IS NOT NULL
ORDER BY table_schema, table_name;

-- Check table history for clustering changes
SELECT 
    table_name,
    created,
    last_altered,
    deleted,
    active
FROM information_schema.table_history
WHERE table_name = 'orders'
ORDER BY created DESC;

-- Monitor table size related to clustering
SELECT 
    table_name,
    active_bytes / (1024*1024*1024) as size_gb,
    fail_safe_bytes / (1024*1024*1024) as fail_safe_gb,
    time_travel_bytes / (1024*1024*1024) as time_travel_gb
FROM information_schema.table_storage_metrics
WHERE schema_name = 'ANALYTICS'
ORDER BY active_bytes DESC;
```

#### Automatic Clustering System Views

```sql
-- Monitor automatic clustering activity
SELECT 
    database_name,
    schema_name,
    table_name,
    START_TIME,
    END_TIME,
    DATEDIFF(minute, START_TIME, END_TIME) as duration_minutes,
    ROWS_INSERTED,
    ROWS_UPDATED,
    BYTES_RECLUSTERED / (1024*1024) as bytes_reclustered_mb
FROM snowflake.account_usage.automatic_clustering_history
WHERE start_time >= DATEADD(week, -1, CURRENT_DATE())
ORDER BY start_time DESC;

-- Identify tables with excessive clustering overhead
SELECT 
    table_name,
    COUNT(*) as clustering_operations,
    SUM(DATEDIFF(minute, START_TIME, END_TIME)) as total_duration_minutes,
    SUM(BYTES_RECLUSTERED) / (1024*1024*1024) as total_bytes_gb
FROM snowflake.account_usage.automatic_clustering_history
WHERE start_time >= DATEADD(month, -1, CURRENT_DATE())
GROUP BY table_name
HAVING COUNT(*) > 10
ORDER BY total_bytes_gb DESC;

-- Determine if clustering is worth the cost
WITH clustering_stats AS (
    SELECT 
        table_name,
        SUM(BYTES_RECLUSTERED) / (1024*1024*1024) as bytes_reclustered_gb,
        COUNT(*) as operations_count
    FROM snowflake.account_usage.automatic_clustering_history
    WHERE start_time >= DATEADD(month, -1, CURRENT_DATE())
    GROUP BY table_name
),
query_stats AS (
    SELECT 
        table_name,
        COUNT(*) as query_count,
        SUM(bytes_scanned) / (1024*1024*1024) as total_bytes_scanned_gb,
        AVG(execution_time) as avg_execution_ms
    FROM snowflake.account_usage.query_history
    WHERE start_time >= DATEADD(month, -1, CURRENT_DATE())
    GROUP BY table_name
)
SELECT 
    c.table_name,
    c.bytes_reclustered_gb,
    c.operations_count,
    q.query_count,
    ROUND(q.total_bytes_scanned_gb, 2) as scanned_gb,
    CASE 
        WHEN c.bytes_reclustered_gb > q.total_bytes_scanned_gb * 0.2 THEN 'HIGH_COST'
        WHEN c.bytes_reclustered_gb > q.total_bytes_scanned_gb * 0.1 THEN 'MODERATE_COST'
        ELSE 'LOW_COST'
    END as clustering_cost_assessment
FROM clustering_stats c
JOIN query_stats q ON c.table_name = q.table_name
ORDER BY c.bytes_reclustered_gb DESC;
```

---

## Warehouse Monitoring

### Warehouse Health Metrics

#### Key Metrics to Monitor

**1. Warehouse Utilization**

```sql
-- CPU and memory utilization
SELECT 
    warehouse_name,
    DATE_TRUNC('hour', start_time) as hour,
    COUNT(DISTINCT query_id) as concurrent_queries,
    COUNT(*) as total_operations,
    AVG(execution_time) as avg_execution_ms,
    MAX(execution_time) as max_execution_ms,
    ROUND(100.0 * COUNT(CASE WHEN execution_time < 1000 THEN 1 END) / COUNT(*), 2) as pct_fast_queries
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_DATE())
GROUP BY warehouse_name, hour
ORDER BY warehouse_name, hour DESC;
```

**2. Queue Analysis**

```sql
-- Warehouse queuing patterns
SELECT 
    warehouse_name,
    warehouse_size,
    DATE_TRUNC('day', start_time) as day,
    COUNT(*) as total_queries,
    COUNT(CASE WHEN queued_provisioning_time > 0 THEN 1 END) as queued_queries,
    ROUND(100.0 * COUNT(CASE WHEN queued_provisioning_time > 0 THEN 1 END) / COUNT(*), 2) as pct_queued,
    AVG(queued_provisioning_time) as avg_queue_ms,
    MAX(queued_provisioning_time) as max_queue_ms,
    SUM(queued_provisioning_time) as total_queue_ms
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(week, -4, CURRENT_DATE())
GROUP BY warehouse_name, warehouse_size, day
ORDER BY warehouse_name, day DESC;

-- Alert on queue formation
SELECT 
    warehouse_name,
    COUNT(CASE WHEN queued_provisioning_time > 0 THEN 1 END) as queued_queries,
    AVG(queued_provisioning_time) as avg_queue_ms
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(hour, -4, CURRENT_TIMESTAMP())
    AND queued_provisioning_time > 0
GROUP BY warehouse_name
HAVING COUNT(*) > 5;
```

**3. Credit Consumption**

```sql
-- Credit usage by warehouse
SELECT 
    warehouse_name,
    DATE_TRUNC('day', start_time) as day,
    SUM(credits_used) as daily_credits,
    COUNT(DISTINCT query_id) as query_count,
    ROUND(AVG(credits_used), 4) as avg_credits_per_query,
    MAX(credits_used) as max_query_credits
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD(month, -3, CURRENT_DATE())
GROUP BY warehouse_name, day
ORDER BY warehouse_name, day DESC;

-- Identify high-cost queries
SELECT 
    query_id,
    query_text,
    warehouse_name,
    warehouse_size,
    total_elapsed_time,
    credits_used,
    bytes_scanned,
    ROUND(credits_used / NULLIF(execution_time, 0) * 1000, 4) as credits_per_second
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_DATE())
    AND credits_used > 10
ORDER BY credits_used DESC
LIMIT 20;
```

**4. Spilling Metrics**

```sql
-- Monitor spilling across warehouse
SELECT 
    warehouse_name,
    warehouse_size,
    DATE_TRUNC('day', start_time) as day,
    COUNT(CASE WHEN bytes_spilled_to_local_storage > 0 THEN 1 END) as local_spill_count,
    COUNT(CASE WHEN bytes_spilled_to_remote_storage > 0 THEN 1 END) as remote_spill_count,
    SUM(bytes_spilled_to_local_storage) / (1024*1024*1024) as total_local_spill_gb,
    SUM(bytes_spilled_to_remote_storage) / (1024*1024*1024) as total_remote_spill_gb,
    COUNT(*) as total_queries
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(month, -1, CURRENT_DATE())
GROUP BY warehouse_name, warehouse_size, day
HAVING total_local_spill_gb > 0 OR total_remote_spill_gb > 0
ORDER BY day DESC;

-- Critical alert for remote spilling
SELECT 
    query_id,
    query_text,
    warehouse_name,
    bytes_scanned / (1024*1024*1024) as bytes_scanned_gb,
    bytes_spilled_to_remote_storage / (1024*1024*1024) as remote_spill_gb,
    execution_time,
    ROUND(bytes_spilled_to_remote_storage::numeric / NULLIF(bytes_scanned, 0), 4) as spill_ratio
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_DATE())
    AND bytes_spilled_to_remote_storage > 0
ORDER BY bytes_spilled_to_remote_storage DESC;
```

#### Warehouse Configuration Analysis

```sql
-- Current warehouse configurations
SHOW WAREHOUSES;

-- Parse warehouse configuration
SELECT 
    "name",
    "type",
    "size",
    "max_cluster_count",
    "min_cluster_count",
    "scaling_policy",
    "auto_suspend",
    "auto_resume",
    "comment"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Compare configuration against utilization
WITH wh_config AS (
    SELECT 
        'interactive_wh' as warehouse_name,
        'SMALL' as size,
        1 as min_clusters,
        3 as max_clusters
    UNION ALL
    SELECT 'batch_wh', 'LARGE', 1, 1
),
wh_usage AS (
    SELECT 
        warehouse_name,
        SUM(credits_used) as daily_credits,
        COUNT(DISTINCT query_id) as query_count,
        AVG(execution_time) as avg_exec_ms,
        MAX(execution_time) as max_exec_ms,
        COUNT(CASE WHEN execution_time > 60000 THEN 1 END) as slow_queries
    FROM snowflake.account_usage.warehouse_metering_history
    WHERE start_time >= DATEADD(day, -7, CURRENT_DATE())
    GROUP BY warehouse_name
)
SELECT 
    c.warehouse_name,
    c."size",
    c.min_clusters,
    c.max_clusters,
    u.daily_credits,
    u.query_count,
    ROUND(u.avg_exec_ms, 0) as avg_exec_ms,
    u.slow_queries,
    CASE 
        WHEN u.slow_queries > u.query_count * 0.1 THEN 'SCALE_UP'
        WHEN u.daily_credits > 1000 THEN 'REVIEW_CONFIG'
        ELSE 'OK'
    END as recommendation
FROM wh_config c
LEFT JOIN wh_usage u ON c.warehouse_name = u.warehouse_name
ORDER BY c.warehouse_name;
```

### Warehouse Performance Troubleshooting

#### Slow Query Diagnosis

```sql
-- Find slow queries on warehouse
SELECT 
    query_id,
    query_text,
    warehouse_name,
    warehouse_size,
    start_time,
    total_elapsed_time,
    execution_time,
    queued_provisioning_time,
    compilation_time,
    bytes_scanned / (1024*1024*1024) as scanned_gb,
    bytes_produced / (1024*1024*1024) as produced_gb,
    rows_produced,
    ROUND(100.0 * execution_time / total_elapsed_time, 2) as execution_pct,
    CASE 
        WHEN bytes_spilled_to_remote_storage > 0 THEN 'SPILLING'
        WHEN bytes_scanned > bytes_produced * 1000 THEN 'FULL_SCAN'
        WHEN rows_produced = 0 THEN 'NO_RESULTS'
        ELSE 'CHECK_LOGIC'
    END as issue_type
FROM snowflake.account_usage.query_history
WHERE warehouse_name = 'analytical_wh'
    AND total_elapsed_time > 60000  -- > 60 seconds
    AND start_time >= DATEADD(day, -7, CURRENT_DATE())
ORDER BY total_elapsed_time DESC
LIMIT 20;

-- Analyze slow query root causes
SELECT 
    issue_type,
    COUNT(*) as count,
    AVG(total_elapsed_time) as avg_duration_ms,
    AVG(bytes_scanned / (1024*1024*1024)) as avg_scanned_gb
FROM (
    SELECT 
        CASE 
            WHEN bytes_spilled_to_remote_storage > 0 THEN 'MEMORY_PRESSURE'
            WHEN bytes_scanned > bytes_produced * 1000 THEN 'POOR_FILTERING'
            WHEN bytes_scanned > 100*1024*1024*1024 THEN 'LARGE_SCAN'
            WHEN execution_time < 1000 THEN 'QUEUE_HEAVY'
            ELSE 'COMPLEX_LOGIC'
        END as issue_type,
        total_elapsed_time,
        bytes_scanned,
        bytes_produced,
        bytes_spilled_to_remote_storage,
        execution_time
    FROM snowflake.account_usage.query_history
    WHERE warehouse_name = 'analytical_wh'
        AND start_time >= DATEADD(day, -7, CURRENT_DATE())
)
GROUP BY issue_type
ORDER BY avg_duration_ms DESC;
```

---

## Optimization Techniques

### Common Performance Issues and Solutions

#### Issue 1: High Bytes Scanned vs. Rows Produced

**Problem**: Query scans excessive data but produces few results.

```sql
-- Identify inefficient queries
SELECT 
    query_id,
    query_text,
    bytes_scanned / (1024*1024*1024) as scanned_gb,
    rows_produced,
    ROUND(bytes_scanned::numeric / NULLIF(rows_produced, 1), 2) as bytes_per_row,
    CASE 
        WHEN bytes_scanned > rows_produced * 1000000 THEN 'CRITICAL'
        WHEN bytes_scanned > rows_produced * 100000 THEN 'HIGH'
        ELSE 'NORMAL'
    END as efficiency
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_DATE())
    AND rows_produced > 0
    AND bytes_scanned > rows_produced * 100000
ORDER BY bytes_scanned DESC;
```

**Solution Roadmap**:

```sql
-- Step 1: Analyze query structure
-- Look for missing WHERE clauses or poor filtering

-- Example - Bad query:
SELECT * FROM large_fact_table;  -- Scans 100 GB

-- Step 2: Add selective WHERE clause
SELECT * FROM large_fact_table 
WHERE transaction_date >= '2024-01-01';  -- Scans 5 GB (if clustered)

-- Step 3: Use clustering key in predicate
-- Ensure predicate matches clustering key
ALTER TABLE large_fact_table CLUSTER BY (transaction_date);
SELECT * FROM large_fact_table 
WHERE transaction_date >= '2024-01-01';  -- Scans 5 GB with pruning

-- Step 4: Add search optimization for high-cardinality columns
ALTER TABLE large_fact_table ADD SEARCH OPTIMIZATION ON (customer_id);
SELECT * FROM large_fact_table 
WHERE customer_id = 'CUST_12345';  -- Scans 10 MB instead of 100 GB

-- Step 5: Materialize intermediate results
CREATE TABLE customer_transactions_2024 AS
SELECT * FROM large_fact_table 
WHERE transaction_date >= '2024-01-01';
-- Query smaller table for repeated queries
```

#### Issue 2: Warehouse Queuing

**Problem**: Queries waiting in queue due to insufficient warehouse capacity.

```sql
-- Diagnose queue patterns
SELECT 
    DATE_TRUNC('hour', start_time) as hour,
    warehouse_name,
    COUNT(*) as total_queries,
    COUNT(CASE WHEN queued_provisioning_time > 0 THEN 1 END) as queued_queries,
    AVG(queued_provisioning_time) as avg_queue_ms,
    MAX(queued_provisioning_time) as max_queue_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY queued_provisioning_time) as p95_queue_ms
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_DATE())
    AND queued_provisioning_time > 0
GROUP BY hour, warehouse_name
HAVING COUNT(*) > 10
ORDER BY hour DESC;
```

**Solutions by Scenario**:

```sql
-- Solution 1: Increase warehouse size
ALTER WAREHOUSE analytical_wh SET WAREHOUSE_SIZE = 'LARGE';
-- Cost: 2x increase, capacity: 2x increase
-- Benefit: 2x more query slots

-- Solution 2: Enable multi-cluster
ALTER WAREHOUSE analytical_wh SET 
    MAX_CLUSTER_COUNT = 3,
    SCALING_POLICY = 'AUTO';
-- Cost: Pay for active clusters only
-- Benefit: Scales automatically with demand

-- Solution 3: Separate workloads
CREATE WAREHOUSE interactive_wh WAREHOUSE_SIZE = 'SMALL';
CREATE WAREHOUSE batch_wh WAREHOUSE_SIZE = 'XLARGE';
-- Route interactive queries to interactive_wh
-- Route batch jobs to batch_wh
-- Benefit: No interference between workloads

-- Solution 4: Implement query timeouts
ALTER SESSION SET STATEMENT_TIMEOUT_IN_SECONDS = 3600;
-- Prevent runaway queries from blocking queue
-- Benefit: Better queue flow
```

#### Issue 3: Memory Pressure and Spilling

**Problem**: Warehouse lacks memory for query operations, spilling to disk.

```sql
-- Identify queries causing spilling
SELECT 
    query_id,
    query_text,
    warehouse_name,
    warehouse_size,
    execution_time,
    bytes_spilled_to_local_storage / (1024*1024) as local_spill_mb,
    bytes_spilled_to_remote_storage / (1024*1024) as remote_spill_mb,
    CASE 
        WHEN bytes_spilled_to_remote_storage > 0 THEN 'CRITICAL'
        WHEN bytes_spilled_to_local_storage > 1024 THEN 'HIGH'
        ELSE 'LOW'
    END as spill_severity
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_DATE())
    AND (bytes_spilled_to_local_storage > 0 OR bytes_spilled_to_remote_storage > 0)
ORDER BY bytes_spilled_to_remote_storage DESC;
```

**Root Cause Analysis and Solutions**:

```sql
-- Root Cause 1: Query produces large intermediate result
-- Solution: Add filtering before aggregation
-- Bad:
SELECT customer_id, SUM(amount) 
FROM large_fact_table 
GROUP BY customer_id;

-- Good:
SELECT customer_id, SUM(amount) 
FROM large_fact_table 
WHERE transaction_date >= '2024-01-01'  -- Filter first
GROUP BY customer_id;

-- Root Cause 2: Large join without optimization
-- Solution: Ensure join selectivity and order
-- Analyze join cardinality
SELECT 
    'table_a' as table_name,
    COUNT(*) as row_count,
    COUNT(DISTINCT join_key) as distinct_keys
FROM table_a
UNION ALL
SELECT 
    'table_b',
    COUNT(*),
    COUNT(DISTINCT join_key)
FROM table_b;

-- Root Cause 3: Warehouse undersized
-- Solution: Scale up temporarily
ALTER WAREHOUSE analytical_wh SET WAREHOUSE_SIZE = 'XLARGE';
-- Run problematic query
-- Monitor improvement
-- Adjust size if needed

-- Root Cause 4: Incorrect data type conversion
-- Solution: Use native types
-- Bad:
SELECT * FROM orders WHERE order_date::STRING = '2024-01-01';
-- Good:
SELECT * FROM orders WHERE order_date = '2024-01-01'::DATE;
```

#### Issue 4: Compilation Time Exceeds Execution Time

**Problem**: Query spends more time being compiled than executed.

```sql
-- Identify compilation-heavy queries
SELECT 
    query_id,
    query_text,
    compilation_time,
    execution_time,
    total_elapsed_time,
    ROUND(100.0 * compilation_time / total_elapsed_time, 2) as compilation_pct,
    CASE 
        WHEN compilation_time > execution_time THEN 'COMPILATION_HEAVY'
        WHEN compilation_time > execution_time * 0.5 THEN 'HIGH_COMPILE'
        ELSE 'NORMAL'
    END as pattern
FROM snowflake.account_usage.query_history
WHERE total_elapsed_time > 0
    AND start_time >= DATEADD(day, -7, CURRENT_DATE())
    AND pattern = 'COMPILATION_HEAVY'
ORDER BY compilation_time DESC;
```

**Solutions**:

```sql
-- Solution 1: Use parameterized queries / stored procedures
CREATE OR REPLACE PROCEDURE get_customer_orders(customer_id VARCHAR)
RETURNS TABLE(order_id INTEGER, amount DECIMAL)
LANGUAGE SQL
AS
$$
    SELECT order_id, amount FROM orders WHERE customer_id = ?;
$$;

-- Call multiple times (compiled once)
CALL get_customer_orders('CUST_001');
CALL get_customer_orders('CUST_002');
-- Compilation: 1x, Execution: 2x

-- Solution 2: Reduce query complexity
-- Break complex nested queries into CTEs or temp tables
-- CTEs with simple logic

-- Solution 3: Use result caching
-- Identical queries reuse cached results
SELECT COUNT(*) FROM large_table;  -- 500ms compile + 5000ms exec
SELECT COUNT(*) FROM large_table;  -- 0ms (cached), 5ms return

-- Solution 4: Optimize metadata access
-- Cache table statistics
ANALYZE TABLE orders;
-- Subsequent compilations use cached statistics
```

### Query Rewriting Techniques

#### Technique 1: Predicate Pushdown

```sql
-- Problem: Filter applied after expensive operation
-- Bad: Filter after join
SELECT a.*, b.value 
FROM large_table a 
JOIN reference_table b ON a.id = b.id 
WHERE b.status = 'ACTIVE';
-- Joins all rows, then filters

-- Good: Filter before join
SELECT a.*, b.value 
FROM large_table a 
JOIN (SELECT id, value FROM reference_table WHERE status = 'ACTIVE') b 
ON a.id = b.id;
-- Filters first, joins smaller set

-- Cost impact:
-- Bad: Join 1M × 50K rows = 50B comparisons, then filter to 10K
-- Good: Filter 50K to 5K, then join 1M × 5K = 5B comparisons
-- 10x improvement
```

#### Technique 2: Join Order Optimization

```sql
-- Problem: Join order affects memory and speed
-- Bad: Large table first
SELECT a.*, b.*, c.* 
FROM large_fact_table a           -- 100M rows
JOIN dimension_table_1 b ON ...   -- 10K rows
JOIN dimension_table_2 c ON ...;  -- 5K rows

-- Good: Small tables first (semi-join or broadcast)
SELECT a.*, b.*, c.* 
FROM dimension_table_2 c          -- 5K rows
JOIN dimension_table_1 b ON ...   -- 10K rows
JOIN large_fact_table a ON ...;   -- 100M rows

-- Memory usage:
-- Bad: Allocates memory for large table first
-- Good: Broadcasts small dimensions, minimal memory
```

#### Technique 3: Window Functions vs. Self-Join

```sql
-- Problem: Self-join for row comparison
-- Bad: Self-join
SELECT a.order_id, a.amount, b.amount as prev_amount
FROM orders a
LEFT JOIN orders b ON a.customer_id = b.customer_id 
    AND a.order_date > b.order_date
WHERE b.order_id = (
    SELECT order_id FROM orders o2 
    WHERE o2.customer_id = a.customer_id 
    ORDER BY order_date DESC LIMIT 1 OFFSET 1
);

-- Good: Window function
SELECT 
    order_id,
    amount,
    LAG(amount) OVER (PARTITION BY customer_id ORDER BY order_date) as prev_amount
FROM orders;

-- Performance:
-- Bad: Multiple self-joins, full scans
-- Good: Single pass with window aggregation
-- 10-100x faster
```

#### Technique 4: Semi-Join instead of Inner Join

```sql
-- Problem: Duplicate rows in result
-- Bad: Inner join (produces duplicates if multiple matches)
SELECT a.* 
FROM large_table a 
INNER JOIN reference_table b ON a.id = b.id
INNER JOIN reference_table c ON a.id = c.id;
-- Result: Cartesian product of matches

-- Good: Semi-join (test existence only)
SELECT a.* 
FROM large_table a 
WHERE a.id IN (SELECT id FROM reference_table)
    AND a.id IN (SELECT id FROM other_reference_table);

-- Or using EXISTS:
SELECT a.* 
FROM large_table a 
WHERE EXISTS (SELECT 1 FROM reference_table b WHERE a.id = b.id)
    AND EXISTS (SELECT 1 FROM other_reference_table c WHERE a.id = c.id);

-- Result size:
-- Bad: 1M rows × 5 matches × 10 matches = 50M rows
-- Good: 1M rows (deduplicated)
```

---

## Micro-Partition Pruning

### Understanding Micropartitions

**Micropartition**: Snowflake's unit of storage organization. Each contains 50-500 MB of compressed data with metadata tracking min/max values for each column.

#### Micropartition Metadata

```
Micropartition Structure:
┌─────────────────────────────────┐
│ Compressed Data (50-500 MB)     │
├─────────────────────────────────┤
│ Metadata:                       │
│ ├─ min(customer_id)             │
│ ├─ max(customer_id)             │
│ ├─ min(order_date)              │
│ ├─ max(order_date)              │
│ ├─ min(amount)                  │
│ ├─ max(amount)                  │
│ └─ ...                          │
└─────────────────────────────────┘

Benefits:
- Skip micropartitions that don't match WHERE clause
- Reduce bytes scanned to relevant micropartitions only
- Dramatically speeds up queries
```

### How Pruning Works

#### Pruning Mechanism

```sql
-- Example table with 100 micropartitions
-- Each micropartition: 1 GB
-- Total table size: 100 GB

-- Without pruning:
SELECT * FROM orders WHERE order_date = '2024-01-15';
-- Scans all 100 micropartitions
-- Bytes scanned: 100 GB
-- Time: 30 seconds

-- With pruning (clustering key on order_date):
SELECT * FROM orders WHERE order_date = '2024-01-15';
-- Micropartition metadata:
--   MP1: date range 2023-12-01 to 2023-12-15 (skip)
--   MP2: date range 2023-12-16 to 2023-12-31 (skip)
--   MP3: date range 2024-01-01 to 2024-01-31 (read)
--   ...
-- Scans only 1-2 micropartitions
-- Bytes scanned: 1-2 GB
-- Time: 0.5 seconds
-- 50x-100x improvement
```

#### Pruning Conditions

**Pruning occurs when WHERE clause can eliminate micropartitions**:

```sql
-- Pruning-eligible predicates:
WHERE customer_id = '12345'              -- Equality, pruning possible
WHERE order_date >= '2024-01-01'        -- Range, pruning possible
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31'  -- Range, pruning
WHERE status IN ('ACTIVE', 'PENDING')   -- List, pruning possible

-- Non-pruning predicates:
WHERE customer_id LIKE '%123%'           -- Pattern, hard to prune
WHERE EXTRACT(year FROM order_date) = 2024  -- Computed, no direct match
WHERE amount > 100 AND customer_id = 'X'    -- Only second part prunes
```

### Partition Pruning Analysis

#### Checking Pruning Effectiveness

```sql
-- Check if pruning is applied
SELECT 
    query_id,
    query_text,
    bytes_scanned,
    rows_produced,
    ROUND(bytes_scanned::numeric / NULLIF(rows_produced, 1), 2) as bytes_per_row,
    CASE 
        WHEN bytes_scanned < 1024*1024*100 THEN 'EXCELLENT_PRUNING'
        WHEN bytes_scanned < 1024*1024*1024 THEN 'GOOD_PRUNING'
        WHEN bytes_scanned < 10*1024*1024*1024 THEN 'FAIR_PRUNING'
        ELSE 'POOR_PRUNING'
    END as pruning_quality
FROM snowflake.account_usage.query_history
WHERE table_name = 'orders'
    AND start_time >= DATEADD(day, -7, CURRENT_DATE())
ORDER BY bytes_scanned DESC;
```

#### Micropartition-Level Analysis

```sql
-- Estimate micropartitions affected
SELECT 
    query_id,
    query_text,
    bytes_scanned,
    CEIL(bytes_scanned::numeric / (1024*1024*200)) as estimated_micropartitions_scanned,
    -- Assuming 200 MB per micropartition average
    CASE 
        WHEN CEIL(bytes_scanned::numeric / (1024*1024*200)) < 5 THEN 'TIGHT_PRUNING'
        WHEN CEIL(bytes_scanned::numeric / (1024*1024*200)) < 20 THEN 'GOOD_PRUNING'
        WHEN CEIL(bytes_scanned::numeric / (1024*1024*200)) < 100 THEN 'MODERATE_PRUNING'
        ELSE 'BROAD_PRUNING'
    END as pruning_level
FROM snowflake.account_usage.query_history
WHERE table_name = 'large_fact_table'
    AND start_time >= DATEADD(day, -7, CURRENT_DATE())
ORDER BY estimated_micropartitions_scanned;
```

### Optimizing for Partition Pruning

#### Strategy 1: Choose Pruning-Eligible Predicates

```sql
-- Compare pruning effectiveness
-- Query 1: Pruning eligible
SELECT * FROM orders 
WHERE order_date = '2024-01-15'
    AND customer_id = 'CUST_123';
-- Bytes scanned: 1 GB (excellent pruning)

-- Query 2: Partial pruning (only date prunes)
SELECT * FROM orders 
WHERE EXTRACT(year FROM order_date) = 2024
    AND customer_id = 'CUST_123';
-- Bytes scanned: 10 GB (poor pruning on date)

-- Query 3: No pruning
SELECT * FROM orders 
WHERE amount > 100;
-- Bytes scanned: 100 GB (no pruning)

-- Solution: Always use direct column predicates
SELECT * FROM orders 
WHERE order_date >= '2024-01-01'  -- Direct, prunable
    AND amount > 100;             -- Filters after pruning
```

#### Strategy 2: Clustering Key Alignment

```sql
-- Cluster by columns used in WHERE clauses
-- Bad: Cluster by non-filtered column
ALTER TABLE orders CLUSTER BY (id);
SELECT * FROM orders WHERE order_date = '2024-01-15';
-- No pruning benefit

-- Good: Cluster by filtered column
ALTER TABLE orders CLUSTER BY (order_date, customer_id);
SELECT * FROM orders WHERE order_date = '2024-01-15';
-- Excellent pruning

-- Verify pruning effectiveness after clustering
WITH before_cluster AS (
    SELECT 
        AVG(bytes_scanned) as avg_bytes_before
    FROM snowflake.account_usage.query_history
    WHERE table_name = 'orders'
        AND query_text ILIKE '%order_date%'
        AND start_time < '2024-01-01'
),
after_cluster AS (
    SELECT 
        AVG(bytes_scanned) as avg_bytes_after
    FROM snowflake.account_usage.query_history
    WHERE table_name = 'orders'
        AND query_text ILIKE '%order_date%'
        AND start_time >= '2024-01-01'
)
SELECT 
    b.avg_bytes_before,
    a.avg_bytes_after,
    ROUND(100.0 * (b.avg_bytes_before - a.avg_bytes_after) / b.avg_bytes_before, 2) as improvement_pct
FROM before_cluster b
CROSS JOIN after_cluster a;
```

#### Strategy 3: Composite Pruning

```sql
-- Use multiple columns in WHERE clause for better pruning
-- Bad: Single column filter
SELECT * FROM sales 
WHERE region = 'WEST';
-- Pruning removes 75% of data (4 regions)

-- Better: Multiple column filters
SELECT * FROM sales 
WHERE region = 'WEST'
    AND year = 2024
    AND product_category = 'ELECTRONICS';
-- Pruning removes 95% of data
-- Bytes scanned: 100 GB → 5 GB

-- Query design principle:
-- Structure WHERE clause to maximize pruning
-- Order by column cardinality (high first)
```

#### Strategy 4: Filtering Before Aggregation

```sql
-- Bad: Aggregate entire table
SELECT product_id, SUM(amount) 
FROM large_sales_table 
GROUP BY product_id;
-- Scans entire table, then aggregates

-- Good: Filter before aggregation
SELECT product_id, SUM(amount) 
FROM large_sales_table 
WHERE year = 2024
    AND region = 'WEST'
GROUP BY product_id;
-- Prunes data first, then aggregates smaller set
-- 10x-100x faster

-- Optimal: Cluster to maximize pruning
ALTER TABLE large_sales_table CLUSTER BY (year, region, product_id);
-- Now queries prune even more aggressively
```

---

## Monitoring and Alerting

### ACCOUNT_USAGE Views for Performance Monitoring

#### Query Performance Views

**QUERY_HISTORY**

```sql
-- Comprehensive query monitoring
SELECT 
    query_id,
    query_text,
    database_name,
    schema_name,
    warehouse_name,
    user_name,
    role_name,
    start_time,
    end_time,
    total_elapsed_time,
    execution_time,
    compilation_time,
    queued_provisioning_time,
    queued_repair_time,
    bytes_scanned,
    bytes_produced,
    rows_produced,
    rows_inserted,
    rows_updated,
    rows_deleted,
    bytes_spilled_to_local_storage,
    bytes_spilled_to_remote_storage,
    credits_used,
    error_code,
    error_message
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -30, CURRENT_DATE())
ORDER BY start_time DESC;

-- Key use cases:
-- 1. Identify slow queries
-- 2. Track credit consumption
-- 3. Diagnose spilling
-- 4. Monitor user activity
-- 5. Troubleshoot errors
```

**WAREHOUSE_METERING_HISTORY**

```sql
-- Track warehouse credit consumption
SELECT 
    start_time,
    end_time,
    warehouse_name,
    warehouse_size,
    credits_used,
    credits_used_compute,
    credits_used_cloud_services,
    DATEDIFF(minute, start_time, end_time) as duration_minutes,
    ROUND(credits_used / NULLIF(DATEDIFF(minute, start_time, end_time), 0), 4) as credits_per_minute
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD(month, -1, CURRENT_DATE())
ORDER BY start_time DESC;

-- Analyze cost by warehouse
SELECT 
    warehouse_name,
    SUM(credits_used) as total_credits,
    SUM(credits_used_compute) as compute_credits,
    SUM(credits_used_cloud_services) as cloud_services_credits,
    COUNT(DISTINCT DATE_TRUNC('day', start_time)) as active_days
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD(month, -1, CURRENT_DATE())
GROUP BY warehouse_name
ORDER BY total_credits DESC;
```

**LOAD_HISTORY**

```sql
-- Monitor data loading performance
SELECT 
    file_name,
    stage_name,
    status,
    row_count,
    row_parsed,
    file_size,
    first_error_message,
    first_error_line_number,
    first_error_character_position,
    last_load_time
FROM snowflake.account_usage.load_history
WHERE database_name = 'STAGING'
    AND last_load_time >= DATEADD(day, -7, CURRENT_DATE())
ORDER BY last_load_time DESC;

-- Identify loading issues
SELECT 
    stage_name,
    status,
    COUNT(*) as file_count,
    SUM(row_count) as total_rows,
    SUM(file_size) / (1024*1024*1024) as total_size_gb,
    COUNT(CASE WHEN status != 'LOADED' THEN 1 END) as failed_files
FROM snowflake.account_usage.load_history
WHERE last_load_time >= DATEADD(day, -7, CURRENT_DATE())
GROUP BY stage_name, status
ORDER BY total_size_gb DESC;
```

#### Resource Monitoring Views

**AUTOMATIC_CLUSTERING_HISTORY**

```sql
-- Monitor clustering activity and costs
SELECT 
    database_name,
    schema_name,
    table_name,
    start_time,
    end_time,
    DATEDIFF(minute, start_time, end_time) as duration_minutes,
    rows_inserted,
    rows_updated,
    bytes_reclustered / (1024*1024*1024) as bytes_reclustered_gb,
    CREDITS_USED
FROM snowflake.account_usage.automatic_clustering_history
WHERE start_time >= DATEADD(month, -1, CURRENT_DATE())
ORDER BY start_time DESC;

-- Calculate clustering ROI
WITH clustering_metrics AS (
    SELECT 
        table_name,
        SUM(CREDITS_USED) as total_clustering_credits,
        COUNT(*) as clustering_operations,
        SUM(bytes_reclustered) / (1024*1024*1024) as total_bytes_reclustered
    FROM snowflake.account_usage.automatic_clustering_history
    WHERE start_time >= DATEADD(month, -1, CURRENT_DATE())
    GROUP BY table_name
),
query_metrics AS (
    SELECT 
        table_name,
        COUNT(*) as query_count,
        AVG(bytes_scanned) as avg_bytes_scanned,
        SUM(CASE WHEN bytes_spilled_to_remote_storage = 0 THEN 1 ELSE 0 END) as non_spill_queries
    FROM snowflake.account_usage.query_history
    WHERE start_time >= DATEADD(month, -1, CURRENT_DATE())
    GROUP BY table_name
)
SELECT 
    c.table_name,
    c.total_clustering_credits,
    c.clustering_operations,
    c.total_bytes_reclustered,
    q.query_count,
    ROUND(q.avg_bytes_scanned / (1024*1024*1024), 2) as avg_bytes_scanned_gb,
    ROUND(100.0 * q.non_spill_queries / q.query_count, 2) as pct_no_spill,
    CASE 
        WHEN c.total_clustering_credits > (q.query_count * 0.05) THEN 'HIGH_COST'
        ELSE 'ACCEPTABLE_COST'
    END as cost_assessment
FROM clustering_metrics c
LEFT JOIN query_metrics q ON c.table_name = q.table_name
ORDER BY c.total_clustering_credits DESC;
```

**STORAGE_USAGE**

```sql
-- Monitor storage consumption
SELECT 
    usage_date,
    database_name,
    schema_name,
    table_name,
    active_bytes / (1024*1024*1024) as active_gb,
    time_travel_bytes / (1024*1024*1024) as time_travel_gb,
    fail_safe_bytes / (1024*1024*1024) as fail_safe_gb,
    retained_for_clone_bytes / (1024*1024*1024) as clone_gb
FROM snowflake.account_usage.storage_usage
WHERE usage_date >= DATEADD(month, -1, CURRENT_DATE())
ORDER BY usage_date DESC, active_bytes DESC;

-- Identify storage growth
WITH storage_trend AS (
    SELECT 
        usage_date,
        database_name,
        SUM(active_bytes) / (1024*1024*1024) as total_active_gb,
        LAG(SUM(active_bytes)) OVER (PARTITION BY database_name ORDER BY usage_date) as prev_active_bytes
    FROM snowflake.account_usage.storage_usage
    GROUP BY usage_date, database_name
)
SELECT 
    usage_date,
    database_name,
    total_active_gb,
    ROUND((total_active_gb - LAG(total_active_gb) OVER (PARTITION BY database_name ORDER BY usage_date)), 2) as daily_growth_gb,
    CASE 
        WHEN total_active_gb > 1000 THEN 'LARGE'
        WHEN total_active_gb > 100 THEN 'MEDIUM'
        ELSE 'SMALL'
    END as size_category
FROM storage_trend
WHERE usage_date >= DATEADD(month, -3, CURRENT_DATE())
ORDER BY usage_date DESC;
```

### INFORMATION_SCHEMA Views for Real-Time Monitoring

#### Detailed Performance Views

**INFORMATION_SCHEMA.TABLES**

```sql
-- Current table metadata
SELECT 
    table_catalog,
    table_schema,
    table_name,
    table_type,
    is_insertable_into,
    clustering_key,
    row_count,
    bytes,
    created,
    last_altered,
    last_data_changed
FROM information_schema.tables
WHERE table_schema = 'ANALYTICS'
ORDER BY bytes DESC;

-- Find tables without clustering keys (candidates for clustering)
SELECT 
    table_name,
    row_count,
    bytes / (1024*1024*1024) as size_gb,
    CASE 
        WHEN bytes > 10*1024*1024*1024 THEN 'HIGH_PRIORITY'
        WHEN bytes > 1024*1024*1024 THEN 'MEDIUM_PRIORITY'
        ELSE 'LOW_PRIORITY'
    END as clustering_priority
FROM information_schema.tables
WHERE table_schema = 'ANALYTICS'
    AND clustering_key IS NULL
    AND bytes > 1024*1024*1024  -- > 1 GB
ORDER BY bytes DESC;
```

**INFORMATION_SCHEMA.TABLE_STORAGE_METRICS**

```sql
-- Real-time table storage information
SELECT 
    table_name,
    active_bytes / (1024*1024*1024) as active_gb,
    time_travel_bytes / (1024*1024*1024) as time_travel_gb,
    fail_safe_bytes / (1024*1024*1024) as fail_safe_gb,
    retained_for_clone_bytes / (1024*1024*1024) as clone_gb,
    (active_bytes + time_travel_bytes + fail_safe_bytes) / (1024*1024*1024) as total_gb
FROM information_schema.table_storage_metrics
WHERE schema_name = 'ANALYTICS'
ORDER BY active_bytes DESC;
```

**INFORMATION_SCHEMA.QUERY_ACCELERATION_ELIGIBLE**

```sql
-- Queries eligible for Query Acceleration Service
SELECT *
FROM TABLE(
    INFORMATION_SCHEMA.QUERY_ACCELERATION_ELIGIBLE(
        WAREHOUSE_NAME => 'analytical_wh',
        ESTIMATED_ACCELERATION_BENEFIT => 2  -- 2x or better
    )
)
ORDER BY estimated_acceleration_benefit DESC
LIMIT 20;
```

### Resource Monitoring

#### Resource Monitor Setup

**Create Resource Monitor for Budget Control**

```sql
-- Create resource monitor
CREATE OR REPLACE RESOURCE MONITOR peak_hours_monitor WITH
    CREDIT_QUOTA = 1000
    FREQUENCY = HOURLY
    START_TIMESTAMP = '2024-01-01 08:00:00'
    END_TIMESTAMP = '2024-12-31 20:00:00'
    NOTIFY_USERS = ('analyst@company.com')
    TRIGGERS
        ON 75 PERCENT DO NOTIFY
        ON 100 PERCENT DO SUSPEND_IMMEDIATE;

-- Assign warehouse to monitor
ALTER WAREHOUSE analytical_wh SET RESOURCE_MONITOR = peak_hours_monitor;

-- Create different monitors for different purposes
CREATE RESOURCE MONITOR development_monitor WITH
    CREDIT_QUOTA = 100
    FREQUENCY = DAILY
    ON 100 PERCENT DO SUSPEND;

CREATE RESOURCE MONITOR production_monitor WITH
    CREDIT_QUOTA = 5000
    FREQUENCY = DAILY
    ON 100 PERCENT DO SUSPEND_IMMEDIATE;

-- Assign warehouses
ALTER WAREHOUSE dev_wh SET RESOURCE_MONITOR = development_monitor;
ALTER WAREHOUSE prod_wh SET RESOURCE_MONITOR = production_monitor;

-- Monitor resource usage
SELECT 
    warehouse_name,
    resource_monitor,
    credit_limit,
    used_credits,
    remaining_credits,
    ROUND(100.0 * used_credits / credit_limit, 2) as pct_used
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD(day, -1, CURRENT_DATE())
GROUP BY warehouse_name, resource_monitor, credit_limit
ORDER BY pct_used DESC;
```

#### Resource Monitor Monitoring

```sql
-- Monitor resource monitor activity
SELECT 
    resource_monitor_name,
    warehouse_name,
    credit_limit,
    credit_used_in_current_period,
    credit_rollover_allowance,
    ROUND(100.0 * credit_used_in_current_period / credit_limit, 2) as pct_used
FROM snowflake.account_usage.resource_monitors
ORDER BY pct_used DESC;

-- Alert on approaching limits
SELECT 
    resource_monitor_name,
    warehouse_name,
    credit_used_in_current_period,
    credit_limit,
    credit_limit - credit_used_in_current_period as remaining_credits
FROM snowflake.account_usage.resource_monitors
WHERE (credit_limit - credit_used_in_current_period) < (credit_limit * 0.25)
ORDER BY remaining_credits ASC;
```

### Alerts and Notifications

#### Email Alerts Setup

```sql
-- Create alert procedure for email notifications
CREATE OR REPLACE PROCEDURE send_performance_alert()
RETURNS STRING
LANGUAGE JAVASCRIPT
AS
$$
    var result = '';
    
    // Query for slow queries
    var stmt = snowflake.createStatement({
        sqlText: `
            SELECT COUNT(*) as slow_query_count
            FROM snowflake.account_usage.query_history
            WHERE start_time >= DATEADD(hour, -1, CURRENT_TIMESTAMP())
                AND total_elapsed_time > 300000  -- > 5 minutes
        `
    });
    
    var resultSet = stmt.execute();
    resultSet.next();
    var slowQueryCount = resultSet.getColumnValue(1);
    
    if (slowQueryCount > 5) {
        result = 'Alert: ' + slowQueryCount + ' slow queries detected in last hour';
        // Send email via notification
    }
    
    return result;
$$;

-- Schedule alert
CREATE OR REPLACE TASK performance_alert_task
    WAREHOUSE = compute_wh
    SCHEDULE = 'USING CRON 0 * * * * UTC'  -- Every hour
AS
    CALL send_performance_alert();

ALTER TASK performance_alert_task RESUME;
```

#### Native Alerts (Snowflake Native)

```sql
-- Create alert for slow queries
CREATE ALERT slow_query_alert
    WAREHOUSE = monitoring_wh
    CONDITION = (
        SELECT COUNT(*) 
        FROM snowflake.account_usage.query_history
        WHERE start_time >= DATEADD(hour, -1, CURRENT_TIMESTAMP())
            AND total_elapsed_time > 300000  -- > 5 minutes
    ) > 5
    ACTION = SEND MESSAGE TO @slack_channel
    'Alert: Multiple slow queries detected';

-- Create alert for high resource usage
CREATE ALERT high_credit_usage_alert
    WAREHOUSE = monitoring_wh
    CONDITION = (
        SELECT SUM(credits_used)
        FROM snowflake.account_usage.warehouse_metering_history
        WHERE start_time >= DATEADD(day, -1, CURRENT_DATE())
    ) > 1000
    ACTION = SEND EMAIL TO 'dba@company.com'
    'Alert: Daily credit usage exceeded 1000 credits';

-- Create alert for spilling
CREATE ALERT spilling_alert
    WAREHOUSE = monitoring_wh
    CONDITION = (
        SELECT COUNT(*)
        FROM snowflake.account_usage.query_history
        WHERE start_time >= DATEADD(hour, -1, CURRENT_TIMESTAMP())
            AND bytes_spilled_to_remote_storage > 0
    ) > 0
    ACTION = SEND MESSAGE TO @slack_channel
    'Alert: Remote spilling detected';
```

### Event Tables for Logging and Tracing

#### Event Table Setup

```sql
-- Create event table for query tracking
CREATE OR REPLACE EVENT TABLE query_audit_events;

-- Create table for manual logging
CREATE TABLE query_performance_log (
    log_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    query_id VARCHAR,
    query_text VARCHAR,
    warehouse_name VARCHAR,
    execution_time_ms INT,
    bytes_scanned_gb FLOAT,
    bytes_produced_gb FLOAT,
    status VARCHAR,
    notes VARCHAR
);

-- Insert events for monitoring
INSERT INTO query_performance_log
SELECT 
    CURRENT_TIMESTAMP(),
    query_id,
    query_text,
    warehouse_name,
    execution_time,
    bytes_scanned / (1024*1024*1024),
    bytes_produced / (1024*1024*1024),
    CASE WHEN error_code IS NULL THEN 'SUCCESS' ELSE 'ERROR' END,
    error_message
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(hour, -1, CURRENT_TIMESTAMP())
    AND (total_elapsed_time > 60000 OR error_code IS NOT NULL);

-- Query event table for analysis
SELECT 
    DATE_TRUNC('hour', log_timestamp) as hour,
    COUNT(*) as total_events,
    COUNT(CASE WHEN status = 'ERROR' THEN 1 END) as error_count,
    AVG(execution_time_ms) as avg_execution_ms,
    AVG(bytes_scanned_gb) as avg_scanned_gb
FROM query_performance_log
GROUP BY hour
ORDER BY hour DESC;
```

#### Comprehensive Logging Dashboard

```sql
-- Create materialized view for performance dashboard
CREATE OR REPLACE MATERIALIZED VIEW performance_dashboard_mv AS
SELECT 
    DATE_TRUNC('hour', start_time) as metric_hour,
    warehouse_name,
    COUNT(*) as query_count,
    COUNT(CASE WHEN queued_provisioning_time > 0 THEN 1 END) as queued_queries,
    AVG(total_elapsed_time) as avg_duration_ms,
    AVG(bytes_scanned) as avg_bytes_scanned,
    SUM(credits_used) as hourly_credits,
    COUNT(CASE WHEN bytes_spilled_to_remote_storage > 0 THEN 1 END) as spilling_queries,
    COUNT(CASE WHEN error_code IS NOT NULL THEN 1 END) as error_count
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(month, -1, CURRENT_DATE())
GROUP BY metric_hour, warehouse_name;

-- Query dashboard
SELECT *
FROM performance_dashboard_mv
ORDER BY metric_hour DESC
LIMIT 168;  -- Last 7 days (hourly)
```

---

## Troubleshooting Workflow

### Step-by-Step Troubleshooting Process

#### Phase 1: Symptom Identification

```
Symptom: "Queries are slow"

Step 1: Define "slow"
├─ Absolute: Query takes > 60 seconds
├─ Relative: Query now takes 2x longer than before
└─ SLA-based: Query misses 5-minute target

Step 2: Isolate scope
├─ All queries?
├─ Specific warehouse?
├─ Specific table?
└─ Specific time period?

Step 3: Gather baseline
├─ Historical performance
├─ Regression point
└─ Affected query patterns
```

#### Phase 2: Data Collection

```sql
-- Collect performance baseline
CREATE TEMPORARY TABLE slow_query_analysis AS
SELECT 
    query_id,
    query_text,
    warehouse_name,
    total_elapsed_time,
    execution_time,
    compilation_time,
    queued_provisioning_time,
    bytes_scanned / (1024*1024*1024) as scanned_gb,
    bytes_produced / (1024*1024*1024) as produced_gb,
    bytes_spilled_to_local_storage / (1024*1024) as local_spill_mb,
    bytes_spilled_to_remote_storage / (1024*1024) as remote_spill_mb,
    rows_produced,
    start_time
FROM snowflake.account_usage.query_history
WHERE query_text ILIKE '%problem_query%'
    AND start_time >= DATEADD(week, -4, CURRENT_DATE())
ORDER BY start_time DESC;

-- Compare periods
WITH before_change AS (
    SELECT 
        'BEFORE' as period,
        AVG(total_elapsed_time) as avg_duration,
        AVG(bytes_scanned) as avg_scanned,
        COUNT(*) as query_count
    FROM slow_query_analysis
    WHERE start_time < '2024-01-15'
),
after_change AS (
    SELECT 
        'AFTER' as period,
        AVG(total_elapsed_time) as avg_duration,
        AVG(bytes_scanned) as avg_scanned,
        COUNT(*) as query_count
    FROM slow_query_analysis
    WHERE start_time >= '2024-01-15'
)
SELECT * FROM before_change
UNION ALL
SELECT * FROM after_change;
```

#### Phase 3: Root Cause Analysis

```
Root Cause Checklist:

1. Query Logic Issues?
   ├─ Check WHERE clause
   ├─ Review JOIN conditions
   ├─ Analyze predicate pushdown
   └─ Verify table access

2. Schema/Data Issues?
   ├─ Check clustering depth
   ├─ Verify clustering key usage
   ├─ Review table size change
   └─ Confirm data distribution

3. Warehouse Issues?
   ├─ Check queue depth
   ├─ Review warehouse size
   ├─ Monitor spilling
   └─ Verify auto-suspend/resume

4. Cache Issues?
   ├─ Check result cache effectiveness
   ├─ Review warehouse cache lifecycle
   └─ Analyze metadata cache

5. System Issues?
   ├─ Check Snowflake status
   ├─ Review maintenance windows
   └─ Monitor cloud provider issues
```

#### Phase 4: Solution Implementation

```sql
-- Solution template

-- 1. Profile current query
SELECT /*+ NO_RESULT_CACHE */ * FROM [problematic_query]
-- (Check query profile)

-- 2. Identify bottleneck
-- (Use query profile and bytes_scanned analysis)

-- 3. Apply optimization based on root cause

-- Case 1: Poor filtering
ALTER TABLE target_table CLUSTER BY (filter_column);

-- Case 2: Warehouse too small
ALTER WAREHOUSE analytical_wh SET WAREHOUSE_SIZE = 'LARGE';

-- Case 3: No pruning
-- Rewrite query with clustering key predicate
SELECT * FROM target_table 
WHERE clustering_key = 'value';

-- Case 4: Spilling
-- Scale up or add filtering
ALTER WAREHOUSE analytical_wh SET WAREHOUSE_SIZE = 'XLARGE';

-- 4. Test improvement
SELECT ... FROM [optimized_query]
-- (Compare performance metrics)

-- 5. Monitor ongoing
-- (Set alerts and dashboard)
```

---

## Exam Tips and Practice Scenarios

### Key Concepts for Certification

**1. Clustering Depth Interpretation**
- Know what depth < 0.3, 0.3-0.6, > 0.8 means
- Understand remediation for poor clustering
- Calculate cost-benefit of reclustering

**2. Micropartition Pruning**
- Recognize pruning-eligible vs. non-eligible predicates
- Estimate micropartition reduction from WHERE clause
- Design queries to maximize pruning

**3. Performance Troubleshooting**
- Identify root causes (query logic, schema, warehouse, cache, system)
- Use ACCOUNT_USAGE and INFORMATION_SCHEMA views
- Apply appropriate solution tier (query → schema → warehouse → premium)

**4. Monitoring and Alerting**
- Use QUERY_HISTORY, WAREHOUSE_METERING_HISTORY for analysis
- Set up resource monitors and alerts
- Create event tables for audit trails

### Practice Scenarios

#### Scenario 1: Degrading Cluster Quality

```
Situation:
- Orders table (500 GB, 1B rows)
- Clustered by (order_date, customer_id)
- clustering_depth(order_date) = 0.25 (good)
- clustering_depth(order_date, customer_id) = 0.82 (poor)
- Queries are slowing down over time

Questions:
1. What's the problem?
2. What caused it?
3. How would you investigate?
4. What solutions would you recommend?

Answer:
1. Secondary clustering key (customer_id) degraded, multi-key pruning failing
2. Random customer_id insertion pattern, auto-clustering can't keep up
3. Check automatic_clustering_history for maintenance activity
4. Options:
   - Disable customer_id from clustering key (keep only date)
   - Increase auto-clustering budget
   - Materialize common queries
   - Archive old data to reset clustering
```

#### Scenario 2: Query Queuing Crisis

```
Situation:
- analytical_wh: MEDIUM (4 credits/min)
- Peak hours: 8 AM - 6 PM
- Queue depth: 50+ queries by 10 AM
- Queued_provisioning_time: 5-15 minutes
- Daily credit cost: 5,000 credits
- Monthly budget: 50,000 credits

Questions:
1. What's the immediate impact?
2. What are three solution options?
3. What's the cost of each option?
4. Which would you recommend?

Answer:
1. Users experiencing 5-15 min delays, SLA violations
2. Options:
   a) Scale up to LARGE: 2x capacity, 2x cost (8 credits/min)
   b) Multi-cluster (1-3): AUTO scaling, ~1.5x cost
   c) Separate workloads: Create interactive_wh (SMALL) + batch_wh (LARGE)
3. Costs:
   a) LARGE: 8 cr/min × 10 hrs × 20 days = 9,600 cr/month
   b) Multi-cluster avg 2 clusters: 8 cr/min × 10 hrs × 20 days = 9,600 cr/month
   c) Separate (1 SMALL + 1 LARGE): 2+8 = 10 cr/min avg = 9,600 cr/month
4. Recommend option (b) multi-cluster because:
   - Scales automatically with demand
   - Lower cost during off-peak
   - Balances cost and performance
```

#### Scenario 3: Spilling in Aggregation

```
Situation:
Query: SELECT customer_id, SUM(amount) FROM sales GROUP BY customer_id
- Table: 1 TB, 1B rows
- Warehouse: MEDIUM (8 GB per node)
- Execution time: 120 seconds
- bytes_scanned: 500 GB
- bytes_spilled_to_remote_storage: 200 GB
- Bytes spilled > bytes scanned (200 > 500 is false, but 200 GB is significant)

Questions:
1. What's causing spilling?
2. Why is execution so slow?
3. What solutions would help most?
4. Rank solutions by ROI

Answer:
1. Group by memory requirement > available warehouse memory
   - 1B unique customer_ids × 8 bytes overhead = 8+ GB minimum
   - With aggregation state = 15-20 GB
   - MEDIUM warehouse = 8 GB per node, insufficient
2. Remote spilling (100x slower than in-memory)
   - 200 GB to remote storage = significant latency
3. Solutions:
   a) Add WHERE clause (filter before GROUP BY)
   b) Pre-aggregate in materialized view
   c) Scale up warehouse (LARGE = 16 GB per node)
   d) Pre-partition data by customer_id
4. ROI ranking:
   a) WHERE filter: 0 cost, 50% improvement if applicable
   b) Materialized view: 100 credits setup, 80% improvement
   c) Scale up: 1000 credits/month, 60% improvement
   d) Pre-partition: 500 credits + schema change, 70% improvement
```

#### Scenario 4: Query Profile Interpretation

```
Situation:
Query Profile shows:
- TableScan [0]: 80 seconds (83% of 96s total)
  └─ Bytes Scanned: 100 GB
  └─ Rows Produced: 10M
  
- Filter [1]: 10 seconds (10% of 96s total)
  └─ Rows Produced: 1M
  
- Aggregate [2]: 6 seconds (6% of 96s total)
  └─ Rows Produced: 500

Questions:
1. What's the main bottleneck?
2. What are the root causes?
3. What would you try first?
4. How would you measure success?

Answer:
1. TableScan dominates (80 seconds, 83%)
2. Root causes:
   - Scans 100 GB to produce 10M rows (efficiency = 100GB/10M rows = 10KB/row)
   - No clustering key applied
   - Filter applied after scan (should push down)
3. Try first (by priority):
   a) Add WHERE clause with clustering key
   b) Check if clustering key exists, verify usage
   c) Rewrite to filter before table scan (impossible in this query structure)
   d) Implement search optimization on frequently filtered column
4. Success metrics:
   - bytes_scanned < 10 GB (90% reduction)
   - execution_time < 10 seconds (8x improvement)
   - rows produced unchanged (10M)
```

### Quick Reference: Troubleshooting Decision Tree

```
Performance Issue Detected
│
├─ Slow Query (> 60 sec)?
│  ├─ High bytes_scanned?
│  │  ├─ Check clustering_depth
│  │  │  ├─ > 0.6? → Fix clustering or add predicate
│  │  │  └─ < 0.3? → Check query logic
│  │  └─ Check bytes_scanned/rows_produced ratio
│  │     ├─ Very high? → Missing WHERE clause
│  │     └─ Normal? → Check for spilling
│  │
│  ├─ High spilling (remote)?
│  │  ├─ Scale warehouse first
│  │  ├─ Then add filtering
│  │  └─ Then optimize query logic
│  │
│  ├─ High compilation time?
│  │  ├─ Use stored procedures
│  │  ├─ Simplify query logic
│  │  └─ Reduce query complexity
│  │
│  └─ In queue (queued_provisioning_time high)?
│     ├─ Scale warehouse or multi-cluster
│     ├─ Or separate workloads
│     └─ Or schedule batch jobs off-peak
│
├─ High Credits Used?
│  ├─ Check warehouse_metering_history
│  ├─ Identify expensive queries
│  ├─ Evaluate auto-clustering cost
│  └─ Consider query optimization
│
└─ High Storage?
   ├─ Check table_storage_metrics
   ├─ Identify large tables
   ├─ Archive old data
   └─ Review time travel settings
```

---

## Best Practices Summary

### Monitoring Best Practices

1. **Regular Baseline Reviews**: Establish performance baselines monthly
2. **Alert Configuration**: Set alerts at 75% thresholds for budgets
3. **Dashboard Creation**: Build materialized views for key metrics
4. **Trend Analysis**: Monitor clustering depth, queue depth, spilling trends
5. **Root Cause Analysis**: Don't just treat symptoms, find underlying causes

### Optimization Best Practices

1. **Query Logic First**: Start with WHERE clause optimization
2. **Schema Second**: Then implement clustering/search optimization
3. **Warehouse Third**: Finally resize warehouse if needed
4. **Avoid Premium Services**: Use QAS/Snowpark only when justified
5. **Measure Impact**: Always compare before/after metrics

### Documentation Best Practices

1. **Log All Changes**: Document why optimization decisions were made
2. **Track Metrics**: Keep historical data for trend analysis
3. **Share Learnings**: Document successful patterns for reuse
4. **Maintain Playbooks**: Create runbooks for common issues
5. **Review Regularly**: Revisit optimization assumptions quarterly

---

**Study Tips**:
- Run actual queries against ACCOUNT_USAGE and INFORMATION_SCHEMA
- Profile sample queries and interpret the results
- Set up test alerts and monitors
- Create test tables and experiment with clustering
- Practice the troubleshooting workflow on sample scenarios

