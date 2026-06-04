# Snowflake Performance Tools and Best Practices
## Advanced Certification Study Guide

---

## Table of Contents
1. [Query Profiling](#query-profiling)
2. [Virtual Warehouse Configurations](#virtual-warehouse-configurations)
3. [Clustering](#clustering)
4. [Search Optimization Service](#search-optimization-service)
5. [Caching](#caching)
6. [Best Practices Summary](#best-practices-summary)

---

## Query Profiling

### Overview
Query profiling is the foundation of performance optimization in Snowflake. It provides detailed insights into how queries execute, where time is spent, and where bottlenecks exist.

### Query Profile Interpretation

#### Accessing Query Profile
```sql
-- View query profile in the UI under "Query Details"
-- Or programmatically:
SELECT * FROM TABLE(RESULT_SCAN('<QUERY_ID>'));

-- Get query execution details
SELECT * FROM INFORMATION_SCHEMA.QUERY_HISTORY 
WHERE QUERY_ID = '<QUERY_ID>';
```

#### Key Components of Query Profile

**1. Profile Structure**
- **Operator Tree**: Visual representation of query execution plan
- **Execution Timeline**: Shows sequential operation execution
- **Statistics**: Duration, rows processed, data scanned, memory usage
- **Performance Insights**: Automatic recommendations based on detected patterns

**2. Profile Metrics to Analyze**

| Metric | Definition | What to Look For |
|--------|-----------|------------------|
| Duration | Total query execution time | High values indicate optimization opportunity |
| Rows Produced | Output rows from operation | Compare to input rows to identify filtering efficiency |
| Bytes Scanned | Amount of data examined | High values indicate poor pruning or large tables |
| Bytes Spilled | Data moved to disk due to memory pressure | Indicates warehouse is undersized |
| Execution Time | Time spent in actual processing | Excludes queuing and compilation |

#### Common Query Profile Scenarios

**Scenario 1: Table Scan Bottleneck**
```
Profile shows:
- High "Bytes Scanned" with many rows produced
- No clustering key applied
- Likely missing WHERE clause filtering

Recommendation:
- Add WHERE clause with clustering key predicate
- Implement or adjust clustering key
- Consider partitioning strategy
```

**Scenario 2: Join Performance Issues**
```
Profile shows:
- High rows produced from JOIN operations
- Memory pressure indicators
- Uneven data distribution

Recommendation:
- Review join order and selectivity
- Check for cross joins inadvertently created
- Ensure join keys have appropriate statistics
- Consider rewriting join logic
```

**Scenario 3: Aggregation Bottleneck**
```
Profile shows:
- Large number of rows into GROUP BY
- High bytes spilled
- Long execution time for aggregation

Recommendation:
- Add filtering before aggregation
- Increase warehouse size (vertical scaling)
- Consider approximate counts for exploratory queries
- Pre-aggregate if possible
```

### Identifying Bottlenecks

#### Method 1: Sequential Analysis
1. **Start from top**: Examine root operator first
2. **Move downward**: Follow the operator tree to leaf nodes
3. **Calculate time distribution**: Identify which operators consume most time
4. **Correlate metrics**: Match high duration with resource metrics (bytes scanned, spilled)

#### Method 2: Critical Path Analysis
```
Time Distribution Analysis:
- Compilation Time: Time to parse/optimize query
- Queuing Time: Waiting for warehouse resources
- Execution Time: Actual data processing

Focus optimization efforts on largest segment
```

#### Method 3: Metric Anomalies
- **Disproportionate bytes scanned to rows produced**: Inefficient filtering
- **High memory/spilled bytes**: Undersized warehouse or expensive operations
- **Long compilation time**: Complex query structure, missing statistics
- **Data skew**: Uneven distribution across worker nodes

### Performance Recommendations Framework

#### Four-Tier Recommendation Approach

**Tier 1: Query Logic Optimization** (Cheapest)
- Reorder joins to filter early
- Push predicates down the execution tree
- Use semi-joins instead of inner joins when possible
- Eliminate unnecessary columns

```sql
-- Bad: Full table join, filter after
SELECT a.id, b.value 
FROM large_table a 
JOIN reference_table b ON a.id = b.id 
WHERE b.status = 'ACTIVE'

-- Better: Filter before join
SELECT a.id, b.value 
FROM large_table a 
JOIN (SELECT id, value FROM reference_table WHERE status = 'ACTIVE') b 
ON a.id = b.id
```

**Tier 2: Schema Optimization** (Moderate cost)
- Add or adjust clustering keys
- Implement search optimization service
- Create materialized views for repeated aggregations
- Archive old data

**Tier 3: Warehouse Right-Sizing** (Cost trade-off)
- Increase warehouse size for memory-bound queries
- Use multi-cluster warehouses for concurrency
- Implement auto-suspend for idle warehouses

**Tier 4: Query Acceleration Service** (Premium)
- Use for queries with complex analytics and high latency
- Benefits dependent on query patterns

---

### Metadata Functions for Performance Analysis

#### System Functions for Query Analysis

**QUERY_HISTORY()**
```sql
-- Get comprehensive query history
SELECT 
    query_id,
    query_text,
    database_name,
    warehouse_name,
    total_elapsed_time,
    queued_provisioning_time,
    queued_repair_time,
    bytes_scanned,
    bytes_produced,
    rows_produced,
    compilation_time,
    execution_time,
    bytes_spilled_to_local_storage,
    bytes_spilled_to_remote_storage
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD(day, -7, CURRENT_DATE())
ORDER BY start_time DESC
LIMIT 100;
```

**QUERY_ACCELERATION_ELIGIBLE()**
```sql
-- Identify queries that could benefit from QAS
SELECT * FROM TABLE(
    INFORMATION_SCHEMA.QUERY_ACCELERATION_ELIGIBLE(
        WAREHOUSE_NAME => 'MY_WAREHOUSE',
        ESTIMATED_ACCELERATION_BENEFIT => 3
    )
);
```

**KEY_COLUMNS_FOR_CLUSTERING()**
```sql
-- Identify best clustering candidates
SELECT 
    column_name,
    clustering_depth
FROM SYSTEM$CLUSTERING_DEPTH('table_name');

-- Returns scoring of column effectiveness for clustering
```

**TABLE_STORAGE_METRICS()**
```sql
-- Monitor table size and growth
SELECT 
    table_id,
    table_name,
    active_bytes,
    time_travel_bytes,
    fail_safe_bytes,
    retained_for_clone_bytes
FROM INFORMATION_SCHEMA.TABLE_STORAGE_METRICS
WHERE schema_name = 'ANALYTICS'
ORDER BY active_bytes DESC;
```

#### Key Metadata Views

| View | Purpose | Use Case |
|------|---------|----------|
| QUERY_HISTORY | Query execution details | Performance analysis, cost tracking |
| TABLE_STORAGE_METRICS | Table sizes and retention | Capacity planning, cost optimization |
| STAGE_DIRECTORY_TABLE_USAGE | Stage file metrics | Data loading optimization |
| WAREHOUSE_METERING_HISTORY | Credit consumption | Cost allocation, budget planning |
| LOAD_HISTORY | Data loading statistics | ETL monitoring, performance tuning |
| COPY_HISTORY | COPY command performance | Data loading optimization |

---

### Warehouse Queuing

#### Understanding Queue Mechanics

**Queue Formation**
- Occurs when submitted queries exceed warehouse capacity
- Separate queues per warehouse, not per schema/database
- FIFO (First In, First Out) within same priority level
- Priority levels affect queue order

#### Queue Time Components

```
Total Elapsed Time = 
  Compilation Time + 
  Queuing Time + 
  Execution Time
```

#### Causes of Excessive Queuing

1. **Warehouse Size Too Small**: Cannot handle concurrent workload
2. **Insufficient Credit Budget**: Resource constraints
3. **Long-Running Queries**: Blocking other queries
4. **Burst Workloads**: Sudden increase in concurrent users
5. **Multi-Cluster Configuration Issue**: Scaling policy not optimized

#### Diagnosing Queue Issues

```sql
-- Query to identify queuing problems
SELECT 
    query_id,
    query_text,
    warehouse_name,
    queued_provisioning_time,
    queued_repair_time,
    execution_time,
    total_elapsed_time,
    CASE 
        WHEN queued_provisioning_time > 0 THEN 'Warehouse full'
        WHEN queued_repair_time > 0 THEN 'Warehouse restarting'
        ELSE 'No queue detected'
    END as queue_type
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE warehouse_name = 'MY_WAREHOUSE'
    AND start_time >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
    AND (queued_provisioning_time > 0 OR queued_repair_time > 0)
ORDER BY start_time DESC;
```

#### Queue Mitigation Strategies

**Strategy 1: Increase Warehouse Size**
```sql
ALTER WAREHOUSE my_warehouse SET WAREHOUSE_SIZE = 'LARGE';
-- Cost increase: 2x (from MEDIUM to LARGE)
-- Benefit: 2x query slots available
```

**Strategy 2: Implement Multi-Cluster Warehouses**
```sql
ALTER WAREHOUSE my_warehouse SET 
    MAX_CLUSTER_COUNT = 3,
    MIN_CLUSTER_COUNT = 1,
    SCALING_POLICY = 'AUTO';
-- Auto-adds clusters when queue depth > 0
-- Auto-removes clusters after 2-3 minutes idle
```

**Strategy 3: Separate Workloads**
```sql
-- Create separate warehouses for different workload patterns
CREATE WAREHOUSE interactive_wh WAREHOUSE_SIZE = 'SMALL';
CREATE WAREHOUSE batch_wh WAREHOUSE_SIZE = 'LARGE';
CREATE WAREHOUSE reporting_wh WAREHOUSE_SIZE = 'MEDIUM';

-- Route queries to appropriate warehouse
USE WAREHOUSE interactive_wh;
```

**Strategy 4: Query Prioritization**
```sql
-- Use resource monitors and query acceleration
CREATE RESOURCE MONITOR peak_monitor WITH 
    CREDIT_QUOTA = 1000
    FREQUENCY = HOURLY
    TRIGGER ON 100 PERCENT DO SUSPEND_IMMEDIATE;

-- Set up query timeout for runaway queries
ALTER SESSION SET STATEMENT_TIMEOUT_IN_SECONDS = 3600;
```

---

### Warehouse Spilling

#### Understanding Spilling

**Definition**: When a query requires more memory than available in the warehouse, intermediate results are written to disk (local SSD or remote cloud storage).

#### Spilling Categories

**1. Local Spilling**
- Writes to SSD on warehouse node
- Fast but limited by SSD capacity
- Indicates tight memory conditions
- Example: 10GB query on 32GB warehouse with other queries

**2. Remote Spilling**
- Writes to Snowflake's object storage
- Much slower than local spilling
- Indicates severe memory pressure
- Dramatic performance impact (100x+ slower than in-memory)

#### Spilling Impact Analysis

```
Memory Hierarchy by Speed:
1. Warehouse RAM (100x100): Nanoseconds - In-memory operations
2. Local SSD (10-100x): Microseconds - Local spilling
3. Remote Storage (1x): Milliseconds - Remote spilling
   └─ Network latency + object storage access

Impact Example:
- In-memory join: 100ms
- Local SSD spill: 1-2 seconds
- Remote spill: 30+ seconds
```

#### Detecting Spilling

```sql
-- Method 1: Query Profile
-- Look for "Bytes Spilled to Local/Remote Storage" in UI

-- Method 2: Query History
SELECT 
    query_id,
    query_text,
    warehouse_name,
    warehouse_size,
    bytes_scanned,
    bytes_produced,
    bytes_spilled_to_local_storage,
    bytes_spilled_to_remote_storage,
    execution_time,
    CASE 
        WHEN bytes_spilled_to_remote_storage > 0 THEN 'CRITICAL'
        WHEN bytes_spilled_to_local_storage > 1000000000 THEN 'SEVERE'
        WHEN bytes_spilled_to_local_storage > 0 THEN 'MODERATE'
        ELSE 'OK'
    END as spill_severity
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD(day, -1, CURRENT_DATE())
ORDER BY bytes_spilled_to_remote_storage DESC;

-- Method 3: Monitor in real-time
SELECT 
    warehouse_name,
    SUM(bytes_spilled_to_local_storage) as total_local_spill,
    SUM(bytes_spilled_to_remote_storage) as total_remote_spill,
    COUNT(*) as query_count
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD(hour, -1, CURRENT_TIMESTAMP())
    AND warehouse_name = 'MY_WAREHOUSE'
GROUP BY warehouse_name;
```

#### Root Causes of Spilling

| Cause | Symptom | Solution |
|-------|---------|----------|
| Undersized warehouse | Consistent local spilling on normal queries | Scale up warehouse |
| Large result set | Bytes spilled > bytes scanned | Filter data earlier, reduce output columns |
| Heavy aggregations | Spilling in GROUP BY operations | Increase warehouse size or pre-filter data |
| Large joins | Remote spilling in JOIN operations | Optimize join order, reduce result set |
| Lack of filtering | High bytes scanned, unnecessary data loaded | Add WHERE clauses with clustering key predicates |
| Memory-intensive functions | UDFs causing spilling | Review UDF logic, consider native functions |

#### Spilling Resolution Roadmap

**Immediate Actions (Quick wins)**
1. Add WHERE clause filtering to reduce data volume
2. Remove unnecessary columns from SELECT
3. Push predicates down through CTEs
4. Materialize intermediate results

**Short-term Actions (Days)**
1. Scale up warehouse size (temporary solution)
2. Add or adjust clustering keys
3. Implement search optimization service
4. Rewrite query logic for better join order

**Long-term Actions (Weeks)**
1. Archive historical data
2. Redesign schema for better performance
3. Implement incremental loading for large tables
4. Evaluate data modeling approach

---

## Virtual Warehouse Configurations

### Auto-Suspend and Auto-Resume

#### Auto-Suspend Behavior

**Definition**: Automatically suspends (pauses) warehouse after specified idle time to reduce credit consumption.

```sql
-- Default: Auto-suspend after 10 minutes
CREATE WAREHOUSE demo_wh 
    WAREHOUSE_SIZE = 'SMALL'
    AUTO_SUSPEND = 600;  -- Seconds (10 minutes)

-- Disable auto-suspend (NOT recommended for cost optimization)
ALTER WAREHOUSE demo_wh SET AUTO_SUSPEND = NULL;

-- Aggressive auto-suspend (2 minutes idle)
ALTER WAREHOUSE demo_wh SET AUTO_SUSPEND = 120;
```

#### Cost Implications

```
Credit Consumption Model:
- Active warehouse: 1 credit per minute (for SMALL size)
- Suspended warehouse: 0 credits
- Resuming warehouse: 60-90 seconds startup time

Example:
SMALL warehouse with 10 interactive users
- AUTO_SUSPEND = 60 seconds: 0.6 credits/hour idle time
- AUTO_SUSPEND = 600 seconds: 6 credits/hour idle time
- AUTO_SUSPEND = NULL: Continuous consumption

Best practice: Set to 300-600 seconds for interactive workloads
```

#### Auto-Resume Behavior

**Definition**: Automatically starts suspended warehouse when query submitted.

```sql
-- Resume on query submission
-- Happens automatically, no configuration needed
-- Startup time: 60-90 seconds typically

-- Strategy: Disable for dev/test, enable for production
ALTER WAREHOUSE dev_wh SET AUTO_RESUME = FALSE;  -- Keep control
ALTER WAREHOUSE prod_wh SET AUTO_RESUME = TRUE;   -- Auto-recovery
```

#### Optimization Scenarios

**Scenario 1: Development Environment**
```sql
CREATE WAREHOUSE dev_wh 
    WAREHOUSE_SIZE = 'XSMALL'
    AUTO_SUSPEND = 120           -- Quick suspend
    AUTO_RESUME = TRUE;          -- Resume on demand
```

**Scenario 2: Scheduled Batch Processing**
```sql
CREATE WAREHOUSE batch_wh 
    WAREHOUSE_SIZE = 'LARGE'
    AUTO_SUSPEND = 60            -- Don't leave hanging
    INITIALLY_SUSPENDED = TRUE;  -- Start suspended
```

**Scenario 3: 24/7 Interactive Workload**
```sql
CREATE WAREHOUSE interactive_wh 
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND = NULL          -- Keep running
    AUTO_RESUME = TRUE;          -- Ensure availability
```

---

### Scale Up/Down (Resizing)

#### Warehouse Sizing Basics

**Size Categories** (Credit multipliers)
```
XSMALL    = 1 credit/minute
SMALL     = 2 credits/minute
MEDIUM    = 4 credits/minute
LARGE     = 8 credits/minute
XLARGE    = 16 credits/minute
2XLARGE   = 32 credits/minute
3XLARGE   = 64 credits/minute
4XLARGE   = 128 credits/minute
```

#### Resizing Mechanics

```sql
-- Resize warehouse (immediately effective for new queries)
ALTER WAREHOUSE analytical_wh SET WAREHOUSE_SIZE = 'LARGE';

-- Running queries continue on original size
-- Newly submitted queries use new size
-- Suspension/resumption NOT triggered
```

#### When to Scale Up

**Decision Criteria**
1. **High Bytes Spilled**: Remote spilling detected in queries
2. **Memory Pressure**: Consistent memory utilization > 80%
3. **Query Timeouts**: Queries hitting timeout limits
4. **Queuing Issues**: Persistent queue depth > 0
5. **User Complaints**: Performance degradation reported

#### When to Scale Down

**Cost Optimization Indicators**
1. **Low Utilization**: CPU, memory < 30% consistently
2. **No Spilling**: Zero bytes spilled in queries
3. **Meeting SLA**: Queries completing within time targets
4. **Reduced Concurrency**: Fewer concurrent users expected

#### Resizing Analysis Framework

```sql
-- Analyze warehouse utilization to determine optimal size
WITH query_stats AS (
    SELECT 
        warehouse_name,
        warehouse_size,
        query_id,
        bytes_spilled_to_remote_storage,
        bytes_spilled_to_local_storage,
        execution_time,
        queued_provisioning_time
    FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE warehouse_name = 'analytical_wh'
        AND start_time >= DATEADD(day, -7, CURRENT_DATE())
)
SELECT 
    warehouse_name,
    COUNT(*) as query_count,
    COUNT(CASE WHEN bytes_spilled_to_remote_storage > 0 THEN 1 END) as remote_spill_count,
    COUNT(CASE WHEN bytes_spilled_to_local_storage > 0 THEN 1 END) as local_spill_count,
    AVG(execution_time) as avg_execution_ms,
    MAX(execution_time) as max_execution_ms,
    SUM(queued_provisioning_time) as total_queue_time_ms,
    CASE 
        WHEN COUNT(CASE WHEN bytes_spilled_to_remote_storage > 0 THEN 1 END) > query_count * 0.1 
            THEN 'SCALE UP'
        WHEN MAX(execution_time) > 300000 
            THEN 'CONSIDER SCALE UP'
        WHEN AVG(execution_time) < 5000 AND SUM(queued_provisioning_time) = 0 
            THEN 'SCALE DOWN'
        ELSE 'OPTIMAL'
    END as recommendation
FROM query_stats
GROUP BY warehouse_name, warehouse_size;
```

#### Cost-Performance Trade-off

```
Resizing Decision Matrix:

Warehouse Size | Credits/Min | Best For
XSMALL        | 1           | Development, testing, single-user
SMALL         | 2           | Light analytics, infrequent queries
MEDIUM        | 4           | Standard analytics, typical workload
LARGE         | 8           | Heavy analytics, concurrent users
XLARGE+       | 16+         | Enterprise scale, real-time analytics

Example: 10 hours batch job
- MEDIUM (4 cr/min): 10 × 60 × 4 = 2,400 credits
- LARGE (8 cr/min): 10 × 60 × 8 = 4,800 credits (2x cost)
- But may complete in 6 hours: 6 × 60 × 8 = 2,880 credits

ROI: Check if reduced runtime justifies increased cost
```

---

### Scale In/Out (Multi-Cluster Warehouse and Auto-Scaling)

#### Multi-Cluster Warehouse Concept

**Definition**: Logical warehouse that spans multiple physical clusters, automatically adding/removing clusters based on demand.

```sql
-- Create multi-cluster warehouse
CREATE WAREHOUSE multi_cluster_wh 
    WAREHOUSE_SIZE = 'MEDIUM'
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 4
    SCALING_POLICY = 'AUTO';

-- Or enable on existing warehouse
ALTER WAREHOUSE analytical_wh SET 
    MAX_CLUSTER_COUNT = 3,
    MIN_CLUSTER_COUNT = 1,
    SCALING_POLICY = 'AUTO';
```

#### Scaling Policies

**1. AUTO Scaling Policy**
```
Behavior:
- Adds cluster when queue depth > 0 for 1 minute
- Removes cluster after idle for 2-3 minutes
- Respects MIN/MAX cluster boundaries
- Best for variable concurrency workloads

Use Case: Interactive analytics, BI tools
```

**2. ECONOMY Scaling Policy**
```
Behavior:
- Adds cluster when queue depth > 0 for 3 minutes
- Removes cluster after idle for 5-6 minutes
- Delays cluster addition to avoid short bursts
- Optimizes credit cost over speed

Use Case: Batch processing, less time-sensitive work
```

#### Multi-Cluster Configuration Examples

**Example 1: Interactive BI Dashboard**
```sql
CREATE WAREHOUSE bi_analytics_wh 
    WAREHOUSE_SIZE = 'MEDIUM'
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 4
    SCALING_POLICY = 'AUTO'
    AUTO_SUSPEND = 300
    AUTO_RESUME = TRUE;

-- Starts with 1 cluster
-- Adds up to 3 more clusters if queue detected
-- Saves cost by scaling down quickly
-- Responsive for user queries
```

**Example 2: Scheduled Batch Processing**
```sql
CREATE WAREHOUSE batch_processing_wh 
    WAREHOUSE_SIZE = 'XLARGE'
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 2
    SCALING_POLICY = 'ECONOMY'
    AUTO_SUSPEND = 60
    INITIALLY_SUSPENDED = TRUE;

-- Starts with 1 cluster
-- Adds 1 more only if sustained queue
-- Reduces unnecessary scaling costs
-- Suitable for predictable workload
```

**Example 3: 24/7 Production Workload**
```sql
CREATE WAREHOUSE prod_core_wh 
    WAREHOUSE_SIZE = 'LARGE'
    MIN_CLUSTER_COUNT = 2      -- Always have capacity
    MAX_CLUSTER_COUNT = 5      -- Scale for peaks
    SCALING_POLICY = 'AUTO'
    AUTO_SUSPEND = NULL        -- Always available
    AUTO_RESUME = TRUE;

-- Minimum 2 clusters for HA/redundancy
-- Scales to 5 for peak loads
-- No suspension for 24/7 availability
```

#### Cost Analysis: Single vs. Multi-Cluster

```
Scenario: 100 concurrent queries, peak traffic 8am-6pm (10 hours)

Option 1: Single XLARGE warehouse
Cost: 10 hours × 60 minutes × 16 credits/minute = 9,600 credits

Option 2: Multi-cluster (MIN=1, MAX=3) with MEDIUM clusters
- 2 clusters average needed
- Cost: 10 hours × 60 minutes × (4 + 4) credits = 4,800 credits
- Benefit: 50% cost reduction, same throughput
- Trade-off: Additional latency for queue-based scaling
```

#### Monitoring Multi-Cluster Warehouse

```sql
-- Monitor cluster utilization
SELECT 
    warehouse_name,
    warehouse_size,
    MAX_CLUSTER_COUNT,
    MIN_CLUSTER_COUNT,
    DATE_TRUNC('MINUTE', start_time) as minute,
    COUNT(DISTINCT query_id) as queries,
    COUNT(DISTINCT cluster_number) as clusters_used,
    AVG(queued_provisioning_time) as avg_queue_ms
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE warehouse_name = 'multi_cluster_wh'
    AND start_time >= DATEADD(day, -7, CURRENT_DATE())
GROUP BY 1, 2, 3, 4, 5
ORDER BY start_time DESC;

-- Determine if scaling policy is effective
SELECT 
    DATEDIFF(hour, MIN(start_time), MAX(start_time)) as hours_monitored,
    COUNT(DISTINCT DATE_TRUNC('hour', start_time)) as active_hours,
    COUNT(CASE WHEN queued_provisioning_time > 0 THEN 1 END) as queued_queries,
    COUNT(*) as total_queries,
    ROUND(100.0 * COUNT(CASE WHEN queued_provisioning_time > 0 THEN 1 END) / COUNT(*), 2) as queue_percentage
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE warehouse_name = 'multi_cluster_wh'
    AND start_time >= DATEADD(week, -1, CURRENT_DATE());
```

---

### Query Acceleration Service (QAS)

#### Understanding Query Acceleration Service

**Definition**: Optional cloud service that reduces query latency for long-running queries by using cached results and optimized processing.

#### How QAS Works

```
Traditional Query Path:
1. Parse & Compile (seconds)
2. Queue (if busy)
3. Execute on warehouse (seconds to minutes)
4. Return results

QAS-Accelerated Path:
1. Parse & Compile (seconds)
2. Check acceleration cache (milliseconds)
3. If eligible: Use QAS cluster (seconds)
4. If not: Fall back to warehouse (seconds to minutes)
5. Cache result for future queries
```

#### Eligibility Criteria for QAS

Queries are good candidates for QAS when they have:

1. **Long Execution Time**: > 30 seconds typically
2. **Complex Scanning**: Full table scans with filtering
3. **Limited Concurrency**: Not many identical queries in flight
4. **Repeatable**: Same/similar queries run multiple times
5. **Analytical Nature**: Aggregations, scans, not DML

```sql
-- Check QAS eligibility
SELECT *
FROM TABLE(
    INFORMATION_SCHEMA.QUERY_ACCELERATION_ELIGIBLE(
        WAREHOUSE_NAME => 'analytical_wh',
        ESTIMATED_ACCELERATION_BENEFIT => 2  -- 2x or better
    )
)
LIMIT 10;
```

#### Enabling Query Acceleration Service

```sql
-- Enable on warehouse
ALTER WAREHOUSE analytical_wh SET 
    ENABLE_QUERY_ACCELERATION = TRUE
    QUERY_ACCELERATION_MAX_RUNNING_QUERIES = 10;

-- Check if enabled
SHOW WAREHOUSES LIKE 'analytical_wh';

-- Disable if needed
ALTER WAREHOUSE analytical_wh SET 
    ENABLE_QUERY_ACCELERATION = FALSE;
```

#### QAS Cost Model

```
Query Acceleration Service Charges:
- Charged per query scanned through QAS
- Cost: Fraction of compute credits (~20% of compute cost)
- Benefit: Queries run 2-10x faster typically

Example:
Query execution without QAS: 100 seconds × (1 credit/60s) = 1.67 credits
Query execution with QAS: 30 seconds × (1 credit/60s) × 0.2 = 0.1 credits
Net benefit: 1.57 credits saved = 94% cost reduction

BUT: Only applies if QAS actually accelerates the query
```

#### When to Use QAS

**Good Use Cases**
- Dashboard queries running repeatedly
- Analytical reports with consistent patterns
- Complex aggregations on large tables
- Executive dashboards with strict SLA requirements

**Poor Use Cases**
- Real-time operational queries
- Highly variable queries with different filters
- Queries already < 10 seconds
- One-time analytical queries
- Queries with constantly changing data

#### Optimization with QAS

```sql
-- Example: Enable QAS and analyze impact
ALTER WAREHOUSE dashboard_wh SET ENABLE_QUERY_ACCELERATION = TRUE;

-- Run query normally (first time)
SELECT COUNT(*) FROM large_fact_table;

-- Monitor impact
SELECT 
    query_id,
    query_text,
    execution_time,
    is_query_acceleration_eligible,
    query_acceleration_bytes_scanned,
    query_acceleration_bytes_freed
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE warehouse_name = 'dashboard_wh'
    AND start_time >= DATEADD(hour, -1, CURRENT_TIMESTAMP())
ORDER BY start_time DESC;
```

---

### Snowpark-Optimized Warehouses

#### Purpose and Benefits

**Definition**: Warehouse configuration optimized for Snowpark (Scala, Java, Python) workloads by providing additional memory and resources.

```sql
-- Create Snowpark-optimized warehouse
CREATE WAREHOUSE snowpark_wh 
    WAREHOUSE_TYPE = 'SNOWPARK-OPTIMIZED'
    WAREHOUSE_SIZE = 'MEDIUM';

-- Benefits:
-- - More CPU per node
-- - More memory per node
-- - Optimized for UDF execution
-- - Better performance for Snowpark code
```

#### Use Cases for Snowpark Warehouses

**Scenario 1: Machine Learning Model Training**
```python
# Snowpark with ML libraries
from snowflake.snowpark import Session
from snowflake.snowpark.functions import col, mean, stddev

session = Session.builder.configs(connection_params).create()

df = session.table("ml_training_data")
df_processed = df.select(
    col("feature1"),
    ((col("feature2") - mean("feature2")).over() / 
     stddev("feature2").over()).alias("feature2_normalized")
)
df_processed.write.mode("overwrite").save_as_table("ml_training_normalized")
```

**Scenario 2: Complex Data Transformation with UDFs**
```sql
CREATE OR REPLACE FUNCTION process_array_udf(arr ARRAY)
RETURNS ARRAY
LANGUAGE PYTHON
RUNTIME_VERSION = 3.10
HANDLER = 'transform_array'
SNOWPARK_OPTIMIZED = TRUE
AS
$$
def transform_array(arr):
    return [x * 2 for x in arr if x > 0]
$$;

-- Execute on Snowpark-optimized warehouse
USE WAREHOUSE snowpark_wh;
SELECT process_array_udf(array_column) FROM large_table;
```

#### Snowpark vs. Standard Warehouse Comparison

| Aspect | Standard | Snowpark-Optimized |
|--------|----------|-------------------|
| Memory/Node | Base allocation | 50% more memory |
| CPU/Node | Base allocation | 50% more CPU |
| Best For | SQL, simple queries | Complex code, UDFs |
| Cost | Lower | Higher (premium) |
| Compilation Time | Fast | Slightly slower |
| Runtime Performance | Good | Excellent for code |

#### Cost-Performance Consideration

```
Snowpark Warehouse Sizing:

Use Case: ETL with complex transformations
- Standard LARGE: 1GB memory/worker, may spill
- Snowpark MEDIUM: More memory, no spilling

Cost Analysis:
- Standard LARGE: 8 credits/min
- Snowpark MEDIUM: 4 credits/min (same size class)
- Premium for optimization: Included in size

Decision: Use Snowpark if code performance is critical
```

---

## Clustering

### Natural Clustering

#### Definition and Concept

**Natural Clustering**: The physical organization of data that already exists in tables, based on how data was inserted or naturally correlates.

```
Example: Orders table
- Data loaded in descending order by DATE
- Data naturally clustered by DATE
- No explicit clustering key defined
- But queries on DATE range benefit from natural clustering

Why it matters:
- Snowflake auto-recognizes and leverages natural clustering
- Provides free optimization without configuration
- Performance depends on INSERT order
```

#### How Natural Clustering Works

```
Physical Data Layout:

Natural Cluster 1: 2024-01-01 to 2024-01-15
Natural Cluster 2: 2024-01-16 to 2024-01-31
Natural Cluster 3: 2024-02-01 to 2024-02-15

Query: SELECT * FROM orders WHERE date = '2024-01-10'
- Scans only Cluster 1 (pruning Clusters 2-3)
- No explicit clustering key needed
- Works automatically
```

#### Measuring Natural Clustering

```sql
-- Check clustering quality for column
SELECT SYSTEM$CLUSTERING_DEPTH('table_name', 'column_name');

-- Returns clustering depth (0-1 scale)
-- 0 = perfectly clustered
-- 1 = randomly ordered
-- 0.5 = moderately clustered

-- Interpret results:
-- < 0.3: Good natural clustering
-- 0.3-0.6: Moderate clustering
-- > 0.6: Poor clustering, consider clustering key
```

#### When Natural Clustering is Sufficient

**Sufficient Scenarios**
- Data loaded in order of query predicates
- Time-series data loaded chronologically
- Data partitioned by source system (naturally isolated)
- Few distinct query patterns
- Ad-hoc exploratory queries

**Insufficient Scenarios**
- Random INSERT order
- No correlated query predicates
- Multiple query patterns on different columns
- Need for pruning across dimensions
- Large tables with high cardinality

---

### Auto-Clustering

#### Understanding Auto-Clustering

**Definition**: Snowflake background service that automatically maintains clustering on defined clustering keys without manual maintenance.

#### Enabling Auto-Clustering

```sql
-- Enable at table level
ALTER TABLE orders 
    CLUSTER BY (customer_id, order_date);

-- This automatically:
-- 1. Analyzes data distribution
-- 2. Reorganizes data to match clustering key
-- 3. Maintains clustering as new data arrives
-- 4. Removes clustering when maintenance cost exceeds benefit

-- Check auto-clustering status
SHOW TABLES IN schema_name LIKE 'orders';
-- Look for CLUSTERING_KEY column
```

#### Auto-Clustering Cost Model

```
Clustering Cost Calculation:

Cost to cluster = (Bytes touched for clustering) / (1 GB)

Example:
- Table: 100 GB
- Auto-cluster maintenance: 20 GB reorganized per day
- Daily cost: 20 GB / 1 GB × cost_per_credit = 20 credits
- Monthly: 20 × 30 = 600 credits

Decision threshold:
- If clustering saves > 600 credits per month in queries: Enable
- If saves < 600 credits: Don't enable
```

#### When to Enable Auto-Clustering

**Enable Auto-Clustering When:**
1. **Significant Query Filtering**: Most queries filter on clustering key
2. **Large Table**: > 10 GB, where pruning saves substantial scan
3. **High Insert Rate**: Constant data loading
4. **Predictable Access Patterns**: Consistent query filters
5. **Cost Justified**: Query benefits exceed clustering cost

```sql
-- Example: Sales analytics table
-- Most queries filter by DATE and REGION
-- High daily insert rate (1M rows/day)
-- Table size: 500 GB

ALTER TABLE sales 
    CLUSTER BY (date, region);

-- Cost analysis:
-- Saved per query: 80% of scan bytes
-- Query cost without clustering: 5 credits
-- Query cost with clustering: 1 credit (4 credits saved)
-- Average 50 queries/day on this table
-- Daily benefit: 50 × 4 = 200 credits
-- Auto-clustering cost: ~15 credits/day
-- Net benefit: 185 credits/day = 5,550 credits/month
```

#### When NOT to Enable Auto-Clustering

**Don't Enable When:**
1. **Diverse Query Patterns**: No dominant filtering columns
2. **Ad-hoc Queries**: Unpredictable access patterns
3. **Small Tables**: < 100 MB, clustering cost exceeds benefit
4. **Infrequent Queries**: Low query frequency on table
5. **Real-time Updates**: Constant updates make clustering expensive

```sql
-- Example: Lookup table
-- Size: 10 MB
-- Query frequency: 2-3 times daily
-- Various access patterns

-- Don't cluster - cost exceeds benefit
-- Natural pruning or simple indexes sufficient
```

---

### Clustering Keys

#### Clustering Key Definition

**Clustering Key**: One or more columns specified to physically organize data for optimal query performance.

```sql
-- Define clustering key
ALTER TABLE large_table 
    CLUSTER BY (column1, column2);

-- Single column clustering key
ALTER TABLE orders 
    CLUSTER BY (order_date);

-- Multi-column clustering key
ALTER TABLE sales 
    CLUSTER BY (year, month, product_id);
```

#### How Clustering Keys Work

```
Clustering Key Selection: (customer_id, order_date)

Physical Data Organization:
Micropartition 1: Customer 1001-1100, Jan 2024
Micropartition 2: Customer 1001-1100, Feb 2024
Micropartition 3: Customer 1101-1200, Jan 2024
Micropartition 4: Customer 1101-1200, Feb 2024

Query: SELECT * WHERE customer_id = 1050 AND order_date = '2024-01-15'
- Scans only Micropartition 1 (prunes 3 others)
- Pruning efficiency: 75% reduction in scan

Query: SELECT * WHERE order_date = '2024-02-01'
- Scans Micropartitions 2 and 4 (prunes 1 and 3)
- Less efficient due to secondary key, still beneficial
```

#### Clustering Key Design Principles

**Principle 1: Match Query Predicates**
```sql
-- Bad: Cluster by random column
ALTER TABLE orders CLUSTER BY (id);
-- Queries don't typically filter by id (natural key)

-- Good: Cluster by frequent filter columns
ALTER TABLE orders CLUSTER BY (order_date, customer_id);
-- Most queries filter by these columns
```

**Principle 2: Order by Selectivity**
```sql
-- Primary cluster key: Most selective filters first
-- Secondary cluster key: Moderate selectivity
-- Tertiary: Support queries but less critical

-- Example for sales table:
-- Query 1: 60% queries filter by REGION (high selectivity)
-- Query 2: 80% queries filter by YEAR (lower selectivity)
-- Query 3: 40% queries filter by PRODUCT_ID

ALTER TABLE sales CLUSTER BY (region, year, product_id);
-- Order matches query frequency and selectivity
```

**Principle 3: Cardinality Consideration**
```sql
-- High cardinality: Many distinct values
-- Low cardinality: Few distinct values

-- Good clustering key cardinality:
-- - First column: Moderate-to-high (10-1000 distinct)
-- - Second column: High (100-10000 distinct)
-- - Avoid: Boolean or very low cardinality first

-- Example:
ALTER TABLE events CLUSTER BY (date, event_type, user_id);
-- date: 365 distinct (good)
-- event_type: 50 distinct (acceptable)
-- user_id: 1M distinct (good)
```

#### Multi-Column Clustering Key Strategy

```
Decision Matrix for Multi-Column Keys:

1. Analyze query patterns:
   - Which columns are in WHERE clause?
   - What percentage of queries use each column?
   - What's the filtering selectivity?

2. Calculate key efficiency:
   Column A: Used in 80% of queries, 90% selectivity
   Column B: Used in 60% of queries, 50% selectivity
   Column C: Used in 40% of queries, 20% selectivity

3. Design clustering key:
   ALTER TABLE t CLUSTER BY (column_a, column_b, column_c);
   
   - Most queries use A (high benefit)
   - Many queries also benefit from B
   - Fewer queries use C (lower priority)

4. Trade-off analysis:
   - 2-column key: Simpler, lower maintenance cost
   - 3-column key: Better optimization, higher cost
   - Choose based on cost-benefit
```

#### Clustering Key Cost-Benefit Analysis

```sql
-- Determine optimal clustering key
WITH query_analysis AS (
    SELECT 
        column_name,
        COUNT(*) as filter_count,
        ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM query_history), 2) as pct_of_queries
    FROM query_history
    WHERE table_name = 'target_table'
        AND column_name IN (WHERE_clause_columns)
    GROUP BY column_name
    ORDER BY pct_of_queries DESC
)
SELECT 
    column_name,
    filter_count,
    pct_of_queries,
    CASE 
        WHEN pct_of_queries > 50 THEN 'PRIMARY'
        WHEN pct_of_queries > 30 THEN 'SECONDARY'
        WHEN pct_of_queries > 10 THEN 'TERTIARY'
        ELSE 'SKIP'
    END as clustering_priority
FROM query_analysis;

-- Result guides clustering key order
```

#### Practical Clustering Key Examples

**Example 1: E-commerce Orders Table**
```sql
-- Typical queries:
-- 1. "Orders by customer in date range" (90%)
-- 2. "Orders by order_date" (70%)
-- 3. "Orders by status" (20%)

ALTER TABLE orders CLUSTER BY (customer_id, order_date);

-- Justification:
-- - customer_id is most selective (millions of values)
-- - order_date supports time-range queries
-- - status has low cardinality (skip it)
```

**Example 2: Analytics Events Table**
```sql
-- Typical queries:
-- 1. "Events by user on specific date" (80%)
-- 2. "Events by event_type on date range" (60%)
-- 3. "Daily aggregations" (90%)

ALTER TABLE events CLUSTER BY (event_date, user_id, event_type);

-- Justification:
-- - event_date is primary (all queries use it)
-- - user_id provides granular filtering
-- - event_type supports categorization
```

**Example 3: Financial Transactions Table**
```sql
-- Typical queries:
-- 1. "Transactions by account_id" (85%)
-- 2. "Transactions by transaction_date range" (95%)
-- 3. "Transactions by transaction_type" (30%)

ALTER TABLE transactions CLUSTER BY (transaction_date, account_id);

-- Justification:
-- - transaction_date is almost universal (95%)
-- - account_id has high selectivity
-- - Maintain time-series integrity first
```

#### Monitoring Clustering Key Effectiveness

```sql
-- Monitor clustering efficiency
SELECT 
    table_name,
    clustering_key,
    SYSTEM$CLUSTERING_DEPTH(CONCAT(schema_name, '.', table_name), column_name) as clustering_depth,
    CASE 
        WHEN SYSTEM$CLUSTERING_DEPTH(...) < 0.3 THEN 'EXCELLENT'
        WHEN SYSTEM$CLUSTERING_DEPTH(...) < 0.5 THEN 'GOOD'
        WHEN SYSTEM$CLUSTERING_DEPTH(...) < 0.8 THEN 'FAIR'
        ELSE 'POOR'
    END as clustering_quality
FROM INFORMATION_SCHEMA.TABLES
WHERE clustering_key IS NOT NULL
ORDER BY clustering_depth ASC;

-- Identify degraded clustering keys
SELECT 
    query_id,
    table_name,
    bytes_scanned,
    rows_produced,
    ROUND(bytes_scanned::numeric / NULLIF(rows_produced, 0), 2) as bytes_per_row,
    CASE 
        WHEN bytes_scanned > bytes_produced * 1000 THEN 'POOR CLUSTERING'
        WHEN bytes_scanned > bytes_produced * 100 THEN 'DEGRADED CLUSTERING'
        ELSE 'ACCEPTABLE'
    END as clustering_assessment
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE table_name = 'target_table'
    AND start_time >= DATEADD(day, -7, CURRENT_DATE())
ORDER BY bytes_per_row DESC;
```

---

## Search Optimization Service

### Understanding Search Optimization Service

**Definition**: Optional service that creates and maintains search indexes on specified columns to dramatically accelerate point lookups and range scans.

#### How Search Optimization Works

```
Traditional Query (without SOS):
WHERE customer_id = 'CUST_12345'
- Full table scan required
- 100 GB table = 100 GB scanned
- Time: 10-30 seconds

With Search Optimization:
WHERE customer_id = 'CUST_12345'
- Leverages optimized index structure
- Scans only relevant micropartitions
- 100 GB table = 50 MB scanned
- Time: 1-2 seconds (10-15x faster)
```

#### When Search Optimization Service Helps

**Ideal Use Cases**
1. **Point Lookups**: WHERE column = value
2. **Range Scans**: WHERE column BETWEEN x AND y
3. **IN Predicates**: WHERE column IN (list)
4. **Text Search**: LIKE patterns on VARCHAR columns
5. **High Cardinality Columns**: Many distinct values

**Poor Use Cases**
1. **Wildcard Patterns**: WHERE column LIKE '%value%' (leading wildcard)
2. **Full Table Scans**: No WHERE clause
3. **Aggregations**: GROUP BY without filtering
4. **Complex Joins**: Optimization doesn't apply well
5. **Columns with NULL**: NULL filtering less optimized

#### Enabling Search Optimization Service

```sql
-- Add search optimization to column
ALTER TABLE orders ADD SEARCH OPTIMIZATION ON (customer_id);

-- Add to multiple columns
ALTER TABLE orders ADD SEARCH OPTIMIZATION ON (customer_id, order_date);

-- Create table with search optimization
CREATE TABLE optimized_table (
    id VARCHAR,
    customer_id VARCHAR,
    order_date DATE,
    amount DECIMAL(10, 2)
)
WITH SEARCH OPTIMIZATION;

-- Enable on existing table
ALTER TABLE existing_table 
    ADD SEARCH OPTIMIZATION ON (customer_id, order_date);

-- Check enabled columns
SHOW SEARCH OPTIMIZATION ON TABLE orders;

-- Disable search optimization
ALTER TABLE orders DROP SEARCH OPTIMIZATION ON (customer_id);
```

#### Search Optimization Cost Model

```
Maintenance Cost:
- Initial indexing: One-time cost (background)
- Ongoing maintenance: ~10-20% of compute credits for ingestion
- Storage overhead: ~10-15% additional storage

Example:
Table: 1 TB, 10M rows inserted daily
- Without SOS: Baseline cost
- With SOS: +2-5 credits/day maintenance
- Query speedup: 10-30 queries/day save 5 credits each = 50-150 credits/day benefit
- Net benefit: 50-150 credits/day - 2-5 credits = 45-145 credits/day

Decision: Implement if query benefits exceed maintenance costs
```

#### Search Optimization Performance Impact

```sql
-- Before enabling search optimization
SELECT COUNT(*) FROM large_table 
WHERE customer_id = 'CUST_12345';
-- Execution time: 25 seconds
-- Bytes scanned: 100 GB

-- After enabling search optimization
ALTER TABLE large_table ADD SEARCH OPTIMIZATION ON (customer_id);

SELECT COUNT(*) FROM large_table 
WHERE customer_id = 'CUST_12345';
-- Execution time: 2 seconds (12.5x faster)
-- Bytes scanned: 8 GB (1.25% of original)

-- Query profile shows:
-- - Prune phase: Very fast (sub-second)
-- - Scan phase: Minimal bytes (92% reduction)
```

#### Monitoring Search Optimization

```sql
-- Check search optimization usage
SELECT 
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    search_optimization_used
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE table_name = 'target_table'
    AND start_time >= DATEADD(day, -7, CURRENT_DATE())
ORDER BY bytes_scanned DESC;

-- Analyze search optimization effectiveness
WITH seo_comparison AS (
    SELECT 
        'WITH_SOS' as optimization_status,
        AVG(execution_time) as avg_execution_ms,
        AVG(bytes_scanned) as avg_bytes_scanned,
        COUNT(*) as query_count
    FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE table_name = 'target_table'
        AND search_optimization_used = TRUE
        AND start_time >= DATEADD(day, -7, CURRENT_DATE())
)
SELECT 
    optimization_status,
    avg_execution_ms,
    avg_bytes_scanned,
    query_count
FROM seo_comparison
ORDER BY avg_execution_ms;

-- Cost of search optimization
SELECT 
    SUM(credits_used) as total_credits_for_sos
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY
WHERE service = 'SEARCH_OPTIMIZATION'
    AND start_time >= DATEADD(month, -1, CURRENT_TIMESTAMP());
```

---

## Caching

### Cache Layers in Snowflake

Snowflake implements multiple cache layers to optimize performance and reduce data scanning.

#### Layer 1: Result Cache (Query Cache)

**Definition**: Caches complete query results for 24 hours, returning results instantly if same query executed again.

```
Query Execution without Result Cache:
1. Compile & Plan: 100ms
2. Execute on warehouse: 10,000ms
3. Total: 10,100ms
4. Cost: 1 credit (for SMALL warehouse)

Query Execution with Result Cache (hit):
1. Result lookup: 5ms
2. Return cached results
3. Total: 5ms
4. Cost: 0 credits
```

**Cache Hit Conditions**:
```sql
-- Result cache hits when:
-- 1. Same SQL text (exact match, case-insensitive)
-- 2. Same database/schema context
-- 3. Same warehouse size (or equivalent)
-- 4. Cache still valid (< 24 hours)

-- Cache hit example:
SELECT COUNT(*) FROM orders;  -- 10s, 1 credit
SELECT count(*) FROM orders;  -- 5ms, 0 credits (hit!)
SELECT COUNT(*) FROM ORDERS;  -- 5ms, 0 credits (hit!)

-- Cache miss example:
SELECT COUNT(*) FROM orders WHERE year = 2024;  -- Different query, miss
SELECT COUNT(*) FROM orders;  -- 5ms, 0 credits (still cached)
```

**Disabling Result Cache**:
```sql
-- Disable for current session
ALTER SESSION SET USE_CACHED_RESULT = FALSE;

-- Disable at query level
SELECT COUNT(*) FROM orders
WHERE 1=1 OR 1=0;  -- Forces execution, bypasses cache

-- Or use NO_RESULT_CACHE hint
SELECT /*+ NO_RESULT_CACHE */ COUNT(*) FROM orders;
```

#### Layer 2: Warehouse Cache (Local Disk)

**Definition**: SSD cache on warehouse nodes that stores recently accessed data blocks for fast re-access.

```
Cache Behavior:
- Persists for warehouse lifecycle
- Cleared when warehouse suspended
- Size depends on warehouse size (25% for SMALL, 50% for LARGE)
- Stores up to 1 TB of data typically

Hit rate example:
- Query 1: SELECT * FROM large_table WHERE id = 1  (cache miss, 15s)
- Query 2: SELECT * FROM large_table WHERE id = 2  (cache hit, 2s)
- Query 3 (after suspend): SELECT * FROM large_table WHERE id = 3 (cache miss, 15s)
```

**Optimizing Warehouse Cache**:
```sql
-- Keep warehouse running to maintain cache
ALTER WAREHOUSE analytical_wh SET AUTO_SUSPEND = NULL;
-- Or set to longer interval
ALTER WAREHOUSE analytical_wh SET AUTO_SUSPEND = 3600;  -- 1 hour

-- Monitor cache hit rates
SELECT 
    warehouse_name,
    DATE_TRUNC('hour', start_time) as hour,
    COUNT(*) as total_queries,
    COUNT(CASE WHEN queued_provisioning_time = 0 THEN 1 END) as not_queued,
    AVG(execution_time) as avg_execution_ms
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD(day, -7, CURRENT_DATE())
GROUP BY warehouse_name, hour
ORDER BY hour DESC;
```

#### Layer 3: Metadata Cache

**Definition**: Caches metadata (table structure, statistics) to speed up compilation.

```
Impact:
- First compilation of query pattern: 500ms
- Subsequent compilations (same pattern): 50ms
- 10x speedup for compilation-heavy workloads

Example:
SELECT * FROM table WHERE col1 = 1  (first run, 500ms compile)
SELECT * FROM table WHERE col1 = 2  (cached metadata, 50ms compile)
SELECT * FROM table WHERE col1 = 3  (cached metadata, 50ms compile)
```

#### Cache Expiration and Invalidation

**Result Cache Expiration**:
```sql
-- 24-hour TTL
-- Expires when:
-- 1. 24 hours elapsed since query execution
-- 2. Underlying table data modified (INSERT/UPDATE/DELETE)
-- 3. Table structure modified (ALTER TABLE)
-- 4. User executes CLEAR QUERY CACHE command

-- Manual cache clearing
SELECT SYSTEM$CANCEL_ALL_QUERIES();  -- Not for cache, but for query cancellation
-- Cache automatic invalidation on data change

-- Example:
SELECT COUNT(*) FROM orders;  -- 10s, cached
INSERT INTO orders VALUES (...);  -- Cache invalidated
SELECT COUNT(*) FROM orders;  -- 10s, cache miss (new count needed)
```

**Warehouse Cache Invalidation**:
```
Invalidation triggers:
1. Warehouse suspension (clears entire cache)
2. Warehouse resize (may partially clear)
3. Underlying data modification (incremental invalidation)
4. 24-hour idle (cleanup mechanism)

Result:
- Active warehouses maintain cache longer
- Suspended warehouses lose cache benefits
- Trade-off: Suspension cost vs. cache benefit
```

---

### Cache Impact on Costs

#### Scenario Analysis: Cost Impact of Caching

**Scenario 1: Dashboard with 5 Queries (Interactive Use)**
```
Dashboard refresh frequency: Every 5 minutes
Average query cost: 1 credit
Query count: 5 per refresh

Without caching:
5 queries × 1 credit × 12 refreshes/hour × 10 hours/day = 600 credits/day

With result caching (2 cache hits/refresh):
Effective query cost: 3 queries × 1 credit = 3 credits per refresh
3 credits × 12 refreshes/hour × 10 hours/day = 360 credits/day

Cost savings: 240 credits/day = 7,200 credits/month
```

**Scenario 2: Batch Processing (Same Queries)**
```
Batch job runs 200 queries
Each query costs 5 credits
Queries are repeatable (run daily)

Day 1 (cold cache):
200 queries × 5 credits = 1,000 credits

Day 2+ (warm cache):
Assuming 50 unique queries × 5 credits = 250 credits
+ 150 repeat queries from cache = 0 credits
Total: 250 credits

Monthly savings: 30 days × 250 credits × 29 days = 217,500 credits
OR: 29 × 750 credits/day = 21,750 credits
```

**Scenario 3: Warehouse Suspension vs. Cache Maintenance**
```
Scenario A: Keep warehouse running for cache
- Warehouse cost: 4 credits/hour × 24 hours = 96 credits/day
- Query savings from cache: 150 credits/day
- Net benefit: +54 credits/day

Scenario B: Suspend warehouse hourly
- Warehouse cost: 0 credits (suspended)
- Query cost without cache: 150 credits/day
- Total: 150 credits/day

Decision: Keep running if cache saves > suspension cost
- 96 credits/day suspension cost
- 54 credits/day cache benefit
- = -42 credits/day (NOT worth it)

Instead: Suspend at 300 seconds (5 min)
- Partial cache maintenance during active hours
- Better cost-benefit ratio
```

#### Cache Cost Optimization Strategy

**Strategy 1: Leverage Result Cache for BI/Dashboard**
```sql
-- Best for: Stable, repeated queries
ALTER SESSION SET USE_CACHED_RESULT = TRUE;
-- Default is already TRUE, so explicit setting rare

-- Monitor cache effectiveness
SELECT 
    query_text,
    COUNT(*) as executions,
    SUM(CASE WHEN execution_time < 100 THEN 1 ELSE 0 END) as cache_hits,
    ROUND(100.0 * SUM(CASE WHEN execution_time < 100 THEN 1 ELSE 0 END) / COUNT(*), 1) as cache_hit_rate,
    AVG(execution_time) as avg_execution_ms
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD(day, -7, CURRENT_DATE())
GROUP BY query_text
HAVING COUNT(*) > 10
ORDER BY cache_hit_rate DESC;
```

**Strategy 2: Optimize Warehouse Suspension for Cache**
```sql
-- For high-query-volume warehouses
ALTER WAREHOUSE analytical_wh SET AUTO_SUSPEND = 600;  -- 10 minutes

-- For batch processing warehouses
ALTER WAREHOUSE batch_wh SET AUTO_SUSPEND = 60;  -- 1 minute (cache less important)

-- For 24/7 operations
ALTER WAREHOUSE prod_wh SET AUTO_SUSPEND = NULL;  -- Always warm
```

**Strategy 3: Combine Caching with Clustering**
```sql
-- Cached result + cluster pruning = maximum efficiency

-- Setup:
ALTER TABLE large_table CLUSTER BY (date, customer_id);

-- Query with both benefits:
SELECT * FROM large_table 
WHERE customer_id = 'CUST_123' 
  AND date >= '2024-01-01';

-- First execution:
-- - Clustering prunes 80% of data
-- - Remaining data cached
-- - Result cached

-- Second identical execution:
-- - Result cache hit, return in 5ms
-- - 0 credits

-- Similar but different query:
SELECT * FROM large_table 
WHERE customer_id = 'CUST_456' 
  AND date >= '2024-01-01';

-- Uses cluster cache (warehouse SSD) + clustering pruning
-- No result cache, but fast warehouse execution
-- Minimal credit cost
```

#### Cache Cost Estimation Tool

```sql
-- Analyze potential cache savings
WITH query_analysis AS (
    SELECT 
        query_text,
        COUNT(*) as total_executions,
        COUNT(DISTINCT DATE_TRUNC('day', start_time)) as days_executed,
        AVG(total_elapsed_time) as avg_execution_ms,
        SUM(credits_used) as total_credits,
        ROW_NUMBER() OVER (ORDER BY COUNT(*) DESC) as execution_rank
    FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE start_time >= DATEADD(day, -30, CURRENT_DATE())
    GROUP BY query_text
)
SELECT 
    execution_rank,
    query_text,
    total_executions,
    days_executed,
    ROUND(avg_execution_ms, 0) as avg_exec_ms,
    total_credits,
    CASE 
        WHEN total_executions / days_executed >= 2 
            THEN ROUND(total_credits * 0.5, 0)  -- 50% savings estimate
        WHEN total_executions / days_executed >= 1.5 
            THEN ROUND(total_credits * 0.3, 0)  -- 30% savings estimate
        ELSE 0  -- No caching benefit
    END as potential_savings
FROM query_analysis
WHERE execution_rank <= 20
ORDER BY potential_savings DESC;
```

---

## Best Practices Summary

### Performance Optimization Hierarchy

```
Priority 1 - Query Logic (Highest ROI)
  └─ Fix WHERE clause predicates
  └─ Optimize JOIN order
  └─ Eliminate unnecessary columns
  └─ Push filtering early
  └─ Cost: $0, Benefit: 10-100x speedup

Priority 2 - Schema Optimization (Medium ROI)
  └─ Implement clustering keys
  └─ Enable search optimization service
  └─ Create materialized views
  └─ Archive old data
  └─ Cost: $1-100, Benefit: 2-10x speedup

Priority 3 - Warehouse Configuration (Moderate ROI)
  └─ Right-size warehouse
  └─ Implement multi-cluster
  └─ Optimize auto-suspend
  └─ Cost: $10-1000/month, Benefit: 2-5x speedup

Priority 4 - Premium Services (Lowest ROI, last resort)
  └─ Query acceleration service
  └─ Snowpark-optimized warehouses
  └─ Cost: $100-1000/month, Benefit: 1.5-3x speedup
```

### Query Profile Best Practices

1. **Always check query profile for queries > 10 seconds**
2. **Identify bottleneck (highest time percentage)**
3. **Apply recommendations in priority order**
4. **Implement Tier 1 fixes before Tier 3 warehouse changes**
5. **Re-run query to measure improvement**

### Warehouse Configuration Best Practices

1. **Interactive Workload**:
   ```sql
   WAREHOUSE_SIZE = MEDIUM
   MIN_CLUSTER_COUNT = 1
   MAX_CLUSTER_COUNT = 3
   SCALING_POLICY = AUTO
   AUTO_SUSPEND = 300
   AUTO_RESUME = TRUE
   ```

2. **Batch Processing**:
   ```sql
   WAREHOUSE_SIZE = LARGE
   MIN_CLUSTER_COUNT = 1
   MAX_CLUSTER_COUNT = 2
   SCALING_POLICY = ECONOMY
   AUTO_SUSPEND = 60
   INITIALLY_SUSPENDED = TRUE
   ```

3. **Production 24/7**:
   ```sql
   WAREHOUSE_SIZE = XLARGE
   MIN_CLUSTER_COUNT = 2
   MAX_CLUSTER_COUNT = 5
   SCALING_POLICY = AUTO
   AUTO_SUSPEND = NULL
   AUTO_RESUME = TRUE
   ```

### Caching Best Practices

1. **Leverage result cache for dashboards** (< 24 hour data freshness requirement)
2. **Keep warehouses running** if cache benefits exceed suspension costs
3. **Combine caching with clustering** for maximum efficiency
4. **Monitor cache hit rates** regularly
5. **Don't disable result cache** unless data changes require it

### Clustering Best Practices

1. **Cluster by frequent filter columns** (used in >50% queries)
2. **Order by selectivity** (most selective first)
3. **Use auto-clustering** for large tables with high insert rates
4. **Monitor clustering depth** monthly
5. **Review clustering effectiveness** after data pattern changes

### Cost Optimization Quick Checklist

- [ ] Enable result caching for repeated queries
- [ ] Implement clustering keys on tables > 100 GB
- [ ] Use multi-cluster warehouses for variable concurrency
- [ ] Set appropriate auto-suspend (60-600 seconds)
- [ ] Review query profiles for bottlenecks monthly
- [ ] Evaluate search optimization ROI quarterly
- [ ] Archive historical data to reduce table size
- [ ] Monitor bytes spilled and add filtering predicates
- [ ] Right-size warehouses based on utilization data
- [ ] Implement query acceleration service for heavy analytics

---

## Exam Tips for Snowflake Advanced Certification

### Key Concepts to Master

1. **Query Profile Interpretation**:
   - Know how to read operator tree
   - Identify top time consumers
   - Recognize spilling patterns
   - Apply appropriate solutions

2. **Performance Trade-offs**:
   - Cost vs. Speed (warehouse sizing)
   - Maintenance vs. Query benefit (clustering)
   - Storage overhead vs. Lookup speed (search optimization)

3. **Configuration Scenarios**:
   - Match warehouse config to workload type
   - Understand multi-cluster scaling mechanics
   - Know when to use each caching layer

4. **Metadata Functions**:
   - QUERY_HISTORY queries
   - CLUSTERING_DEPTH calculation
   - Performance metric analysis

### Practice Scenarios for Exam

**Scenario 1**: Dashboard query taking 30 seconds. Query profile shows 90% time in table scan, 100 GB bytes scanned, 1M rows produced. Recommend optimization.
- Answer: Add WHERE clause filtering with clustering key, implement search optimization

**Scenario 2**: Multi-cluster warehouse with AUTO scaling policy. Peak hour has consistent queue. Recommend changes.
- Answer: Increase MIN_CLUSTER_COUNT or increase base warehouse size

**Scenario 3**: Warehouse spilling 50 GB to remote storage. Recommend solution.
- Answer: Scale up warehouse, add filtering, materialize intermediate results

**Scenario 4**: Production warehouse 24/7 with 100-300 concurrent users. Design warehouse.
- Answer: Multi-cluster (MIN=2, MAX=5), XLARGE, AUTO scaling, AUTO_SUSPEND=NULL

---

## References and Additional Resources

### Key Snowflake Documentation
- Query Profile Guide
- Warehouse Configuration Reference
- Clustering Best Practices
- Cache Behavior Documentation
- Search Optimization Service Guide

### Metrics to Monitor Regularly
- Query execution time distribution
- Bytes scanned vs. bytes produced ratio
- Cache hit rates
- Warehouse utilization percentage
- Credit consumption trends

---

**Study Tips**:
- Hands-on practice with QUERY_HISTORY queries
- Create test tables and experiment with clustering
- Profile sample queries and interpret results
- Calculate ROI for optimization techniques
- Design warehouse configurations for given scenarios

