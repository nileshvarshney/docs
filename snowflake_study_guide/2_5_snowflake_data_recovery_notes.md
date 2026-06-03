# ❄️ Snowflake Advanced Certification Study Notes
## Topic: Data Recovery Solutions

> **Exam Domain:** SnowPro Advanced – Data Engineer / Architect / Administrator
> **Last Updated:** June 2025
> **References:** Official Snowflake Documentation, Snowflake Engineering Blog, Well-Architected Framework

---

## Table of Contents

1. [Snowflake's Data Protection Philosophy](#1-snowflakes-data-protection-philosophy)
2. [Continuous Data Protection (CDP) – The Framework](#2-continuous-data-protection-cdp--the-framework)
3. [Time Travel](#3-time-travel)
   - 3.1 What Time Travel Is and How It Works
   - 3.2 Time Travel by Table Type
   - 3.3 The Retention Period – Configuration and Hierarchy
   - 3.4 Time Travel Operations
   - 3.5 UNDROP – Restoring Dropped Objects
   - 3.6 Time Travel Costs
   - 3.7 Query Performance Impacts
4. [Data Corruption – Impact and Response](#4-data-corruption--impact-and-response)
   - 4.1 What Data Corruption Means in Snowflake
   - 4.2 The Recovery Decision Matrix
   - 4.3 Limitations That Shape Recovery
5. [Zero-Copy Cloning for Recovery](#5-zero-copy-cloning-for-recovery)
   - 5.1 How Zero-Copy Cloning Works
   - 5.2 Cloning as a Pre-Change Safety Net
   - 5.3 Cloning at a Historical Point in Time
   - 5.4 What Gets Cloned and What Doesn't
   - 5.5 Clone Cost Behavior
6. [Fail-Safe](#6-fail-safe)
   - 6.1 What Fail-Safe Is
   - 6.2 The Relationship Between Time Travel and Fail-Safe
   - 6.3 What Fail-Safe Is NOT
   - 6.4 Fail-Safe by Table Type
7. [Disaster Recovery](#7-disaster-recovery)
   - 7.1 Snowflake's Built-In High Availability
   - 7.2 Why High Availability Is Not Disaster Recovery
   - 7.3 Replication Groups
   - 7.4 Failover Groups
   - 7.5 The Failover and Failback Process
   - 7.6 Client Redirect
   - 7.7 RPO and RTO in Snowflake
   - 7.8 Edition Requirements for Disaster Recovery
8. [Choosing the Right Recovery Tool](#8-choosing-the-right-recovery-tool)
9. [Exam Tips & Common Gotchas](#9-exam-tips--common-gotchas)

---

## 1. Snowflake's Data Protection Philosophy

Before examining individual recovery features, it is important to understand the layered model Snowflake uses to protect data and how each layer addresses a different category of risk.

Snowflake's data protection is designed to handle a spectrum of threats, from minor operational mistakes to catastrophic regional failures. These threats are fundamentally different in nature and require fundamentally different recovery mechanisms:

**Operational accidents** are the most common threat — a developer runs a DELETE without a WHERE clause, a pipeline overwrites a table with incorrect data, a schema is dropped by mistake. These events are localized, often noticed quickly, and require fast, granular recovery at the object level. Time Travel and zero-copy cloning address this layer.

**Silent data corruption** is a more subtle threat — incorrect data is written without triggering an error, a transformation bug introduces bad values, or referential integrity violations create orphaned records. These errors are often not discovered until downstream consumers report inconsistencies, potentially days or weeks after the corruption occurred. The recovery window depends on how quickly the corruption is detected relative to the Time Travel retention period.

**Infrastructure failures** are the broadest threat — a cloud region becomes unavailable due to a network outage, a natural disaster, or a cloud provider incident. Individual table recovery tools are useless here; what is needed is the ability to serve all workloads from a completely different account in a different region or cloud provider. Failover groups and Client Redirect address this layer.

Understanding which layer a given incident falls into determines which recovery tool is appropriate. Using the wrong tool for the wrong threat type — trying to address a regional outage with Time Travel, for example — wastes time and compounds the impact.

---

## 2. Continuous Data Protection (CDP) – The Framework

Snowflake groups its data protection features under the umbrella of **Continuous Data Protection (CDP)** — a set of mechanisms that are always active, require no manual backup jobs, and protect data at all times without any operational intervention.

The three pillars of CDP are:

**Time Travel** — the customer-accessible layer of historical data preservation. For a configurable retention period (up to 90 days on Enterprise Edition), any state of any table can be queried, cloned, or restored. This is the primary tool for operational recovery.

**Fail-Safe** — the internal, Snowflake-managed layer of last-resort data preservation. After the Time Travel window expires, Snowflake retains data for an additional 7 days in a protected zone accessible only to Snowflake Support. This is not a customer-accessible recovery tool.

**Replication and Failover** — the cross-account, cross-region layer for disaster recovery scenarios. Databases and account objects are continuously synchronized to a secondary account; in the event of a regional failure, the secondary can be promoted to primary in minutes.

The key conceptual distinction that the exam tests repeatedly: **Time Travel is for customers; Fail-Safe is for Snowflake Support.** This single sentence captures a critical architectural truth that shapes how recovery procedures are designed.

---

## 3. Time Travel

### 3.1 What Time Travel Is and How It Works

Time Travel is Snowflake's ability to access the historical state of any table, schema, or database — as it existed at any point during the retention window — without any prior backup configuration, manual snapshot creation, or operational preparation. It is **automatically enabled for all accounts** and requires no setup.

The mechanism behind Time Travel is Snowflake's immutable micro-partition architecture. When data in a table is modified — whether through an INSERT, UPDATE, DELETE, or DROP — Snowflake does not overwrite the existing micro-partitions. It writes new micro-partitions containing the modified state and retains the original micro-partitions for the duration of the retention period. The retention period specifies how long Snowflake keeps those original micro-partitions accessible for historical queries and recovery operations.

This design has an important architectural consequence: **Snowflake stores only the changed rows, not full table copies, during the retention period**. If 10% of a table's rows are updated, only those rows' before-images are retained in the Time Travel storage, not a complete duplicate of the entire table. However, there is one exception to this rule: when a table is dropped or truncated, Snowflake retains a full copy of the entire table for the retention period, because there are no surviving rows to serve as the "current" state.

### 3.2 Time Travel by Table Type

Time Travel availability and maximum retention vary by table type, making the table type decision one of the most consequential cost and recovery planning choices in any Snowflake architecture.

**Permanent tables** are the only table type with the full range of Time Travel options. On Standard Edition, the retention period can be set between 0 and 1 day. On Enterprise Edition and above, the retention period can be set between 0 and 90 days. This is the table type for production data requiring full recovery capability.

**Transient tables** have a maximum Time Travel retention of 1 day regardless of Snowflake edition. Setting `DATA_RETENTION_TIME_IN_DAYS` to any value greater than 1 on a transient table is an error. Transient tables also have no Fail-Safe period. The combination of limited Time Travel and no Fail-Safe makes transient tables appropriate only for data that can be reproduced from source without risk.

**Temporary tables** are scoped to the session that created them. They support a maximum Time Travel retention of 1 day, but critically, **the retention period is capped by the session lifetime** — when the session ends, the temporary table is dropped and any Time Travel data associated with it is immediately purged, regardless of what the configured retention period says. If a temporary table was set to 1-day retention but the session ends after 2 hours, there is a maximum of 2 hours of Time Travel history, not 24.

**External tables** have no Time Travel. The data lives in external cloud storage, which Snowflake does not manage. There is no Fail-Safe either.

**Hybrid tables** have a more nuanced Time Travel story — they function similarly to permanent tables for Time Travel purposes, but the specific behavior of the row-store component differs from columnar tables. Hybrid tables cannot be undropped with the UNDROP command.

**Dynamic tables** support Time Travel for the duration of their configured retention period.

### 3.3 The Retention Period – Configuration and Hierarchy

The `DATA_RETENTION_TIME_IN_DAYS` parameter controls the Time Travel window. It can be set at four levels, with lower (more specific) levels overriding higher levels:

1. **Account level:** Sets the default for all objects in the account. If no object-level override exists, all tables in the account inherit the account-level setting.
2. **Database level:** Overrides the account-level default for all schemas and tables within the database.
3. **Schema level:** Overrides the database-level default for all tables within the schema.
4. **Table level:** Overrides all higher-level settings for that specific table.

This hierarchy is a powerful governance and cost management tool. A common production pattern sets a 30-day retention at the account level for production databases, overrides to 0 days for transient staging schemas (which should use transient tables anyway), and sets 90-day retention on specific compliance-critical fact tables where regulatory requirements mandate extended audit history.

Setting `DATA_RETENTION_TIME_IN_DAYS = 0` effectively **disables Time Travel** for the object. A dropped object with a 0-day retention period skips the Time Travel period entirely and immediately enters Fail-Safe (for permanent tables) — making UNDROP impossible. This is a sharp edge that must be understood: setting retention to 0 for cost savings removes the ability to UNDROP the object.

**Standard Edition limit:** On Standard Edition, the maximum retention period for permanent tables is 1 day (not 90 days). The 90-day maximum is an Enterprise Edition and above feature. This edition difference is a high-frequency exam topic.

### 3.4 Time Travel Operations

Time Travel supports three distinct operations: historical queries, object cloning at a historical point, and object restoration.

**Historical queries** use the `AT` or `BEFORE` clause in a SELECT statement to specify the point in time to query. The reference can be expressed three ways:

- **TIMESTAMP:** A specific datetime value. The query returns the state of the table as of that exact timestamp.
- **OFFSET:** A negative integer specifying how many seconds to go back from the current time. For example, `OFFSET => -3600` returns the state of the table as it was one hour ago.
- **STATEMENT:** A statement ID (the query ID from a previously executed query). The `AT` clause with a statement ID returns the state immediately after that statement executed; the `BEFORE` clause returns the state immediately before it executed. This is the most precise mechanism when you know exactly which query caused the problem.

**Cloning at a historical point** uses the `CREATE ... CLONE ... AT(TIMESTAMP => ...)` syntax to create a zero-copy clone of an object as it existed at a specific point in time. The clone is an independent, writable copy of the historical state, with no impact on the source table. This is the practical recovery mechanism for most data corruption scenarios — the historical clone is created, validated, and then used to restore the affected rows or replace the corrupted table.

**UNDROP** restores a dropped object (table, schema, or database) to its state at the time it was dropped. UNDROP is covered in detail in the next subsection.

### 3.5 UNDROP – Restoring Dropped Objects

UNDROP is the simplest form of Time Travel recovery: a single command that restores a dropped object to its pre-drop state. It works for tables, schemas, and databases. It does **not** work for hybrid tables.

**Critical constraints for UNDROP:**

The most common mistake is attempting UNDROP when an object with the same name already exists. UNDROP will fail if a table, schema, or database with the same name currently exists in the same location. The existing object must be renamed or dropped before UNDROP can proceed.

When a schema or database is dropped, the child objects within it (tables, views, etc.) are also dropped. UNDROP of the parent schema or database restores all child objects that were within their own retention windows. However, child objects that had already expired from Time Travel before the parent was dropped cannot be restored by UNDROP of the parent.

**Resolving ambiguity with multiple same-named dropped objects:** If a table named `orders` has been dropped and recreated multiple times, and you want to restore a specific version, you cannot assume UNDROP will restore the most useful version. Snowflake restores the most recently dropped version by default. If a different version is needed, the `IDENTIFIER()` clause with the system-generated table ID (queryable from `SNOWFLAKE.ACCOUNT_USAGE.TABLES` where `DELETED IS NOT NULL`) can target a specific historical instance.

### 3.6 Time Travel Costs

Time Travel generates storage costs because historical micro-partition data is retained beyond the current state of the table. Understanding how these costs accumulate is essential for architecture decisions.

**The fundamental cost rule:** Snowflake bills for Time Travel storage based on the amount of data that *changed*, not the total table size. For every row that was updated or deleted, the before-image of that row is retained in Time Travel storage for the retention period. A table with 100GB of data where only 1% of rows change per day accumulates approximately 1GB of daily Time Travel storage per day of retention.

**The exception — DROP and TRUNCATE:** When a table is dropped or truncated, the entire table is retained in Time Travel storage (not just the changed rows), because there are no remaining current rows. Dropping a 500GB table with a 30-day retention period means approximately 500GB of additional storage is billed for the next 30 days for that dropped table alone.

**High-change tables are the biggest cost driver:** Tables that experience frequent bulk updates or heavy DML — large staging tables that are re-populated every run, tables that receive continuous CDC updates — can accumulate Time Travel storage that significantly exceeds the current table size. A 100GB table that gets completely replaced daily will accumulate up to `100GB × retention_days` in Time Travel storage.

**Cost management strategies:**
- Use transient tables for staging and intermediate ELT data (no Fail-Safe, max 1-day Time Travel)
- Set `DATA_RETENTION_TIME_IN_DAYS = 0` for objects where Time Travel is not needed (but accept the loss of UNDROP)
- Apply shorter retention at the database or schema level for non-critical data, overriding longer account-level defaults only for specific tables that genuinely need extended history
- Be aware that **schema and database clones** inherit the source's Time Travel settings — a clone of a large database with a 90-day retention period immediately starts accumulating Time Travel storage costs as data in the clone diverges from the original

### 3.7 Query Performance Impacts

Running Time Travel queries (historical queries using AT/BEFORE) against very large tables with long retention periods can exhibit performance characteristics that differ from queries against current data.

The Snowflake query optimizer generally handles Time Travel queries efficiently for recent historical points. However, querying data that is many days or weeks in the past against a high-change table requires the engine to reconstruct the historical state by combining current micro-partitions with their historical before-images. For tables with very high change rates and long retention periods, this reconstruction work can add latency compared to queries against current data.

**Practical performance considerations:**
- Time Travel queries are most efficient for recent time points (within hours or a few days)
- For deep historical queries (weeks ago on high-change tables), using a pre-created point-in-time clone is more reliable and efficient than querying the source table with a far-past AT clause
- A long-running Time Travel query **delays the progression of historical data into Fail-Safe**. Snowflake will not purge historical micro-partitions into Fail-Safe while an active Time Travel query is reading them. This is an important guarantee (your in-progress recovery query won't be invalidated mid-execution) but also means very long historical queries can delay the cleanup of storage

**Information Schema performance impact of temporary tables:** When sessions accumulate large numbers of undropped temporary tables, queries against the `INFORMATION_SCHEMA.TABLES` and `INFORMATION_SCHEMA.COLUMNS` views become slower because those views must enumerate all temporary tables in the metadata catalog. This is an operational anti-pattern — temporary tables should be explicitly dropped before sessions end, and sessions should be properly terminated rather than left idle.

---

## 4. Data Corruption – Impact and Response

### 4.1 What Data Corruption Means in Snowflake

"Data corruption" in a Snowflake context almost never refers to storage-level physical corruption (bit rot, hardware failure) — Snowflake's cloud storage layer handles this automatically, and physical data corruption is Snowflake's operational responsibility. The corruption scenarios architects and engineers must plan for are **logical corruption** — data that is structurally valid but semantically wrong:

- A pipeline bug that transforms values incorrectly and writes bad data to a table
- An accidental DELETE or UPDATE without a proper WHERE clause
- A MERGE statement with an incorrect join condition that updates the wrong rows
- Business rule changes applied retroactively that produce incorrect historical values
- Cross-table consistency violations where related tables are updated separately and a failure mid-process leaves them in an inconsistent state

### 4.2 The Recovery Decision Matrix

The appropriate recovery approach for logical data corruption depends on two key variables: **how much data was affected** and **how quickly the corruption was detected**.

**Small number of rows, detected quickly (within Time Travel window):**
Query the historical state of the table using the AT/BEFORE clause to identify the correct values. Use a MERGE or UPDATE statement to correct just the affected rows in the current table. This is the most surgical approach — it corrects the problem without touching unaffected data.

**Large number of rows or complete table replacement, detected within Time Travel window:**
Create a zero-copy clone of the table at the last known good timestamp. Validate the clone (check row counts, spot-check values, run data quality checks). Rename the corrupt current table and rename the clone to the production name. Drop the corrupt renamed table when confidence in the restore is established. Alternatively, use an `INSERT OVERWRITE` or `CREATE OR REPLACE TABLE ... CLONE ...` to replace the table entirely.

**Multiple related tables affected (consistency problem), detected within Time Travel window:**
Use a consistent point-in-time clone of all affected tables (specifying the same timestamp for all AT clauses) to create a consistent historical snapshot. Validate the snapshot holistically across all tables before replacing production objects. This preserves the relational consistency that would be violated if each table were restored individually to slightly different points in time.

**Detected after the Time Travel window has expired but within Fail-Safe (7 days after retention ends):**
Contact Snowflake Support. There is no self-service recovery mechanism for data that has moved into Fail-Safe. Support can initiate a recovery process but it requires Snowflake intervention and may take time.

**Detected after both Time Travel and Fail-Safe have expired:**
The data is permanently unrecoverable by any mechanism. Prevention is the only option — the retention period must be long enough to exceed the maximum expected detection lag for corruption in that dataset.

### 4.3 Limitations That Shape Recovery

**The retention period is the hard boundary for all self-service recovery.** Once data ages out of the Time Travel window, only Snowflake Support (through Fail-Safe) can potentially recover it. This makes the retention period configuration one of the most consequential governance decisions in a Snowflake deployment.

**Detection lag determines required retention.** If a compliance-critical table is audited quarterly, and an incorrect transformation could produce wrong values that go unnoticed for months, then a 1-day Time Travel retention is effectively no protection. The retention period must be calibrated to the maximum plausible gap between when corruption occurs and when it would be detected.

**Transient tables have no Fail-Safe backstop.** For a transient table with 1-day retention, data that exits the Time Travel window is permanently gone — there is no 7-day Fail-Safe period as there is for permanent tables. This makes transient tables categorically unsuitable for any data where recovery might be needed more than 1 day after an event.

---

## 5. Zero-Copy Cloning for Recovery

### 5.1 How Zero-Copy Cloning Works

Zero-copy cloning creates an independent copy of a database, schema, or table as a metadata operation — it does not physically copy any data. The clone initially shares all the underlying micro-partitions of the source object. From a storage perspective, the clone is free until either the source or the clone begins diverging through DML operations.

The "zero cost" nature of cloning is what makes it the practical recovery tool of choice. A 5TB production database can be cloned instantaneously with no storage cost and no impact on the source. This makes it safe to create recovery clones preemptively (before risky operations) and exploratively (to investigate what data looked like at a historical point without committing to a recovery action until confident).

### 5.2 Cloning as a Pre-Change Safety Net

The most powerful pattern for using cloning in recovery scenarios is to **create a clone immediately before a risky operation**. A schema migration, a bulk update, a business rule change, a large MERGE — any operation that modifies significant amounts of data should be preceded by a zero-copy clone of the affected tables or schema.

If the operation succeeds cleanly, the clone is dropped. If the operation produces incorrect results or partially fails, the clone is already in place, ready to serve as the restoration point. The recovery is as simple as renaming the clone to replace the corrupted object. This eliminates the dependency on Time Travel and its retention window — the clone is a reliable, immediately accessible snapshot regardless of the retention period configuration.

### 5.3 Cloning at a Historical Point in Time

Zero-copy cloning supports the `AT(TIMESTAMP => ...)` and `AT(STATEMENT => ...)` clauses, allowing a clone to be created from any historical state within the Time Travel window. This is the practical recovery mechanism for most corruption scenarios:

- Identify the last known good state (timestamp or the statement ID of the operation that caused corruption)
- Clone the object at that historical point
- Validate the clone against expected data
- Replace the corrupted production object with the validated clone

The combination of Time Travel (which identifies the correct historical point) and cloning (which creates a safe, independent copy at that point) is more powerful than either mechanism alone. Time Travel alone allows queries but requires care when applying corrections directly to the live table. Cloning provides an isolated environment for validation before any changes are committed to production.

### 5.4 What Gets Cloned and What Doesn't

When a database, schema, or table is cloned, the behavior of child objects requires careful attention:

**What IS included in a clone:**
- Table structure (all columns, data types, clustering keys, constraints)
- All current table data (shared as zero-copy micro-partition references)
- Stages associated with the cloned schema
- File formats
- Sequences (the current value is carried forward)
- Database roles (from the source object)
- Tags and their assignments
- Masking policies and row access policies that are applied to the cloned objects

**What is NOT included in a clone:**
- Grants (privileges) on the cloned objects — the clone starts with no grants; RBAC must be re-established explicitly
- Pipes (Snowpipe) — pipes are not cloned
- Tasks — tasks are not cloned when using a historical AT clause for the clone; they ARE cloned in a current-state clone (without AT clause)
- Streams — streams are not preserved (a cloned table starts with a new, empty stream)
- External tables — while the external table definition may be cloned, the underlying external data is not controlled by Snowflake and is not affected
- Load history of COPY INTO operations — a clone does not inherit the load history of the source table, meaning the same files could be loaded again into the clone

**The grants omission is the most operationally significant.** After restoring a cloned table to production, RBAC must be re-applied. In a managed access schema, this requires the schema owner to re-grant privileges. A recovery playbook should always include explicit re-granting steps.

### 5.5 Clone Cost Behavior

A fresh clone has zero storage cost because it shares all micro-partitions with the source. Storage cost accrues on the clone only when rows are inserted, updated, or deleted in either the clone or the source — those operations write new micro-partitions that are owned by the writing object rather than shared.

An important architectural implication: **a clone of a database with a long Time Travel retention inherits that retention period**. All the Time Travel storage costs for the clone accumulate at the same rate as the source. If a 100GB database with 30-day retention is cloned, the clone immediately starts accumulating storage costs as it diverges, and those costs include Time Travel overhead. Long-lived clones of large databases with long retentions can generate significant ongoing storage costs.

---

## 6. Fail-Safe

### 6.1 What Fail-Safe Is

Fail-Safe is a **7-day internal data recovery period** that begins for permanent tables immediately after their Time Travel retention period expires. During the Fail-Safe period, the historical data that has aged out of Time Travel is not immediately deleted — Snowflake retains it in a protected storage zone for an additional 7 days.

This additional window exists to protect against catastrophic data loss scenarios where Snowflake itself might need to recover data on behalf of a customer — for example, a corruption event that affects Snowflake's own infrastructure rather than a customer's logical operations, or a severe customer incident where the Time Travel window was just barely missed.

### 6.2 The Relationship Between Time Travel and Fail-Safe

Understanding how these two periods relate is critical. For a permanent table with a 7-day Time Travel retention:

```
Data is modified (Day 0)
    ↓
Days 0–7:    TIME TRAVEL WINDOW
             (customer can query, clone, undrop using self-service SQL)
    ↓
Days 7–14:   FAIL-SAFE PERIOD
             (data retained by Snowflake, NOT accessible to customers)
    ↓
Day 14+:     Data is PERMANENTLY PURGED
```

For a permanent table with 30-day Time Travel retention, the total protected window is 37 days (30 days Time Travel + 7 days Fail-Safe). The Fail-Safe period is always 7 days and is not configurable by customers regardless of the Time Travel retention period.

The Fail-Safe period only starts when Time Travel ends. If a table has `DATA_RETENTION_TIME_IN_DAYS = 0`, there is effectively no Time Travel window, and the table enters Fail-Safe immediately upon data changes. This means a permanent table with 0-day retention is still protected by Fail-Safe — it is NOT the same as a transient table with 0-day retention, which has no Fail-Safe at all.

### 6.3 What Fail-Safe Is NOT

**Fail-Safe is not a customer-accessible recovery tool.** This is the most tested distinction in this entire topic area. There is no SQL command a customer can run against Fail-Safe data. There is no Snowflake UI panel for accessing Fail-Safe. The only way to access Fail-Safe data is to contact Snowflake Support and open a support case.

Snowflake Support can initiate Fail-Safe recovery, but it is not a guaranteed, SLA-bound service. Recovery from Fail-Safe is a best-effort internal operation. There is no guaranteed recovery time — it may take days. Fail-Safe should never be relied upon as part of a recovery plan or runbook. It is a last resort with no customer-controlled access.

**Fail-Safe storage is billed to the customer.** The 7-day Fail-Safe period for permanent tables incurs storage costs, billed at the same rate as regular storage. This is part of the reason transient tables are cost-effective for staging data — eliminating Fail-Safe removes 7 days of storage costs for the changed rows.

### 6.4 Fail-Safe by Table Type

| Table Type | Time Travel Max | Fail-Safe | Customer-Accessible? |
|---|---|---|---|
| Permanent (Standard) | 1 day | 7 days | Time Travel: Yes; Fail-Safe: No |
| Permanent (Enterprise+) | 90 days | 7 days | Time Travel: Yes; Fail-Safe: No |
| Transient | 1 day | None | Time Travel: Yes (≤1 day); No backstop |
| Temporary | 1 day or session end (whichever is shorter) | None | Time Travel: Yes (within session); No backstop |
| External | None | None | Neither |
| Hybrid | Similar to Permanent | 7 days | Time Travel: Yes; Fail-Safe: No (cannot UNDROP) |

---

## 7. Disaster Recovery

### 7.1 Snowflake's Built-In High Availability

Every Snowflake account has **built-in high availability within its region** as part of the platform's baseline service. Snowflake's cloud services layer runs across multiple availability zones within the account's region, meaning the failure of a single data center or availability zone does not bring down the Snowflake service. This built-in HA is automatic, requires no customer configuration, and is not the customer's responsibility to design or manage.

However, this within-region HA does not protect against a scenario where the **entire cloud region** becomes unavailable — a rare but real event that major cloud providers have experienced due to widespread outages, severe weather, or infrastructure failures affecting an entire geographic region.

### 7.2 Why High Availability Is Not Disaster Recovery

The distinction between high availability (HA) and disaster recovery (DR) is fundamental to understanding what Snowflake provides by default versus what requires additional configuration:

**High Availability** protects against component failures within a region — a single server, a single availability zone. The system stays operational because redundant components within the same region take over seamlessly. Users typically don't notice. This is what Snowflake provides automatically.

**Disaster Recovery** protects against the entire region failing — when every component in a geographic area becomes unavailable simultaneously. Recovery requires failing over to a completely different account in a different region or cloud provider. Users must be redirected; data must be available from the secondary location. This requires deliberate architectural design, configuration, and testing. This is what Snowflake's Replication and Failover features enable, but they must be explicitly configured.

### 7.3 Replication Groups

A **Replication Group** is a Snowflake object that defines a set of databases (and optionally other account objects) to be replicated from a source account to one or more target accounts, on a configurable schedule. It is the foundational primitive for all cross-account data synchronization in Snowflake.

A replication group specifies:
- **Which objects to replicate:** One or more databases, with optional inclusion of account-level objects (users, roles, warehouses, resource monitors, network policies, etc.)
- **Where to replicate:** One or more target Snowflake accounts, which can be in the same or different regions and on the same or different cloud providers
- **How often to replicate:** A schedule expressed as a fixed interval or a CRON expression

Replication is **asynchronous** — changes written to the primary account are not instantly reflected in the secondary. The lag between primary and secondary is bounded by the replication interval. An organization configuring a 15-minute replication interval accepts that the secondary may lag by up to approximately 15 minutes (potentially up to 30 minutes in practice, since the replication run itself takes time after changes accumulate).

**Point-in-time consistency within a group:** All databases and objects within a single replication group are replicated atomically — they are consistent with each other as of the same point in time. If two databases have a foreign-key-like relationship (orders table referencing customers table), keeping them in the same replication group ensures the replica always has consistent data across both databases. Objects in different replication groups may be at different points in time, which can create cross-group consistency issues if those groups contain related data.

**Replication groups are read-only at the secondary.** The replicated databases in the secondary account are accessible for queries (analytics, reporting, disaster recovery drills) but cannot be written to. All writes must go to the primary.

### 7.4 Failover Groups

A **Failover Group** is a replication group with an additional capability: the secondary account can be **promoted to primary**, making the previously read-only replicated data writable. This is what transforms a read-only replica into a full disaster recovery capability.

The objects that can be included in a failover group span the full account surface:
- Databases and shares
- Users and roles (ensuring authentication works on the secondary after promotion)
- Warehouses (ensuring compute configurations are replicated)
- Resource monitors (ensuring cost controls are in place on the secondary)
- Network policies (ensuring access controls are consistent)
- Account-level parameters
- Security integrations and password policies
- Notifications integrations

The value of including non-data objects in replication cannot be overstated. A replica that only has table data but not the users, roles, and warehouses needed to access that data is not a functioning DR environment. A bulk failover must promote all interdependent failover groups simultaneously to avoid inconsistency — for example, failing over databases but not roles would leave the secondary unable to enforce access controls.

**Failover groups vs. Replication groups:** A replication group supports reading from the secondary but not promotion to primary. A failover group supports both — it extends the replication group with the ability to promote. For DR purposes, failover groups are always the appropriate choice.

### 7.5 The Failover and Failback Process

**Planned Failover (DR Drill):**
A planned failover is executed proactively — for a scheduled DR drill, a region migration, or a test of recovery procedures. Because it is planned, the operations team can:
1. Pause all write operations to the primary account (or accept a brief data loss window)
2. Trigger a final manual refresh of all failover groups to minimize the lag at the moment of promotion
3. Promote the secondary account to primary using a single `ALTER FAILOVER GROUP ... PRIMARY` command per failover group
4. Update Client Redirect to point to the new primary (or it updates automatically if Client Redirect was pre-configured)
5. Validate the new primary account is operational
6. Resume write operations pointing to the new primary

**Unplanned Failover (Actual Disaster):**
An unplanned failover occurs when the primary region becomes unavailable unexpectedly. The operations team:
1. Confirms the primary account is genuinely unavailable (not just temporarily slow)
2. Promotes the secondary account to primary — accepting that there will be data loss equal to the replication lag at the moment of the outage (the RPO)
3. Client Redirect automatically routes traffic to the new primary if pre-configured
4. Validates the new primary account and assesses data loss magnitude
5. Communicates the outage and recovery status to stakeholders

**Failback:**
After the original primary region recovers, failback returns the environment to the original architecture:
1. Enable replication from the promoted secondary (now acting as primary) back to the original account
2. Perform a refresh to synchronize all changes made during the DR period back to the original account
3. Promote the original account back to primary
4. Resume normal operation

### 7.6 Client Redirect

**Client Redirect** solves the most operationally painful aspect of a failover — the need to update connection strings in every application, BI tool, ETL pipeline, and client that connects to Snowflake. Without Client Redirect, a failover requires updating potentially hundreds of configurations across dozens of systems simultaneously, under time pressure, during a stressful incident. This is a major source of extended RTO.

Client Redirect provides a **stable connection URL** (a Connection object in Snowflake) that automatically routes traffic to whichever account is currently primary. Applications connect to the connection URL rather than directly to an account URL. When a failover is executed, the connection object is updated to point to the new primary, and all clients using that connection URL seamlessly reconnect to the correct account without any configuration changes.

Client Redirect is not simply a DNS alias. It is a Snowflake-managed object that understands the primary/secondary relationship and handles the routing at the connection negotiation level. This means it works for all Snowflake drivers, JDBC, ODBC, Python connectors, BI tools, and native Snowflake interfaces.

**Client Redirect requires Business Critical Edition or higher.** It is not available on Standard or Enterprise Edition.

### 7.7 RPO and RTO in Snowflake

**Recovery Point Objective (RPO)** is the maximum amount of data loss an organization can tolerate, measured in time. If an outage occurs and the last successful replication was 12 minutes before the outage, the RPO is at most 12 minutes — everything written in the last 12 minutes may be lost.

In Snowflake, **the replication schedule directly determines the RPO**. An organization configuring a 5-minute replication interval has an RPO of approximately 5 minutes (plus any propagation lag). An organization configuring an hourly interval has an RPO of up to approximately one hour.

**Recovery Time Objective (RTO)** is the maximum acceptable time from when a disaster occurs to when operations are fully restored. RTO in Snowflake is primarily determined by:
1. The time to detect that a failover is needed
2. The time to execute the failover (promote the secondary) — typically minutes with Snowflake's `ALTER FAILOVER GROUP ... PRIMARY`
3. The time to redirect clients to the new primary — nearly instant with Client Redirect; potentially hours without it

Well-configured Snowflake DR architectures can achieve RTO measured in minutes. The primary variables are detection time, the operational team's familiarity with the failover procedure (which requires regular DR drills), and whether Client Redirect is configured.

### 7.8 Edition Requirements for Disaster Recovery

| Feature | Standard | Enterprise | Business Critical |
|---|---|---|---|
| **Database-level replication** | ✅ | ✅ | ✅ |
| **Account object replication** (users, roles, warehouses, policies) | ❌ | ❌ | ✅ |
| **Failover / Failback** (promotion of secondary to primary) | ❌ | ❌ | ✅ |
| **Client Redirect** | ❌ | ❌ | ✅ |
| **Cross-cloud/cross-region replication** | ✅ | ✅ | ✅ |

The pattern is clear: **database replication for read-only purposes is available on all editions**. The full DR feature set — including promotion, account object sync, and Client Redirect — requires **Business Critical Edition or higher**.

This edition gate is one of the most tested distinctions for the DR topic. An organization that needs genuine disaster recovery with automatic client routing and full account object consistency must be on Business Critical or VPS.

---

## 8. Choosing the Right Recovery Tool

The right recovery mechanism depends on the type of incident, the scope of impact, and the time elapsed since the event.

| Scenario | Primary Tool | Secondary Tool | Notes |
|---|---|---|---|
| Accidental DROP TABLE (within retention) | UNDROP TABLE | — | Instant; entire table restored to pre-drop state |
| Accidental DELETE/UPDATE (within retention) | AT/BEFORE query + selective restore | Clone at historical timestamp | Surgical (fix rows) vs. full replace |
| Corruption detected quickly | Clone at last good timestamp | AT/BEFORE query for rows | Validate clone before replacing production |
| Pre-change safety snapshot | Zero-copy clone (pre-operation) | — | Drop clone on success; use it on failure |
| Corruption detected after Time Travel expires | Fail-Safe (Snowflake Support) | — | Not self-service; no SLA; last resort |
| Corruption after Fail-Safe expires | No recovery available | — | Prevention via adequate retention planning |
| Transient table data lost | No recovery | — | Transient tables have no Fail-Safe backstop |
| Regional cloud outage | Failover Group promotion | Client Redirect | Requires Business Critical; secondary must be pre-configured |
| DR drill / planned migration | Planned Failover | Client Redirect | Zero data loss if writes paused during promotion |
| Multiple related tables corrupted | Consistent-timestamp clone all tables | — | Same AT timestamp for all clones preserves consistency |

**The overall principle:** Use Time Travel and cloning for **operational recovery** (anything that can be expressed as "go back to a known good state for specific objects"). Use Failover Groups for **infrastructure recovery** (anything where the Snowflake account itself is unavailable). Never rely on Fail-Safe as an operational recovery tool.

---

## 9. Exam Tips & Common Gotchas

### ⚡ High-Yield Conceptual Points

**Time Travel is customer-accessible; Fail-Safe is not.** The single most important distinction in this entire topic. There is no SQL command to access Fail-Safe data. Only Snowflake Support can recover from Fail-Safe.

**90-day Time Travel requires Enterprise Edition or higher.** On Standard Edition, the maximum retention for permanent tables is 1 day. This is tested frequently.

**Transient tables have NO Fail-Safe.** A transient table with 1-day retention has exactly 1 day of protection. After that, data is permanently gone. No Fail-Safe backstop exists.

**Temporary tables lose Time Travel when the session ends.** Regardless of the configured retention period, a temporary table's Time Travel window is capped by the session lifetime.

**Setting DATA_RETENTION_TIME_IN_DAYS = 0 makes UNDROP impossible.** An object with 0-day retention immediately enters Fail-Safe on drop (for permanent tables) or is immediately purged (for transient). The object cannot be undropped.

**UNDROP fails if an object with the same name exists.** The existing object must be renamed or dropped before UNDROP can proceed.

**Zero-copy clones do not inherit grants from the source.** Privileges must be explicitly re-granted on the clone after creation.

**Clone tasks are not included when cloning with an AT clause.** Tasks are cloned in a current-state clone but not in a historical (AT) clone.

**Clone load history is not preserved.** The same data files can be loaded again into a cloned table because the clone has no COPY INTO history.

**Fail-Safe storage is billed to the customer.** The 7-day Fail-Safe period costs money — this is part of why transient tables are cheaper for staging data.

**Replication is asynchronous — lag equals RPO.** The replication interval determines the maximum data loss if the primary fails immediately after a replication cycle. Actual lag may be up to 2× the interval during a refresh cycle.

**Point-in-time consistency is only guaranteed within a single replication or failover group.** Objects in different groups may be at different points in time.

**Account object replication, Failover/Failback, and Client Redirect require Business Critical.** Database-level replication is available on all editions.

**Client Redirect is the key to achieving low RTO.** Without it, every application, pipeline, and BI tool must have its connection string manually updated during failover — a process that takes hours and is error-prone under stress.

**A Failover Group extends a Replication Group** — it adds the ability to promote the secondary to primary. A Replication Group alone only supports read-only replicas.

**Fail-Safe begins after Time Travel ends, not concurrently.** For a table with 30-day retention, the total protected window is 37 days (30 days Time Travel + 7 days Fail-Safe), not 30 days.

**A permanent table with 0-day retention still has Fail-Safe.** Setting retention to 0 removes Time Travel but not Fail-Safe. This is unlike a transient table with 0-day retention, which has no Fail-Safe at all.

**Long-running Time Travel queries delay data moving to Fail-Safe.** Snowflake will not purge historical micro-partitions while an active query references them.

### 🚫 Classic Exam Traps

| What the Exam Tests | The Correct Answer |
|---|---|
| "How does a customer access Fail-Safe data?" | They cannot — only Snowflake Support can access Fail-Safe data |
| "What is the maximum Time Travel retention on Standard Edition?" | 1 day (not 90 days — that requires Enterprise Edition) |
| "Does a transient table have a Fail-Safe period?" | No — transient tables have no Fail-Safe |
| "A temporary table has a 1-day retention. The session ends after 3 hours. How much history is available?" | Only 3 hours — the session end caps the retention period |
| "DATA_RETENTION_TIME_IN_DAYS is set to 0. Can UNDROP be used?" | No — a 0-day retention means objects immediately exit to Fail-Safe (permanent) or are purged (transient); UNDROP requires Time Travel data |
| "UNDROP TABLE fails. What is the most likely cause?" | A table with the same name already exists in that schema |
| "Does a clone inherit the grants of the source object?" | No — grants are not inherited; they must be re-applied |
| "What is the minimum Snowflake edition for Failover Groups?" | Business Critical |
| "What is the minimum edition for Client Redirect?" | Business Critical |
| "Can database-level replication be done on Standard Edition?" | Yes — database replication is available on all editions; only failover, account object replication, and Client Redirect require Business Critical |
| "What does setting 0-day retention do to a permanent table that is dropped?" | It enters Fail-Safe immediately (no Time Travel window), but Fail-Safe still applies |
| "What is the difference between a replication group and a failover group?" | A failover group can promote the secondary to primary; a replication group creates a read-only replica only |
| "How does Client Redirect reduce RTO?" | It provides a single stable connection URL that automatically routes to the current primary without requiring connection string updates in client applications |
| "Replication is configured with a 10-minute interval. What is the approximate RPO?" | Up to 10 minutes (approximately) — data written in the 10 minutes before the primary fails may be lost |
| "Which replication group properties ensure point-in-time consistency across multiple related databases?" | Place all related databases in the same replication/failover group — they are replicated atomically |
| "Can Fail-Safe data be accessed via SQL?" | No — Fail-Safe is entirely internal to Snowflake's infrastructure |
| "When does the Fail-Safe period start for a table with 30-day retention?" | Day 30 (after the 30-day Time Travel window expires), ending on Day 37 |
| "Are zero-copy clone tasks included when cloning with AT (historical timestamp)?" | No — tasks are only cloned in current-state clones, not in historical AT clones |
| "Does zero-copy cloning impact the performance or availability of the source object?" | No — cloning is a metadata operation with no impact on the source |

---

## 📚 Reference Links

| Resource | URL |
|---|---|
| Understanding and Using Time Travel | https://docs.snowflake.com/en/user-guide/data-time-travel |
| Storage Costs for Time Travel and Fail-Safe | https://docs.snowflake.com/en/user-guide/data-cdp-storage-costs |
| Working with Temporary and Transient Tables | https://docs.snowflake.com/en/user-guide/tables-temp-transient |
| Zero-Copy Cloning | https://docs.snowflake.com/en/user-guide/object-clone |
| Cloning Considerations | https://docs.snowflake.com/en/user-guide/object-clone#cloning-considerations |
| UNDROP TABLE | https://docs.snowflake.com/en/sql-reference/sql/undrop-table |
| UNDROP SCHEMA | https://docs.snowflake.com/en/sql-reference/sql/undrop-schema |
| UNDROP DATABASE | https://docs.snowflake.com/en/sql-reference/sql/undrop-database |
| Introduction to Business Continuity & DR | https://docs.snowflake.com/en/user-guide/replication-intro |
| Introduction to Replication and Failover | https://docs.snowflake.com/en/user-guide/account-replication-intro |
| Failing Over Account Objects | https://docs.snowflake.com/en/user-guide/account-replication-failover-failback |
| CREATE FAILOVER GROUP | https://docs.snowflake.com/en/sql-reference/sql/create-failover-group |
| Client Redirect | https://docs.snowflake.com/en/user-guide/client-redirect |
| Well-Architected Framework – Reliability | https://www.snowflake.com/en/developers/guides/well-architected-framework-reliability/ |

---

*© Study Notes for Snowflake Advanced Certification — For educational use only.*
