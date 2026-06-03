# ❄️ Snowflake Advanced Certification Study Notes
## Topic: Snowflake Object Hierarchy

> **Exam Domain:** SnowPro Advanced – Data Engineer / Architect / Administrator
> **Last Updated:** June 2025
> **References:** Official Snowflake Documentation, Snowflake Engineering Blog

---

## Table of Contents

1. [Why the Object Hierarchy Matters](#1-why-the-object-hierarchy-matters)
2. [The Complete Object Hierarchy](#2-the-complete-object-hierarchy)
3. [Roles](#3-roles)
   - 3.1 Account-Level Roles
   - 3.2 Database Roles
   - 3.3 How the Hierarchy Shapes Role Design
4. [Virtual Warehouses](#4-virtual-warehouses)
   - 4.1 What a Virtual Warehouse Is
   - 4.2 Warehouse Sizing
   - 4.3 Auto-Suspend and Auto-Resume
   - 4.4 Multi-Cluster Warehouses
   - 4.5 Serverless Compute vs. Virtual Warehouses
5. [Databases](#5-databases)
6. [Schemas](#6-schemas)
   - 6.1 Standard Schemas
   - 6.2 Managed Access Schemas
   - 6.3 The INFORMATION_SCHEMA and ACCOUNT_USAGE Schemas
7. [Tables](#7-tables)
   - 7.1 Permanent Tables
   - 7.2 Transient Tables
   - 7.3 Temporary Tables
   - 7.4 External Tables
   - 7.5 Hybrid Tables
   - 7.6 Dynamic Tables
   - 7.7 Apache Iceberg Tables
   - 7.8 Table Type Comparison
8. [Views](#8-views)
   - 8.1 Standard Views
   - 8.2 Secure Views
   - 8.3 Materialized Views
9. [Stages](#9-stages)
   - 9.1 User Stages
   - 9.2 Table Stages
   - 9.3 Named Internal Stages
   - 9.4 Named External Stages
10. [File Formats](#10-file-formats)
11. [Functions and Procedures](#11-functions-and-procedures)
    - 11.1 User-Defined Functions (UDFs)
    - 11.2 User-Defined Table Functions (UDTFs)
    - 11.3 Stored Procedures
    - 11.4 The Difference Between Functions and Procedures
12. [Streams and Tasks](#12-streams-and-tasks)
    - 12.1 Streams
    - 12.2 Tasks
    - 12.3 Task DAGs
    - 12.4 Streams and Tasks Working Together
13. [How the Hierarchy Impacts Architecture](#13-how-the-hierarchy-impacts-architecture)
14. [Exam Tips & Common Gotchas](#14-exam-tips--common-gotchas)

---

## 1. Why the Object Hierarchy Matters

Snowflake's object hierarchy is not merely an organizational catalogue — it is the structural backbone that governs three of the most critical aspects of any Snowflake implementation: **access control**, **cost attribution**, and **data governance**.

**Access control is always hierarchical.** A user querying a table needs three things at minimum: USAGE on the database containing the table, USAGE on the schema within that database, and SELECT on the table itself. Missing any one of these grants results in an access denied error, even if the other two are in place. The hierarchy enforces that every level of the stack must be explicitly traversed by the access control model — you cannot grant SELECT on a table and have the database and schema levels implicitly permitted. This is intentional; it prevents accidental exposure through high-level container grants.

**Cost is scoped to virtual warehouses, not to databases or schemas.** A query running against a table in Database A and a query against a table in Database B cost the same amount of compute if they run on the same warehouse with the same resource profile. Understanding this decoupling between the storage hierarchy (databases, schemas, tables) and the compute hierarchy (warehouses) is fundamental to cost attribution. Organizations attribute costs by assigning dedicated warehouses per team, workload type, or environment — not by splitting databases.

**Governance policies are scoped within the storage hierarchy.** A masking policy on a column applies to that column in all queries regardless of who runs them or which warehouse they use. A row access policy on a table filters rows for everyone accessing that table through any path. Tags propagate from database level down to table and column level. The hierarchy defines the inheritance and propagation path for all governance decisions.

Understanding the hierarchy conceptually — which objects live where, what privileges they require, and how they relate to each other — enables an architect to design systems that are secure, cost-efficient, and governable at scale.

---

## 2. The Complete Object Hierarchy

Snowflake's object hierarchy has two distinct tracks: the **account-level track** for compute and administrative objects, and the **storage hierarchy track** for data objects. These two tracks intersect in queries, where a database object is processed by a compute object, governed by a security object.

```
ORGANIZATION
    └── ACCOUNT
            ├── ACCOUNT-LEVEL OBJECTS (not inside a database)
            │       ├── Users
            │       ├── Roles (Account Roles)
            │       ├── Virtual Warehouses
            │       ├── Resource Monitors
            │       ├── Network Policies
            │       ├── Integrations (Storage, API, Security, Notification)
            │       ├── Replication / Failover Groups
            │       └── Shares
            │
            └── DATABASES
                    └── SCHEMAS (per database)
                            ├── Tables (Permanent, Transient, Temporary,
                            │         External, Hybrid, Dynamic, Iceberg)
                            ├── Views (Standard, Secure, Materialized)
                            ├── Stages (Named Internal, Named External)
                            ├── File Formats
                            ├── Sequences
                            ├── Functions (UDFs, UDTFs — Scalar, Tabular)
                            ├── Procedures (Stored Procedures)
                            ├── Streams
                            ├── Tasks
                            ├── Pipes (Snowpipe)
                            ├── Alerts
                            ├── Event Tables
                            ├── Dynamic Tables
                            └── Database Roles (scoped to this database)
```

Two built-in schemas exist automatically in every database and cannot be dropped:

**INFORMATION_SCHEMA:** An ANSI-standard schema containing read-only views and table functions that describe the objects within the current database — its tables, columns, views, stages, functions, and more. Querying INFORMATION_SCHEMA yields metadata about the current database only.

**PUBLIC:** The default schema. All users have access to PUBLIC by default through the PUBLIC role. It is typically left empty in governed environments, as granting objects to PUBLIC means every user in the account can see them.

---

## 3. Roles

Roles sit at the account level in Snowflake's hierarchy — they are not contained within a database or schema. They are the bridge between users (who need to do things) and securable objects (which have privileges that allow things to be done).

### 3.1 Account-Level Roles

Account roles have account-wide scope. A privilege granted to an account role on an object anywhere in the account is valid — an account role is not limited to a particular database or schema. Account roles can be granted to users or to other account roles.

The six system-defined account roles are ACCOUNTADMIN, SECURITYADMIN, USERADMIN, SYSADMIN, PUBLIC, and ORGADMIN. These cannot be dropped. All custom roles should eventually be granted to SYSADMIN in the hierarchy to prevent orphaned objects that SYSADMIN and ACCOUNTADMIN cannot manage.

Every session in Snowflake has an active **primary role** — the role under which object creation and ownership decisions are made. Any object created in a session is owned by the primary role, not by the user. This is the DAC (Discretionary Access Control) component of Snowflake's access control model, and it means the hierarchy of role ownership is as architecturally significant as the hierarchy of database objects.

### 3.2 Database Roles

Database roles are scoped to a single database. They cannot be activated in a session directly — they must be granted to an account role. Their privileges only apply to objects within their home database. Account roles cannot be granted to database roles.

Database roles exist to solve the role sprawl problem. In a large enterprise, implementing fine-grained access roles (schema_A_read, schema_B_read, schema_A_write, etc.) at the account level creates hundreds of roles visible to every user in the role-switching dropdown. Database roles confine access roles within their database, keeping the account-level role namespace clean while still enabling precise privilege management.

When a database is shared via Snowflake's Data Sharing mechanism, the provider can grant a database role to the share — cleanly packaging access permissions alongside the data. The consumer receives the database role along with the shared data.

### 3.3 How the Hierarchy Shapes Role Design

The correct pattern for enterprise RBAC in Snowflake mirrors the object hierarchy:

**Database roles act as access roles** — they are defined within a specific database, hold the object-level privileges (SELECT, INSERT, USAGE), and represent a named access pattern on that database's objects.

**Account roles act as functional roles** — they represent a job function (Data Analyst, Data Engineer) and are composed by granting database roles (from one or more databases) to them.

**Users are granted functional account roles** — the user activates the functional role in their session and receives all the database-scoped privileges that have been delegated through the role chain.

This layered design means that changing what objects a "Data Analyst" can access is a matter of changing the database role granted to the analyst account role — a single point of change that propagates to all users holding that role.

---

## 4. Virtual Warehouses

Virtual warehouses sit at the account level, parallel to the database hierarchy rather than within it. This is architecturally significant: a warehouse is never "inside" a database, and a database has no inherent relationship to any warehouse. Any warehouse can query any database it has privileges to access, and any database can be queried by any warehouse. The two hierarchies are fully independent.

### 4.1 What a Virtual Warehouse Is

A virtual warehouse is a named set of compute resources — an independent MPP (Massively Parallel Processing) cluster — that executes SQL queries, loads data, and runs transformations. When a warehouse is active and running, it consists of one or more nodes that read data from cloud storage, execute query operations in parallel, and return results to the session.

A warehouse does not store data. All storage is in cloud object storage (Snowflake-managed or external), entirely outside the warehouse. The warehouse is purely a compute resource. When it suspends, no storage is allocated to it, and no credits are consumed. When it resumes, it picks up where the storage layer left off — exactly as it was.

### 4.2 Warehouse Sizing

Warehouse size (X-Small through 6X-Large and beyond) determines the number of compute nodes in the cluster and, consequently, the amount of memory and parallelism available to process a query. Each size up roughly doubles the compute capacity and doubles the credit consumption rate.

The key architectural insight about sizing is that **scaling up (bigger warehouse) reduces query latency** for individual complex queries that can exploit parallelism, while **scaling out (more clusters in a multi-cluster warehouse) increases throughput** for many concurrent queries. These are different problems requiring different solutions.

For large, complex single queries (heavy ELT transformations, complex aggregations over billions of rows), scaling up the warehouse is the right lever. For many concurrent simple queries (fifty analysts simultaneously querying BI dashboards), scaling out with multi-cluster is the right lever. Choosing the wrong approach wastes credits: a 4XL warehouse running 50 simple concurrent queries does not benefit from its size; 4 concurrent Medium-cluster warehouses would handle the concurrency far more efficiently.

### 4.3 Auto-Suspend and Auto-Resume

Auto-suspend automatically suspends the warehouse after a configurable period of inactivity (no queries running). When suspended, the warehouse consumes no credits. Auto-resume starts the warehouse again the moment a new query arrives. The resume is typically sub-second for most workloads.

The auto-suspend window is an important cost-management lever. An analytics warehouse that serves analyst queries throughout the business day might use a 5-minute auto-suspend — brief enough to stop waste during brief lulls but short enough to be quickly available when the next query arrives. An ETL warehouse that runs scheduled batch jobs every hour might use a 1-minute auto-suspend — since it runs intensively and then sits completely idle for 59 minutes, rapid suspension saves significant credits.

The 60-second minimum billing period per resume means very frequent, very short queries on a warehouse that suspends between each query can be expensive. A warehouse that suspends and resumes 50 times in an hour, each time for a 10-second query, still bills 50 minutes of credits. For high-frequency, low-duration queries, either keeping the warehouse active with a longer auto-suspend or using serverless compute (which avoids the 60-second minimum) may be more cost-efficient.

### 4.4 Multi-Cluster Warehouses

A multi-cluster warehouse is a warehouse configured to run multiple simultaneous compute clusters of the same size. When concurrency demand exceeds what a single cluster can handle without queuing, additional clusters are started automatically (Auto-scale mode) or kept running permanently (Maximized mode).

**Auto-scale mode** is the typical choice for production analytics workloads with variable concurrency. Snowflake monitors query queue depth and starts additional clusters when queries are waiting. When demand drops, idle clusters are suspended. The scaling policy (Standard or Economy) controls how aggressively clusters are started and stopped.

**Maximized mode** runs all configured clusters simultaneously regardless of current demand. It is appropriate for workloads where latency variability is unacceptable — the cluster is always ready at full capacity, even during low-demand periods. Maximized mode is more expensive but eliminates the brief delay when a new cluster must start.

Multi-cluster warehouses require **Enterprise Edition or higher**. They are the primary mechanism for serving high-concurrency BI and reporting workloads without queries queuing.

### 4.5 Serverless Compute vs. Virtual Warehouses

Not all Snowflake compute runs on user-managed virtual warehouses. Several features use Snowflake-managed **serverless compute** — compute that Snowflake provisions, scales, and manages automatically, billed per-use rather than per warehouse-second:

- **Snowpipe** (continuous file ingestion)
- **Serverless Tasks** (task executions with `USER_TASK_MANAGED_INITIAL_WAREHOUSE_SIZE` set instead of a warehouse)
- **Automatic Clustering** (background micro-partition reorganization)
- **Materialized View maintenance** (background refresh)
- **Search Optimization Service** (background index building)
- **Snowpark Container Services** compute pools

Serverless compute has no 60-second minimum billing period — it bills for actual execution time. It is frequently more cost-efficient than a virtual warehouse for sporadic, short-duration work. However, it is not configurable in the same way as a warehouse — the engineer specifies only the initial warehouse size hint (for tasks), and Snowflake manages the rest.

---

## 5. Databases

A database is the first and broadest organizational container within a Snowflake account for data objects. It is an account-level object — databases are peers of virtual warehouses, not children of them.

**Every database automatically contains two built-in schemas:** INFORMATION_SCHEMA (read-only metadata views about the database's objects) and PUBLIC (the default, empty schema). Additional schemas are created explicitly.

**Databases as isolation boundaries:** In a multi-zone architecture (raw, curated, analytics), using separate databases per zone provides clean isolation boundaries for governance policies, Time Travel settings, and access control. `DATA_RETENTION_TIME_IN_DAYS` set at the database level cascades to all schemas and tables within unless overridden at a lower level — making it practical to, for example, set a 0-day retention on a raw staging database and a 30-day retention on an analytics database.

**Databases as sharing units:** The database is the primary object that can be shared via Snowflake's Data Sharing mechanism. A provider creates a share and adds objects from a database to it. The consumer creates a shared database from the share. The database boundary defines the outermost scope of what can be shared in one unit.

**Namespace resolution:** When executing SQL in a Snowflake session, the active database and schema define the namespace for unqualified object references. `SELECT * FROM my_table` resolves to the my_table object in the currently active database and schema. Fully qualified names (`my_database.my_schema.my_table`) bypass session context and work regardless of active session settings.

**Databases cannot be nested.** There is no hierarchy between databases — they are all peers at the account level. Cross-database queries are fully supported using fully qualified names and provided the querying role has the necessary USAGE and object privileges on both databases.

**Cloning:** Databases can be zero-copy cloned with a single DDL statement. The clone creates a full copy of the database structure (all schemas and objects) as metadata references to the same underlying micro-partitions. This is the primary mechanism for environment creation (clone production to create a dev environment in seconds).

---

## 6. Schemas

A schema is the second-level organizational container within a database — a logical grouping of related database objects. Every object in Snowflake's storage hierarchy (table, view, stage, function, procedure, stream, task, etc.) belongs to exactly one schema.

Together, a database and schema form a **namespace** in Snowflake. When a session has an active database and schema, object names in SQL are resolved against that namespace. The fully qualified name of any schema-level object is `database_name.schema_name.object_name`.

**Schemas as domain boundaries:** The typical use of schemas is to separate objects by business domain, data layer, or functional purpose within a database. In an analytics database, schemas might correspond to `SALES`, `FINANCE`, `MARKETING`, and `OPERATIONS` domains. Within a raw database, schemas might correspond to source systems: `SALESFORCE`, `SAP`, `ORACLE_ERP`.

**Multiple schemas can exist in one database without limit.** Snowflake imposes no hard limits on the number of schemas per database or objects per schema. The practical limit is managability — too many schemas with too many objects can become difficult to navigate and govern without tooling.

### 6.1 Standard Schemas

A standard schema allows any role that owns an object within the schema to grant privileges on that object to other roles. This is the default DAC (Discretionary Access Control) behavior — object owners control who can access their objects.

The risk in standard schemas is that this delegation is uncontrolled. An engineer who creates a table in a standard schema can grant SELECT on it to the PUBLIC role, exposing it to every user in the account, without any central authority reviewing or approving that grant. At scale, this makes privilege audit and governance challenging.

### 6.2 Managed Access Schemas

A managed access schema is created with the `WITH MANAGED ACCESS` option. This changes the privilege grant model fundamentally: **only the schema owner (or ACCOUNTADMIN/SECURITYADMIN) can grant privileges on objects within the schema**, even though object creators still own their objects in the DAC sense.

This centralizes access control decisions. A data engineer who creates a table in a managed access schema cannot accidentally (or deliberately) share that table with inappropriate roles — the schema owner's role must approve and execute the grant. This is essential for governance at scale: it ensures a single, auditable decision point for all access grants within a production schema.

The tradeoff is operational overhead — every access request to objects in a managed access schema must go through the schema owner role, which may require an approval workflow in the data governance process.

### 6.3 The INFORMATION_SCHEMA and ACCOUNT_USAGE Schemas

These two special schemas are essential for governance, monitoring, and administration:

**INFORMATION_SCHEMA** (inside each database): Contains read-only views about the objects within the current database. It follows the ANSI SQL standard for information schema views, plus Snowflake-specific extensions for stages, file formats, and other non-standard objects. Queries against INFORMATION_SCHEMA reflect the current state of objects. There is a per-database version and an account-level version that spans all databases.

**SNOWFLAKE.ACCOUNT_USAGE** (in the SNOWFLAKE system database): Contains historical views of account-wide usage, query history, access history, login history, storage usage, and object metadata. Unlike INFORMATION_SCHEMA, ACCOUNT_USAGE views have latency (typically 45 minutes to 3 hours for most views) but retain data for a longer period (most views retain 1 year). These views are the primary source for compliance reporting, cost analysis, and governance auditing.

---

## 7. Tables

Tables are the primary data storage objects within schemas. Snowflake supports multiple distinct table types, each with different properties for Time Travel, Fail-Safe, persistence, and use case. Choosing the wrong table type is one of the most impactful and most common architectural mistakes in Snowflake deployments.

All Snowflake tables store data in Snowflake's columnar, compressed, micro-partition format internally. The differences between table types are about their **persistence, recovery capabilities, and storage origin** — not the underlying physical format.

### 7.1 Permanent Tables

Permanent tables are the default table type — `CREATE TABLE` with no additional keywords creates a permanent table. They are designed for data that must be preserved long-term and protected against accidental loss.

**Time Travel:** Permanent tables support Time Travel for the configured `DATA_RETENTION_TIME_IN_DAYS` period — up to 1 day on Standard Edition, up to 90 days on Enterprise Edition and above. Within this window, any historical state of the table can be queried or cloned.

**Fail-Safe:** After the Time Travel window expires, Snowflake maintains an additional 7-day Fail-Safe period. Fail-Safe is an internal Snowflake recovery mechanism, not accessible by customers directly — only Snowflake Support can restore Fail-Safe data, and only in extreme circumstances. Fail-Safe storage costs are included in the table's storage billing.

**Cost implication:** The combination of Time Travel and Fail-Safe means a permanent table's actual storage cost can be significantly higher than the current data size, especially if large volumes of data are frequently updated or deleted (each overwritten version is retained through the Time Travel period).

**Use case:** Production business data — facts, dimensions, curated datasets, analytics tables — where accidental data loss would have serious consequences.

### 7.2 Transient Tables

Transient tables have the same persistence as permanent tables — they exist until explicitly dropped — but they have no Fail-Safe period and a maximum Time Travel of 1 day (regardless of Snowflake edition).

**The cost trade-off is explicit:** Transient tables save the Fail-Safe storage cost. For staging tables, intermediate transformation results, and other data that is easily reproducible from source, the 7-day Fail-Safe period on a permanent table would generate significant storage costs for no meaningful protection benefit. If staging data is lost, it can simply be re-loaded from source.

**Transient databases and transient schemas** are also supported. Creating a schema as transient means all tables created within it are automatically transient — enforcing the transient property at the schema level rather than requiring developers to remember to specify it on each table.

Transient tables cannot contain hybrid tables — hybrid tables require the full protection semantics of permanent tables.

**Use case:** Staging tables (raw zone in an ELT architecture), intermediate transformation results, large intermediate aggregation tables, any data that can be reproduced from source and where the Fail-Safe cost would be disproportionate to the value of the protection.

### 7.3 Temporary Tables

Temporary tables are session-scoped — they exist only within the session that created them, and are automatically dropped when the session ends. Data in a temporary table is never visible to other sessions or users.

**Key behavioral nuance:** A temporary table and a permanent table with the same name can coexist within the same schema. Within the session that created the temporary table, the temporary table takes precedence — any unqualified references to that table name resolve to the temporary table, shadowing the permanent one. This can cause unexpected behavior and is an important architectural consideration in environments where session-scoped objects are used alongside persistent objects.

Temporary tables have no Time Travel and no Fail-Safe. When the session ends, the data is purged and is not recoverable by any mechanism.

**Use case:** ETL intermediate state within a session (holding computed values for subsequent steps in the same procedure), session-specific cached results, scratch space for exploratory analysis. They are genuinely useful but must be used with awareness of the shadowing behavior.

### 7.4 External Tables

External tables are a bridge between Snowflake and data stored in external cloud storage (S3, Azure Blob, GCS) that the organization does not want to (or cannot) load into Snowflake's managed storage. They are **read-only** — DML operations (INSERT, UPDATE, DELETE) are not supported on external tables.

The external table defines a schema on top of files in an external stage, allowing them to be queried with SQL as if they were a Snowflake table. Partition columns can be defined based on the external storage directory structure, enabling partition pruning that makes queries against large external datasets efficient.

**Important limitations:** External tables do not support Time Travel or Fail-Safe (the data is in external storage, not Snowflake's storage layer). They do not benefit from Snowflake's automatic clustering or most optimization features. Query performance against external tables is typically lower than against native Snowflake tables because every query must read from external object storage. Masking policies can be applied to external table columns, but row access policies and certain other governance features have limitations.

**Use case:** Querying legacy data lake content without migrating it, accessing reference data maintained by external systems, incremental migration where data remains in external storage while being made accessible through Snowflake, or hybrid architectures where cost constraints make it impractical to load all data into Snowflake storage.

### 7.5 Hybrid Tables

Hybrid tables are Snowflake's solution for **OLTP workloads** within the Snowflake platform. They store data in both row-oriented format (for fast single-row lookups and writes) and columnar format (for analytical queries), making them suitable for transactional application patterns alongside analytical queries.

The fundamental difference from all other Snowflake table types: **PRIMARY KEY, UNIQUE, and FOREIGN KEY constraints are enforced** on hybrid tables. An INSERT that violates a primary key constraint fails with an error. This is the behavior traditional RDBMS practitioners expect but which standard Snowflake tables do not provide.

Hybrid tables support indexes (unique and non-unique) for fast point lookups, which no other Snowflake table type supports. This makes them appropriate for operational applications that need to look up individual rows by key — a pattern that would be expensive on a standard columnar table requiring a full micro-partition scan.

**Limitations:** Hybrid tables cannot be temporary or transient. They cannot exist within transient schemas or transient databases. Their storage cost model is different — row-store storage is more expensive than columnar storage — so they should not be used for large analytical datasets where the row-store component adds cost without benefit.

**Use case:** Operational applications requiring transactional integrity, low-latency single-row lookups, and concurrent read-write patterns — customer-facing applications, order management systems, session state storage — while keeping the data accessible for SQL analytics within Snowflake.

### 7.6 Dynamic Tables

Dynamic tables are defined by a SQL query and are **automatically kept up to date** by Snowflake as their upstream source tables change. They are the declarative ELT automation mechanism — the engineer defines the transformation as a SELECT statement, and Snowflake manages all refresh orchestration.

The `TARGET_LAG` parameter specifies the maximum acceptable staleness — how far behind the dynamic table can lag relative to its upstream sources. Snowflake uses this parameter to determine how frequently to refresh. Setting a 1-minute target lag on a dynamic table driven by a frequently updated staging table results in near-real-time refresh.

Dynamic tables replace complex Streams + Tasks pipeline patterns for many common use cases. They support incremental refresh — only processing the rows that changed since the last refresh, not re-processing the entire dataset — which makes them efficient for large tables with modest change rates.

**Use case:** ELT pipelines where the output needs to stay continuously fresh with minimal lag, replacing the Streams + Tasks pattern where the orchestration overhead is unwanted, caching complex query results that need periodic refresh, and materializing joined or aggregated views as physical tables for BI tool performance.

### 7.7 Apache Iceberg Tables

Iceberg tables store data using the **Apache Iceberg open table format** — a standard that provides ACID guarantees, schema evolution, hidden partitioning, and snapshot-based time travel over data files stored in cloud object storage. The files are Parquet format.

Snowflake supports two storage models for Iceberg tables:

**Snowflake-managed storage:** Snowflake stores the Iceberg table's data files in its own managed cloud storage. The benefit is that these tables behave very similarly to standard Snowflake tables — governance policies, clustering, and other native Snowflake features apply. These tables also support transient mode.

**Customer-managed (external volume) storage:** The Iceberg table's data files are stored in the customer's own cloud storage, accessed through a Snowflake External Volume. This is the model for interoperability — the same Parquet files can be read and written by Snowflake and by other engines (Apache Spark, Trino, Databricks) simultaneously. This is the hybrid lake architecture pattern.

The key architectural distinction from external tables: Iceberg tables support full DML (INSERT, UPDATE, DELETE, MERGE) on customer-managed storage, while external tables are read-only. Iceberg tables also support full schema evolution and maintain their own metadata and snapshot history, enabling time travel through the Iceberg snapshot model.

**Use case:** Multi-engine data lake architectures where data must be accessible to both Snowflake and non-Snowflake compute engines, open data lake implementations where vendor lock-in to Snowflake's proprietary format is undesirable, and scenarios requiring the open Iceberg table format for regulatory or interoperability reasons.

### 7.8 Table Type Comparison

| Property | Permanent | Transient | Temporary | External | Hybrid | Dynamic | Iceberg |
|---|---|---|---|---|---|---|---|
| **Time Travel** | Up to 90 days | 0–1 day | None | None | Limited | Yes | Yes (Iceberg snapshots) |
| **Fail-Safe** | 7 days | None | None | None | Yes | Yes | Yes (Snowflake storage) |
| **DML supported** | Yes | Yes | Yes | No | Yes (enforced) | No (auto-managed) | Yes |
| **Constraints enforced** | No | No | No | No | Yes | N/A | No |
| **Storage location** | Snowflake | Snowflake | Snowflake | External | Snowflake | Snowflake | Snowflake or External |
| **Session-scoped** | No | No | Yes | No | No | No | No |
| **Cloneable** | Yes | Yes | No | No | No | Yes | Yes |
| **Use case** | Production data | Staging/ETL | Session scratch | External lake | OLTP | ELT pipelines | Multi-engine lake |

---

## 8. Views

Views are schema-level objects that define a named query. Querying a view executes the underlying query and returns results. Views do not store data — every query against a view re-executes the view's definition against the underlying tables.

### 8.1 Standard Views

A standard view is a saved SQL query with a name, stored as an object in a schema. It provides a logical abstraction over tables — presenting a curated subset of columns, joining multiple tables, or applying filtering logic without requiring the consumer to understand the underlying structure.

Any user with SELECT on the view can inspect its definition through INFORMATION_SCHEMA or through `SHOW VIEWS`. This transparency is both a feature (developers can see what the view does) and a risk (the underlying schema and logic are exposed to anyone with view access). For use cases where the underlying logic must be protected, secure views are required.

### 8.2 Secure Views

A secure view hides its SQL definition from all users except the view's owner role. Non-owners receive query results from the view but cannot inspect the view's DDL through any metadata mechanism.

Beyond DDL confidentiality, Snowflake intentionally constrains the query optimizer when processing queries against a secure view, preventing certain side-channel inference attacks where an attacker could infer information about filtered data by observing query execution patterns (which micro-partitions were skipped, for example).

**When secure views are mandatory:** Any view included in a Snowflake Data Share must be a secure view — Snowflake enforces this. A regular view containing security logic (filtering by `CURRENT_ROLE()`, for example) is not safe to share because consumers could read the view's definition and understand exactly what data is being withheld from them.

The trade-off is a slight performance cost — the optimizer bypass means some query plan optimizations are unavailable for secure view queries.

### 8.3 Materialized Views

A materialized view stores the results of its query as a physical snapshot in Snowflake's storage, unlike a standard view which re-executes its query on every access. Snowflake automatically keeps the materialized view's snapshot in sync with the underlying base table as the base table changes — the maintenance happens in the background using serverless compute.

The performance benefit is significant for expensive queries: instead of re-executing a complex aggregation over billions of rows every time a BI tool queries the view, the materialized view serves the pre-computed result instantly. The compute cost of the aggregation is amortized over the background refresh, not charged to the querying user's warehouse.

**Important constraints:** A materialized view can only be defined over a single table (joins are not supported in the base query). The base table cannot be an external table. Materialized views require Enterprise Edition or above. The background maintenance uses serverless compute billed to the account, so heavily changing base tables with expensive materialized view definitions can generate significant background compute costs.

**Use case:** Pre-computing expensive aggregations for BI tools, caching results of queries that are expensive but queried frequently, and accelerating dashboard response times for common query patterns.

---

## 9. Stages

Stages are named pointers to storage locations used for loading and unloading data. They are the intermediary between external data sources and Snowflake tables. Every data loading operation in Snowflake involves a stage — the `COPY INTO <table>` command reads from a stage, and the `COPY INTO <location>` command writes to a stage.

The key architectural distinction is between **implicit stages** (user stages and table stages, which exist automatically) and **named stages** (internal and external, which must be created explicitly and are schema-level objects).

### 9.1 User Stages

Every Snowflake user automatically has a personal user stage — a private internal storage area within Snowflake's managed storage. User stages are referenced with the prefix `@~`. They are private to the individual user: no other user can access another user's stage, even with ACCOUNTADMIN.

User stages are appropriate for individual file uploads that a single user will personally load into a table. They are not suitable for collaborative workflows (multiple users working with the same files) or automated pipelines (where no individual user is the logical owner of the data).

### 9.2 Table Stages

Every table automatically has a corresponding table stage — internal storage tied to that specific table, referenced with `@%table_name`. Table stages can be accessed by any role that has OWNERSHIP, INSERT, or other relevant privileges on the table.

Table stages are purpose-built for loading that specific table. They are appropriate when data files are staged immediately before being loaded and there is a clear one-table-to-one-load relationship. Unlike user stages, table stages are accessible to multiple users with appropriate table privileges.

A critical limitation of both user and table stages: they cannot have file formats or copy options configured at the stage level. These must be specified in the `COPY INTO` command instead.

### 9.3 Named Internal Stages

Named internal stages are schema-level objects — they appear in the schema alongside tables and views and are governed by the same privilege model. They are created explicitly with `CREATE STAGE` and can be granted USAGE or READ privileges to specific roles.

Because they are schema-level objects, named internal stages are governed by RBAC, can be included in shares (with limitations), appear in INFORMATION_SCHEMA, and are managed through the full DDL lifecycle (CREATE, ALTER, DROP). They support associated file format definitions and copy options at the stage level, which simplifies COPY INTO commands.

Named internal stages are the correct choice for any automated pipeline, any workflow involving multiple users or multiple target tables, or any scenario requiring explicit access control over the staging area.

### 9.4 Named External Stages

Named external stages are also schema-level objects but they reference storage outside of Snowflake — S3 buckets, Azure Blob containers, GCS buckets. They serve as a pointer with authentication configuration (via a storage integration), URL, and optional file format defaults.

The authentication model for external stages uses **storage integrations** (account-level objects) rather than embedding access keys in the stage definition. The storage integration establishes a trust relationship between Snowflake and the cloud storage provider using IAM roles (AWS), service principals (Azure), or service accounts (GCP). This is the security best practice — no cloud credentials are stored in stage definitions.

External stages are the entry point for bulk data loading from cloud storage lakes, partner data feeds, and any data that originates outside Snowflake. They are also the destination for data unloading operations.

---

## 10. File Formats

A file format is a schema-level object that captures a reusable configuration describing how data files are structured — the file type (CSV, JSON, Parquet, Avro, ORC, XML), delimiters, encoding, compression, null handling, date formats, and many other parsing parameters.

Without file formats as named objects, every COPY INTO command would require the full specification of dozens of parsing parameters inline. Named file formats allow this configuration to be defined once, tested, and then referenced by name in stage definitions and COPY INTO commands.

**File formats as architecture components:** In a multi-team environment, each data source has its own structural characteristics. Creating a named file format per source system (e.g., `salesforce_csv_format`, `oracle_export_json_format`) encapsulates the knowledge of how to parse that source's files into a reusable, version-controlled schema object. New pipelines for the same source simply reference the existing file format.

**Inheritance from stage to COPY command:** File formats can be set at the stage level (a default for all files loaded through that stage), and individual COPY INTO commands can override specific parameters. This hierarchy allows a standard parsing configuration to be the default while permitting per-load exceptions without duplicating the full configuration each time.

---

## 11. Functions and Procedures

Both functions and procedures are schema-level objects that encapsulate reusable logic. They differ fundamentally in their execution model, return semantics, and what they are allowed to do.

### 11.1 User-Defined Functions (UDFs)

A scalar UDF takes one or more input values and returns a single scalar value. It is called inline within a SQL expression — in a SELECT list, a WHERE clause, a JOIN condition — exactly like a built-in SQL function. A UDF that converts a raw string to a standardized format can be applied to a column in a SELECT statement, transforming every row.

UDFs can be written in SQL, JavaScript, Python, Java, or Scala. The programming language choice affects what kinds of transformations are possible: SQL UDFs are for pure SQL expressions, JavaScript UDFs for lightweight logic, and Python/Java/Scala UDFs for complex processing that benefits from external libraries.

**Secure UDFs:** Like secure views, a secure UDF hides its definition from non-owners. Secure UDFs are required when a UDF is included in a share or when the UDF's logic (and the business rules it encodes) must not be visible to users calling the function.

### 11.2 User-Defined Table Functions (UDTFs)

A UDTF returns a set of rows (a table) rather than a single scalar value. It is called in a query using the TABLE() syntax in a FROM clause — it is treated like a table-valued expression that produces rows.

UDTFs are essential for workloads that need to generate multiple output rows from a single input row — for example, parsing a JSON array field and emitting one row per array element, or splitting a time series record into multiple windowed rows.

### 11.3 Stored Procedures

A stored procedure is a named, schema-level object that executes a block of code — written in JavaScript, Python, Java, Scala, or Snowflake Scripting (procedural SQL) — and returns a single value (or nothing). Procedures are called with `CALL procedure_name()`, not invoked inline in a SQL expression.

The key capability that distinguishes procedures from UDFs is that **procedures can execute DDL statements** — CREATE TABLE, ALTER TABLE, DROP TABLE, and so on. UDFs are pure functions that only read data; procedures can mutate the state of the Snowflake account, create objects, and perform administrative operations. This makes procedures the correct implementation for complex ELT workflows, data pipeline orchestration logic, and administrative automation.

### 11.4 The Difference Between Functions and Procedures

| Aspect | UDF / UDTF | Stored Procedure |
|---|---|---|
| **Called from** | Inline in SQL expressions | `CALL` statement, not inline |
| **Returns** | Scalar value (UDF) or table rows (UDTF) | Single value or nothing |
| **Can run DDL?** | No | Yes |
| **Can query tables?** | Yes | Yes |
| **Transaction behavior** | Runs inside the caller's transaction | Can manage its own transactions |
| **Caller's rights vs. Owner's rights** | UDFs run with caller's rights by default | Can be defined as caller's or owner's rights |

**Caller's rights vs. Owner's rights** is a critical security concept for procedures. An **owner's rights procedure** runs with the privileges of the role that owns the procedure — not the role calling it. This allows a procedure to be granted to a role that would normally not have sufficient privileges for the operations the procedure performs, enabling a carefully controlled escalation of privilege. An **caller's rights procedure** runs with the privileges of the calling role — the caller needs all the privileges that the procedure's operations require.

---

## 12. Streams and Tasks

Streams and tasks are the native Snowflake mechanisms for building continuous, incremental data pipelines entirely within the platform. Together, they replace the need for external orchestration tools for many common ELT patterns.

### 12.1 Streams

A stream is a schema-level object that tracks changes to a source object — a table, external table, directory table on a stage, or view — since the last time the stream was consumed. It provides **Change Data Capture (CDC)** functionality natively within Snowflake.

When rows are inserted, updated, or deleted in the source table, the stream does not immediately process those changes — it merely records them as a change log. Querying the stream returns the set of rows that have changed since the stream's offset was last advanced. The stream contains not just the new row values but also metadata columns: `METADATA$ACTION` (INSERT or DELETE), `METADATA$ISUPDATE` (TRUE if the change is part of an update), and `METADATA$ROW_ID` (a unique identifier for the physical row).

**The stream offset** is the marker that determines which changes the stream has already seen versus which changes are new. The offset advances only when a DML statement consumes the stream within a successfully committed transaction. If the consuming transaction fails or is rolled back, the offset does not advance — the same changes remain available in the stream for the next consumption attempt.

**Insert-only streams** are a more limited but less expensive stream type that tracks only INSERT operations, not updates or deletes. They are appropriate for append-only sources (like landing tables in the raw zone) where updates and deletes never occur.

**Stream staleness:** If a stream is not consumed within the retention period of the source table (the `DATA_RETENTION_TIME_IN_DAYS` setting), the stream becomes stale and can no longer be read. This is because the historical change data it references may have been aged out of Time Travel. The `MAX_DATA_EXTENSION_TIME_IN_DAYS` parameter controls how long Snowflake will extend the source table's retention period to keep active streams viable — a stream on a table automatically extends the table's retention if the stream hasn't been consumed.

### 12.2 Tasks

A task is a schema-level object that defines a SQL statement (or a call to a stored procedure) to be executed on a schedule or triggered by a predecessor task. Tasks are the Snowflake-native scheduler for data pipeline steps.

**Scheduling options:** A task can be scheduled with a simple interval (execute every N minutes or hours) or with a CRON expression for precise time-based scheduling. Tasks can also be triggered immediately by a predecessor task in a DAG (covered below).

**Compute for tasks:** A task can execute on a user-managed virtual warehouse (specified in the task definition) or on Snowflake's serverless compute (`USER_TASK_MANAGED_INITIAL_WAREHOUSE_SIZE` set instead of a warehouse name). Serverless tasks are charged at approximately 90% of the equivalent warehouse cost and have no 60-second minimum — they are often more cost-efficient for tasks that execute quickly or infrequently.

**Conditional execution:** The `WHEN` clause of a task allows defining a boolean condition that must be TRUE for the task to actually execute. The most common use of this is `SYSTEM$STREAM_HAS_DATA('stream_name')` — the task only runs if there is new data in the stream it processes. This prevents unnecessary warehouse starts (and the associated 60-second minimum credit charge) when there is nothing to process.

**Task lifecycle states:** Tasks must be explicitly `RESUMED` before they execute on schedule. A newly created task is in a `SUSPENDED` state. This prevents accidental immediate execution of tasks during deployment. After changes to a task's definition, it must be `SUSPENDED` and then `RESUMED` for changes to take effect.

### 12.3 Task DAGs

A **DAG (Directed Acyclic Graph)** of tasks creates a dependency-aware pipeline of multiple steps. A root task runs on a schedule; child tasks are configured with `AFTER <parent_task>` and execute only when their parent completes successfully.

The DAG structure allows complex multi-step pipelines to be defined and managed entirely within Snowflake, without external orchestration tools. A typical ELT pipeline DAG might include: a root task that loads from an external stage into the raw zone → child task 1 that processes the stream on the raw table and loads the curated zone → child task 2 that refreshes downstream aggregations.

Tasks in a DAG are aware of each other's execution status — a child task only starts if the parent succeeded. If the parent fails, downstream tasks are skipped, and the failure is visible in the task execution history. Monitoring task DAG health is done through the `SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY` view.

A DAG can have only one root task (the task that runs on the clock schedule). All other tasks in the DAG are triggered by predecessor tasks. When a DAG's structure changes (new tasks added or existing tasks re-ordered), the entire DAG must be suspended and resumed for the new structure to take effect.

### 12.4 Streams and Tasks Working Together

The streams-and-tasks pattern is the primary way to build incremental, event-driven data pipelines in Snowflake. The pattern is:

A stream sits on a source table and accumulates all new changes (inserts, updates, deletes) since the last consumption. A task is scheduled to check if the stream has data (`SYSTEM$STREAM_HAS_DATA`). When data is present, the task executes a MERGE or INSERT-SELECT statement that reads from the stream and writes processed records to the destination table. Successfully consuming the stream advances the stream's offset, so the same changes are not processed twice. The task then suspends until the next schedule trigger.

This pattern provides exactly-once semantics for incremental processing — each change is processed exactly once, and idempotency is naturally enforced by the stream offset mechanism. It is the Snowflake-native alternative to external streaming platforms for many CDC and incremental ELT use cases.

---

## 13. How the Hierarchy Impacts Architecture

The object hierarchy is not merely a cataloguing system — it drives every significant architectural decision in Snowflake.

**Access control architecture flows down the hierarchy.** Every privilege grant must trace a complete path: USAGE on the account (implicitly, through the role model) → USAGE on the database → USAGE on the schema → object-level privilege (SELECT, INSERT, etc.) → USAGE on a warehouse to execute queries. A gap anywhere in this chain causes access denial. Designing RBAC means designing this privilege chain for every role-to-object combination, which is why the database role pattern exists — to encapsulate the complete chain for a given database scope in a single, grantable unit.

**Time Travel and Fail-Safe decisions are layered.** `DATA_RETENTION_TIME_IN_DAYS` can be set at account, database, schema, and table levels, with lower levels overriding higher ones. An architecture that sets a long retention (30 days) at the account level for all production databases, overrides to 0 days for transient staging schemas, and sets 90 days on specific compliance-critical tables — all using the hierarchy's override mechanism — is far more manageable than trying to set every table's retention individually.

**Object placement drives collaboration and sharing patterns.** Data in the same schema is trivially joinable with unqualified names. Data in different schemas within the same database requires qualified names but no cross-database privilege grants. Data in different databases requires fully qualified names and cross-database USAGE grants. Data in different accounts requires Data Sharing. The placement of objects in the hierarchy determines the friction of access — objects that are frequently joined together should be in the same schema or database.

**Schema choice determines governance overhead.** Standard schemas allow decentralized access control delegation; managed access schemas enforce centralized control. Production data pipelines that serve regulated data should use managed access schemas to prevent accidental privilege grants. Development sandboxes can use standard schemas to reduce the friction of experimentation.

**Virtual warehouse assignment is orthogonal to the storage hierarchy.** A dedicated warehouse per workload type (ingestion, transformation, analytics) enables independent scaling, cost attribution, and priority management without regard to which databases those workloads touch. An ELT transformation warehouse can query across three databases; a BI warehouse can be restricted to read-only analytical databases through RBAC — the warehouse assignment and the database structure are managed independently.

---

## 14. Exam Tips & Common Gotchas

### ⚡ High-Yield Conceptual Points

**Virtual warehouses are account-level objects, not schema-level objects.** They are peers of databases, not children of them. A warehouse is never "inside" a database or schema.

**The three-privilege minimum to query a table:** USAGE on the database, USAGE on the schema, and SELECT (or higher) on the table. Missing any one of these denies access. Additionally, USAGE on a warehouse is required to execute the query.

**Temporary tables shadow permanent tables with the same name in the same schema within the same session.** This causes unexpected behavior when both exist in the same schema.

**Transient tables have 0–1 day Time Travel and no Fail-Safe.** They still exist until explicitly dropped — the word "transient" refers to the absence of long-term protection, not to limited persistence.

**User stages (@~) are private to the creating user and cannot be accessed by anyone else.** Table stages (@%table_name) are shared among roles with relevant table privileges.

**Named stages (internal and external) are schema-level objects** subject to full RBAC — they appear in INFORMATION_SCHEMA and require USAGE privilege to reference.

**Hybrid tables enforce PRIMARY KEY, UNIQUE, and FOREIGN KEY constraints.** Standard tables do not. This is the key distinguishing feature of hybrid tables.

**External tables are read-only.** INSERT, UPDATE, DELETE, and MERGE are not supported on external tables.

**Dynamic tables are defined by a query and auto-refreshed.** They cannot be the target of DML statements — Snowflake manages their contents through the background refresh process.

**Iceberg tables with customer-managed external volumes support multi-engine access** (Spark, Trino, Databricks can all read/write the same files). Iceberg tables with Snowflake-managed storage behave more like standard Snowflake tables.

**Stored procedures can execute DDL; UDFs cannot.** This is the fundamental functional distinction. A procedure can CREATE TABLE; a UDF cannot.

**Owner's rights procedures run as the procedure owner's role, not the caller's role.** This enables controlled privilege escalation — the caller grants access to the procedure but not to the underlying objects.

**Stream offset advances only when the consuming DML transaction commits successfully.** A rolled-back consuming transaction leaves the stream unchanged — no data is lost from the stream.

**`SYSTEM$STREAM_HAS_DATA` prevents empty task executions.** Without this conditional, a task will resume its warehouse even when there is nothing to process, generating a minimum 60-second credit charge.

**Tasks must be explicitly RESUMED after creation.** A new task is `SUSPENDED` by default and does not execute on schedule until resumed.

**Managed access schemas centralize privilege granting** — only the schema owner (or ACCOUNTADMIN/SECURITYADMIN) can grant privileges on objects within the schema, even to the object's own creator.

**INFORMATION_SCHEMA is per-database and has no latency.** ACCOUNT_USAGE is account-wide but has 45-minute to 3-hour latency. Use INFORMATION_SCHEMA for current-state metadata; use ACCOUNT_USAGE for historical analysis and compliance reporting.

**Multi-cluster warehouses scale out (more clusters), not up (larger clusters).** They address concurrency problems, not single-query performance problems. For a slow query, resize the warehouse; for too many queued queries, use multi-cluster.

### 🚫 Classic Exam Traps

| What the Exam Tests | The Correct Answer |
|---|---|
| "Where in the hierarchy do virtual warehouses live?" | Account level — parallel to databases, not inside them |
| "A user has SELECT on a table but gets an access error. What is likely missing?" | USAGE on the database and/or the schema, or USAGE on a warehouse |
| "A developer creates a temporary table with the same name as a permanent table in the same schema. What happens?" | The temporary table shadows the permanent table for the duration of the session |
| "Which table type has no Fail-Safe and a maximum of 1-day Time Travel?" | Transient table |
| "Which table type is session-scoped and invisible to other users?" | Temporary table |
| "Can a permanent table and a temporary table with the same name exist in the same schema?" | Yes — and the temporary table takes precedence in the creating session |
| "Are external tables writable?" | No — external tables are read-only |
| "Which table type enforces PRIMARY KEY constraints?" | Hybrid tables |
| "What is the key feature that distinguishes stored procedures from UDFs?" | Stored procedures can execute DDL; UDFs cannot |
| "A procedure is defined with EXECUTE AS OWNER. What privileges are used during execution?" | The procedure owner's role privileges, not the caller's |
| "What happens to a stream if it is not consumed before the source table's retention period expires?" | The stream becomes stale and unreadable |
| "What does SYSTEM$STREAM_HAS_DATA return if a stream is stale?" | It raises an error — the stream is no longer usable |
| "Does a new task execute on schedule immediately after creation?" | No — tasks start in SUSPENDED state and must be RESUMED |
| "Can a task DAG have multiple root tasks?" | No — a DAG has exactly one root task that runs on the clock schedule |
| "What is the privilege required to use a named stage?" | USAGE on the stage (for reading); READ or WRITE for explicit file operations |
| "Can a hybrid table be created as transient or temporary?" | No — hybrid tables cannot be temporary or transient |
| "What is the difference between multi-cluster scaling and warehouse resizing?" | Multi-cluster adds more clusters of the same size (concurrency); resizing changes the per-cluster size (single-query performance) |
| "Which ACCOUNT_USAGE view tracks column-level access?" | ACCESS_HISTORY |
| "Can database roles be activated directly in a session with USE ROLE?" | No — database roles must be granted to account roles; they cannot be directly activated |

---

## 📚 Reference Links

| Resource | URL |
|---|---|
| Databases, Tables and Views Overview | https://docs.snowflake.com/en/guides-overview-db |
| Working with Temporary and Transient Tables | https://docs.snowflake.com/en/user-guide/tables-temp-transient |
| Hybrid Tables Overview | https://docs.snowflake.com/en/user-guide/tables-hybrid |
| Apache Iceberg Tables | https://docs.snowflake.com/en/user-guide/tables-iceberg |
| Dynamic Tables | https://docs.snowflake.com/en/user-guide/dynamic-tables-about |
| Introduction to Stages | https://docs.snowflake.com/en/user-guide/data-load-overview |
| Overview of Data Loading (Stages) | https://docs.snowflake.com/en/user-guide/data-load-overview |
| Introduction to Streams and Tasks | https://docs.snowflake.com/en/user-guide/data-pipelines-intro |
| Understanding Streams | https://docs.snowflake.com/en/user-guide/streams-intro |
| Understanding Tasks | https://docs.snowflake.com/en/user-guide/tasks-intro |
| Multi-Cluster Warehouses | https://docs.snowflake.com/en/user-guide/warehouses-multicluster |
| Virtual Warehouse Considerations | https://docs.snowflake.com/en/user-guide/warehouses-considerations |
| Managed Access Schemas | https://docs.snowflake.com/en/user-guide/security-access-control-overview#managed-access-schemas |
| Access Control Privileges | https://docs.snowflake.com/en/user-guide/security-access-control-privileges |
| INFORMATION_SCHEMA | https://docs.snowflake.com/en/sql-reference/info-schema |
| ACCOUNT_USAGE Schema | https://docs.snowflake.com/en/sql-reference/account-usage |

---

*© Study Notes for Snowflake Advanced Certification — For educational use only.*
