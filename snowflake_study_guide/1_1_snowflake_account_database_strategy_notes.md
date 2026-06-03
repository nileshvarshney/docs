# ❄️ Snowflake Advanced Certification Study Notes
## Topic: Account & Database Strategy + Parameter Management

> **Exam Domain:** SnowPro Advanced – Data Engineer / Architect  
> **Last Updated:** June 2025  
> **References:** Official Snowflake Documentation, Snowflake Engineering Blog

---

## Table of Contents

1. [Snowflake Account & Database Strategy Overview](#1-snowflake-account--database-strategy-overview)
2. [Snowflake Parameters – All Levels](#2-snowflake-parameters--all-levels)
   - 2.1 Account Parameters
   - 2.2 Session Parameters
   - 2.3 Object Parameters
3. [Parameter Hierarchy & Relationships](#3-parameter-hierarchy--relationships)
4. [Single Account vs. Multiple Accounts](#4-single-account-vs-multiple-accounts)
5. [Isolating and Segmenting Accounts](#5-isolating-and-segmenting-accounts)
6. [Key Considerations When Defining an Account Strategy](#6-key-considerations-when-defining-an-account-strategy)
7. [Cross-Account Features & Capabilities](#7-cross-account-features--capabilities)
8. [Use Cases for Account Strategies](#8-use-cases-for-account-strategies)
9. [Quick Reference – Key Commands](#9-quick-reference--key-commands)
10. [Exam Tips & Common Gotchas](#10-exam-tips--common-gotchas)

---

## 1. Snowflake Account & Database Strategy Overview

### What is a Snowflake Account?
A **Snowflake account** is the top-level unit of isolation. It defines:
- Cloud provider and region (e.g., AWS `us-east-1`, Azure `eastus`)
- Edition (Standard, Enterprise, Business Critical, Virtual Private Snowflake)
- Billing and credit consumption
- All objects: databases, warehouses, users, roles, integrations

### What is a Snowflake Organization?
An **Organization** is a first-class Snowflake object that links all accounts owned by a single business entity. It enables:
- Centralized account creation and management via the `ORGADMIN` role
- Cross-account replication and failover
- Consolidated billing and usage monitoring
- Seamless cross-region data sharing

> 🔑 **Key Role:** `ORGADMIN` — grants the ability to create, view, and manage all accounts within an organization. The newer `GLOBALORGADMIN` role (in a dedicated Organization Account) is now the recommended approach for managing multi-account organizations at scale.

### Organization Account vs. Regular Account
| Type | Purpose |
|---|---|
| **Organization Account** | Special account for GLOBALORGADMIN; used to manage multi-account orgs and access premium `ORGANIZATION_USAGE` schema views |
| **Regular Account** | Standard Snowflake account for workloads (prod, dev, etc.) |
| **Snowflake Open Catalog Account** | Used specifically for managing Open Catalog catalogs |

### Database Strategy Principles
A well-designed database strategy separates concerns across:

| Dimension | Approach |
|---|---|
| **Environment** | `DEV_`, `QA_`, `PROD_` prefix/suffix naming convention OR separate accounts |
| **Domain / LOB** | One database per business domain (e.g., `FINANCE_DB`, `MARKETING_DB`) |
| **Data Layer** | RAW, CURATED/STAGING, ANALYTICS/PRESENTATION layers as separate databases or schemas |
| **Access Control** | Leverage RBAC with databases as the primary isolation boundary |
| **Data Lifecycle** | Apply `DATA_RETENTION_TIME_IN_DAYS` at DB/schema/table level |

---

## 2. Snowflake Parameters – All Levels

Snowflake provides **three types of parameters** to control behavior of your account, sessions, and objects. All parameters have **default values** set by Snowflake and can be overridden at various levels of the hierarchy.

```sql
-- View all parameters at account level (requires ACCOUNTADMIN or SYSADMIN)
SHOW PARAMETERS IN ACCOUNT;

-- View parameters for the current session
SHOW PARAMETERS IN SESSION;

-- View parameters for a specific object
SHOW PARAMETERS IN DATABASE my_db;
SHOW PARAMETERS IN WAREHOUSE my_wh;
SHOW PARAMETERS IN TABLE my_db.my_schema.my_table;
```

---

### 2.1 Account Parameters

**Account parameters** control global behavior across the entire Snowflake account. They are the **highest level** in the hierarchy and **cannot be overridden** at a lower level.

- Only users with the `ACCOUNTADMIN` role (or a role granted the specific privilege) can set account parameters.
- Once set, account parameters are **visible and binding** for all users and sessions — no user can override them.
- By default, account parameters are **NOT displayed** in `SHOW PARAMETERS` output (you must use `SHOW PARAMETERS IN ACCOUNT`).

**How to Set Account Parameters:**
```sql
-- Requires ACCOUNTADMIN role
ALTER ACCOUNT SET <parameter_name> = <value>;

-- Examples
ALTER ACCOUNT SET REQUIRE_STORAGE_INTEGRATION_FOR_STAGE_CREATION = TRUE;
ALTER ACCOUNT SET NETWORK_POLICY = 'my_network_policy';
ALTER ACCOUNT SET MIN_DATA_RETENTION_TIME_IN_DAYS = 7;
ALTER ACCOUNT SET ALLOW_CLIENT_MFA_CACHING = TRUE;
ALTER ACCOUNT SET MULTI_STATEMENT_COUNT = 0; -- 0 = unlimited statements
ALTER ACCOUNT SET SSO_LOGIN_PAGE = TRUE;
```

**Key Account-Level Parameters:**

| Parameter | Description | Default |
|---|---|---|
| `REQUIRE_STORAGE_INTEGRATION_FOR_STAGE_CREATION` | Requires storage integration for external stage creation | `FALSE` |
| `NETWORK_POLICY` | Restricts account access by IP range | None |
| `MIN_DATA_RETENTION_TIME_IN_DAYS` | Sets a floor on Time Travel retention | `0` |
| `ALLOW_CLIENT_MFA_CACHING` | Allows MFA token caching at client side for batch jobs | `FALSE` |
| `ENABLE_INTERNAL_STAGES_PRIVATELINK` | Allows PrivateLink for internal stages | `FALSE` |
| `SSO_LOGIN_PAGE` | Enables SSO login page | `FALSE` |
| `INITIAL_REPLICATION_SIZE_LIMIT_IN_TB` | Limits initial replication data size | Platform default |
| `SAML_IDENTITY_PROVIDER` | Configures SAML SSO provider at account level | None |

> ⚠️ **Exam Note:** Account parameters **cannot be overridden at any lower level**. This is the key differentiator from session and object parameters.

---

### 2.2 Session Parameters

**Session parameters** control the behavior of individual **user sessions**. They are the most numerous parameter type and the most commonly tuned.

**Hierarchy for Session Parameters:**
```
ACCOUNT (default) → USER (user-level override) → SESSION (session-level override)
```

- `ACCOUNTADMIN` or `SECURITYADMIN` can set session parameter defaults at the **account level** (using `ALTER ACCOUNT`)
- Administrators can set defaults for a **specific user** using `ALTER USER`
- Individual users can override session parameters for their **current session** using `ALTER SESSION`

**How to Set Session Parameters:**
```sql
-- Set default for ALL users at account level (ACCOUNTADMIN)
ALTER ACCOUNT SET DATE_OUTPUT_FORMAT = 'DD/MM/YYYY';

-- Set default for a specific user (SECURITYADMIN or ACCOUNTADMIN)
ALTER USER john_doe SET DATE_OUTPUT_FORMAT = 'MM-DD-YYYY';

-- Override for the current session only (any user for themselves)
ALTER SESSION SET DATE_OUTPUT_FORMAT = 'YYYY-MM-DD';

-- Reset to default
ALTER SESSION UNSET DATE_OUTPUT_FORMAT;
```

**Key Session-Level Parameters:**

| Parameter | Description | Default |
|---|---|---|
| `DATE_OUTPUT_FORMAT` | Format for displaying DATE values | `YYYY-MM-DD` |
| `TIMESTAMP_OUTPUT_FORMAT` | Format for displaying TIMESTAMP values | `YYYY-MM-DD HH24:MI:SS.FF3 TZHTZM` |
| `TIMEZONE` | Session timezone | Account timezone |
| `QUERY_TAG` | Tag attached to queries; queryable in `QUERY_HISTORY` | `''` |
| `STATEMENT_TIMEOUT_IN_SECONDS` | Cancels queries exceeding this duration | `172800` (2 days) |
| `LOCK_TIMEOUT` | Seconds to wait for a lock before aborting | `43200` |
| `TRANSACTION_DEFAULT_ISOLATION_LEVEL` | Default transaction isolation level | `READ COMMITTED` |
| `USE_CACHED_RESULT` | Whether to use cached query results | `TRUE` |
| `AUTOCOMMIT` | Auto-commit DML statements | `TRUE` |
| `BINARY_OUTPUT_FORMAT` | Format for binary data output | `HEX` |
| `ERROR_ON_NONDETERMINISTIC_MERGE` | Raises error for ambiguous MERGE operations | `TRUE` |
| `ROWS_PER_RESULTSET` | Max rows returned (0 = unlimited) | `0` |
| `MULTI_STATEMENT_COUNT` | Number of SQL statements per API call | `1` |
| `LOG_LEVEL` | Severity level for logging (DEBUG, INFO, WARN, ERROR, FATAL, OFF) | `OFF` |
| `TRACE_LEVEL` | Level for trace event ingestion | `OFF` |

> 🔑 **Exam Note:** `QUERY_TAG` is a session parameter — all queries in a tagged session are tagged and queryable from `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY`. Very useful for cost attribution.

---

### 2.3 Object Parameters

**Object parameters** apply to specific Snowflake objects. The applicable objects are:
- **Warehouses** (Virtual Warehouses)
- **Databases**
- **Schemas**
- **Tables**

**Hierarchy for Object Parameters (Database Objects):**
```
ACCOUNT → DATABASE → SCHEMA → TABLE/OBJECT
```

**Warehouse Exception:** Warehouses have their own flat hierarchy — they do **not** follow the database hierarchy. A warehouse parameter set at account level can be overridden at the warehouse level only.

**How to Set Object Parameters:**
```sql
-- Set at ACCOUNT level (becomes default for all objects)
ALTER ACCOUNT SET DATA_RETENTION_TIME_IN_DAYS = 30;

-- Override at DATABASE level
ALTER DATABASE my_db SET DATA_RETENTION_TIME_IN_DAYS = 14;

-- Override at SCHEMA level
ALTER SCHEMA my_db.my_schema SET DATA_RETENTION_TIME_IN_DAYS = 7;

-- Override at TABLE level
ALTER TABLE my_db.my_schema.my_table SET DATA_RETENTION_TIME_IN_DAYS = 1;

-- Warehouse-specific object parameter
ALTER WAREHOUSE my_wh SET MAX_CONCURRENCY_LEVEL = 8;
ALTER WAREHOUSE my_wh SET STATEMENT_QUEUED_TIMEOUT_IN_SECONDS = 300;
```

**Key Object-Level Parameters:**

| Parameter | Applicable To | Description | Default |
|---|---|---|---|
| `DATA_RETENTION_TIME_IN_DAYS` | Account, DB, Schema, Table | Time Travel window in days | `1` (Standard); up to `90` (Enterprise+) |
| `MAX_DATA_EXTENSION_TIME_IN_DAYS` | Account, DB, Schema, Table | Max days Snowflake can extend Time Travel for streams | `14` |
| `DEFAULT_DDL_COLLATION` | Account, DB, Schema, Table | Default collation spec for new string columns | `''` |
| `PIPE_EXECUTION_PAUSED` | Pipe | Whether a pipe's auto-ingest is paused | `FALSE` |
| `MAX_CONCURRENCY_LEVEL` | Warehouse | Max number of concurrent SQL executions per cluster | `8` |
| `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS` | Warehouse | Time a query can stay in queue before being cancelled | `0` (no timeout) |
| `STATEMENT_TIMEOUT_IN_SECONDS` | Warehouse, Account, User | Cancels queries exceeding this duration | `172800` |
| `AUTO_SUSPEND` | Warehouse | Seconds of inactivity before auto-suspend | `600` |
| `AUTO_RESUME` | Warehouse | Whether warehouse auto-resumes on query | `TRUE` |
| `ENABLE_QUERY_ACCELERATION` | Warehouse | Enables Query Acceleration Service | `FALSE` |

---

## 3. Parameter Hierarchy & Relationships

```
┌─────────────────────────────────────────────────────────────────┐
│                    PARAMETER TYPE HIERARCHY                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ACCOUNT PARAMETERS                                             │
│  ┌─────────────────────────────┐                               │
│  │   ACCOUNT (only level)      │  ← Set by ACCOUNTADMIN only   │
│  │   Cannot be overridden      │  ← No lower-level override    │
│  └─────────────────────────────┘                               │
│                                                                 │
│  SESSION PARAMETERS                                             │
│  ┌─────────────────────────────┐                               │
│  │   ACCOUNT (default)         │  ← ALTER ACCOUNT SET ...      │
│  │          ↓                  │                               │
│  │   USER (override default)   │  ← ALTER USER SET ...         │
│  │          ↓                  │                               │
│  │   SESSION (override current)│  ← ALTER SESSION SET ...      │
│  └─────────────────────────────┘                               │
│                                                                 │
│  OBJECT PARAMETERS (Database objects)                           │
│  ┌─────────────────────────────┐                               │
│  │   ACCOUNT (default)         │  ← ALTER ACCOUNT SET ...      │
│  │          ↓                  │                               │
│  │   DATABASE (override)       │  ← ALTER DATABASE SET ...     │
│  │          ↓                  │                               │
│  │   SCHEMA (override)         │  ← ALTER SCHEMA SET ...       │
│  │          ↓                  │                               │
│  │   TABLE/OBJECT (override)   │  ← ALTER TABLE SET ...        │
│  └─────────────────────────────┘                               │
│                                                                 │
│  OBJECT PARAMETERS (Warehouse – flat, no hierarchy)             │
│  ┌─────────────────────────────┐                               │
│  │   ACCOUNT (default)         │  ← ALTER ACCOUNT SET ...      │
│  │          ↓                  │                               │
│  │   WAREHOUSE (override)      │  ← ALTER WAREHOUSE SET ...    │
│  └─────────────────────────────┘                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Override Resolution Rules

1. **Most specific wins** — a parameter set at a lower level always overrides a higher-level default (except for Account Parameters which are absolute).
2. **Account Parameters** — set once, apply everywhere, no override possible.
3. **Session Parameters** — session-level setting takes priority over user-level, which takes priority over account-level default.
4. **Object Parameters** — table-level beats schema-level beats database-level beats account-level.
5. **Warehouses** — warehouse-level beats account-level (no intermediate hierarchy).
6. **Telemetry parameters (LOG_LEVEL, TRACE_LEVEL)** — when set in **both** session and object hierarchies, the **most verbose** level wins.

### Viewing Parameters
```sql
-- All parameters in current session
SHOW PARAMETERS IN SESSION;

-- All parameters for a user
SHOW PARAMETERS IN USER john_doe;

-- All account-level parameters
SHOW PARAMETERS IN ACCOUNT;

-- Parameters for a specific object
SHOW PARAMETERS IN DATABASE my_db;
SHOW PARAMETERS IN SCHEMA my_db.my_schema;
SHOW PARAMETERS IN TABLE my_db.my_schema.my_table;
SHOW PARAMETERS IN WAREHOUSE my_wh;

-- Filter to a specific parameter
SHOW PARAMETERS LIKE 'DATA_RETENTION%' IN ACCOUNT;
```

> 💡 **Default Behavior of SHOW PARAMETERS:** Without modifiers, `SHOW PARAMETERS` only displays **session parameters**. Use `IN ACCOUNT` or `IN <object>` to see other types.

---

## 4. Single Account vs. Multiple Accounts

### Single Account Strategy

**Definition:** One primary Snowflake account containing all environments (dev, QA, prod) and all business units, distinguished by naming conventions and RBAC.

#### ✅ Benefits of a Single Account

| Benefit | Details |
|---|---|
| **Simplified Management** | One set of integrations (SSO, ELT tools, BI tools), one ACCOUNTADMIN to manage |
| **Reduced Configuration Overhead** | Third-party tools (BI, ELT, SSO) configured once |
| **Zero-Copy Cloning Across Environments** | Clone `PROD_MY_DB` → `DEV_MY_DB` instantly within the same account; cross-account cloning requires replication |
| **Simplified Data Sharing** | No replication needed; share views/tables directly using RBAC |
| **Unified Billing** | Single credit pool; easier to track costs using resource monitors and query tags |
| **Easier Governance** | Centralized RBAC, policies (masking, row access), and auditing in one place |
| **Faster CI/CD** | Schema changes can be tested and promoted without cross-account data movement |

#### ❌ Limitations of a Single Account

| Limitation | Details |
|---|---|
| **Naming Convention Discipline Required** | Must use `PROD_`, `DEV_`, `QA_` prefixes — hard to enforce at scale |
| **Risk of Cross-Environment Contamination** | Developer queries can accidentally run against prod objects |
| **No Billing Isolation** | Cannot generate separate invoices per environment or BU within one account |
| **Network Policy Complexity** | Single network policy applies broadly; granular control per environment is harder |
| **Compliance Constraints** | Some regulations may require physical isolation (separate accounts) for production data |
| **Shared Credit Quota** | All warehouses compete for the same account-level resource pool |
| **Edition Constraint** | All workloads must operate on the same Snowflake edition |

---

### Multiple Account Strategy

**Definition:** Two or more Snowflake accounts — often one per environment (dev, QA, prod) or per region/cloud provider — managed under one Organization.

#### ✅ Benefits of Multiple Accounts

| Benefit | Details |
|---|---|
| **Hard Environment Isolation** | Physical boundary between prod and non-prod; eliminates accidental access |
| **Separate Billing per Environment/BU** | Independent invoices, cost centers, or chargebacks |
| **Separate Network Policies** | Tighter access controls on the production account |
| **Regulatory / Compliance Alignment** | Meet data residency, sovereignty, or isolation requirements per region |
| **Independent Editions** | Run Business Critical in prod, Standard in dev — optimizing cost |
| **Disaster Recovery & Geo-Redundancy** | Replicate critical data to a secondary account in another region |
| **Tenant Isolation for SaaS** | Each customer or tenant gets their own isolated Snowflake account |
| **Separate Upgrade/Maintenance Windows** | Non-prod can test new Snowflake feature bundles before prod |

#### ❌ Limitations of Multiple Accounts

| Limitation | Details |
|---|---|
| **Operational Complexity** | Multiple ACCOUNTADMIN setups, integrations, user provisioning |
| **Tool Configuration Multiplied** | Every BI tool, ELT connector, and SSO must be configured N times |
| **Cross-Account Data Movement Costs** | Replication incurs compute and, for cross-region, egress costs |
| **No Native Zero-Copy Cloning** | Cannot clone across accounts; replication must be used |
| **RBAC Not Shared** | Roles, policies, and users must be managed independently per account |
| **Increased Governance Burden** | Masking policies, row access policies, and tags must be duplicated |
| **Latency for Cross-Account Access** | Cross-region replication introduces lag; secondary accounts may have stale data |

---

### Decision Matrix: Single vs. Multiple Accounts

| Factor | Lean Single | Lean Multiple |
|---|---|---|
| **Organization size** | SMB / Mid-market | Large Enterprise |
| **Compliance/regulation** | None or light | HIPAA, PCI-DSS, SOC2, GDPR (data residency) |
| **Environment isolation** | RBAC sufficient | Physical isolation required |
| **Billing** | Consolidated billing OK | Separate cost centers needed |
| **Team size** | Small/centralized data team | Multiple independent teams |
| **DR requirements** | Low | High (RPO/RTO SLAs defined) |
| **Multi-cloud or multi-region** | Not needed | Required by architecture |

---

## 5. Isolating and Segmenting Accounts

### Methods of Isolation Within a Single Account
- **Naming Conventions:** `PROD_`, `DEV_`, `QA_` database prefixes
- **RBAC:** Separate roles per environment (e.g., `PROD_READER`, `DEV_WRITER`)
- **Resource Monitors:** Separate credit quotas per environment via resource monitors
- **Network Policies:** Applied at user or account level
- **Managed Access Schemas:** Centralize privilege grants; prevent object owners from granting privileges

### Methods of Isolation Across Multiple Accounts
- **ORGADMIN / GLOBALORGADMIN:** Create and manage accounts within the organization
- **Account Replication Groups:** Synchronize databases, shares, and other objects from primary to secondary accounts
- **Failover Groups:** For disaster recovery — promote a secondary account to primary
- **Private Listings:** Share data products across accounts within the same org without exposing to the Marketplace

```sql
-- Create a new account (ORGADMIN role required)
CREATE ACCOUNT my_dev_account
  ADMIN_NAME = 'dev_admin'
  ADMIN_PASSWORD = 'Str0ngPa$$word!'
  EMAIL = 'dev-team@company.com'
  EDITION = STANDARD
  REGION = 'AWS_US_EAST_1';

-- Create a replication group (source account, ACCOUNTADMIN)
CREATE REPLICATION GROUP prod_rg
  OBJECT_TYPES = DATABASES, SHARES, ROLES
  ALLOWED_DATABASES = PROD_DB
  ALLOWED_ACCOUNTS = my_org.my_dr_account
  REPLICATION_SCHEDULE = '10 MINUTE';

-- Create failover group for DR
CREATE FAILOVER GROUP prod_fg
  OBJECT_TYPES = DATABASES, USERS, ROLES, WAREHOUSES, RESOURCE MONITORS,
                 NETWORK POLICIES, ACCOUNT PARAMETERS
  ALLOWED_DATABASES = PROD_DB
  ALLOWED_ACCOUNTS = my_org.my_dr_account;
```

---

## 6. Key Considerations When Defining an Account Strategy

### 1. Data Residency & Sovereignty
- Where must data physically reside? (EU, US, APAC)
- Does data cross national borders during replication?
- Regulatory frameworks: GDPR (EU), PIPEDA (Canada), PDPA (Singapore), LGPD (Brazil)

### 2. Snowflake Edition
- **Standard:** Basic features; Time Travel up to 1 day
- **Enterprise:** Multi-cluster warehouses, Time Travel up to 90 days, Dynamic Data Masking, Column-Level Security
- **Business Critical:** HIPAA/PCI compliance, Private Link, Tri-Secret Secure, Customer Managed Keys (Bring Your Own Key)
- **Virtual Private Snowflake (VPS):** Dedicated metadata store; fully isolated

> 🔑 Different accounts in the same org can have **different editions** — use Business Critical for prod and Standard for dev to save costs.

### 3. Cloud Provider & Region Selection
- Snowflake supports AWS, Azure, and GCP
- Choose region close to your source data and consumers to minimize latency/egress
- Cross-cloud replication is supported but incurs cross-cloud egress costs

### 4. Compliance and Security Requirements
- Does PHI/PCI data require Business Critical or VPS?
- Are there audit requirements mandating separate accounts?
- HIPAA BAA is available at Business Critical tier only

### 5. Cost Model
- Credits billed per account; no cross-account pooling
- Cross-region replication incurs data transfer costs
- Separate accounts = separate invoices (good for chargebacks, complex for consolidated view — mitigated by ORGADMIN usage views)

### 6. Team & Operational Capacity
- Do you have the DevOps maturity to manage multiple accounts?
- CI/CD pipelines must handle cross-account deployments (e.g., using Terraform, SnowCLI, or Flyway)

### 7. Data Sharing Requirements
- Same account: zero-copy, near-instant via RBAC
- Same org, different accounts: use Listings or Replication Groups
- External consumers: Snowflake Marketplace or Private Listings

### 8. Disaster Recovery Objectives
- **RPO (Recovery Point Objective):** How much data can you lose? Replication frequency must be ≤ RPO
- **RTO (Recovery Time Objective):** How fast must you recover? Failover groups allow rapid promotion

---

## 7. Cross-Account Features & Capabilities

These Snowflake features work **across accounts** within the same Organization:

| Feature | Description | Requirement |
|---|---|---|
| **Replication Groups** | Replicate databases, shares, tasks, users, roles, policies, etc. to another account | Standard Edition+ |
| **Failover Groups** | Full account object replication + ability to promote secondary → primary for DR | Standard Edition+ |
| **Cross-Region Data Sharing** | Share live data to consumers in a different region via auto-replicated shares | Standard Edition+ |
| **Cross-Cloud Auto-Fulfillment** | Marketplace listings auto-replicated to consumer's cloud/region | Marketplace provider |
| **ORGADMIN / GLOBALORGADMIN** | Manage all accounts, view org-wide usage, create accounts | ORGADMIN role |
| **Organization Usage Views** | `SNOWFLAKE.ORGANIZATION_USAGE` — aggregate billing/usage across all accounts | ORGADMIN |
| **Private Listings** | Share data products securely with specific accounts within or outside your org | Data Sharing enabled |
| **Snowflake Marketplace** | Publish or consume data products across any Snowflake account globally | Any Edition |
| **Tri-Secret Secure / BYOK** | Customer-managed encryption keys (cross-cloud HSM integration) | Business Critical |
| **PrivateLink** | Private connectivity from consumer VPC to Snowflake (no public internet) | Business Critical |

### Organization Usage Views (Key Views)

```sql
-- Switch to ORGADMIN role to access org-level views
USE ROLE ORGADMIN;
USE DATABASE SNOWFLAKE;
USE SCHEMA ORGANIZATION_USAGE;

-- View all accounts in the organization
SELECT * FROM SNOWFLAKE.ORGANIZATION_USAGE.ACCOUNTS;

-- View credit usage across all accounts
SELECT * FROM SNOWFLAKE.ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY;

-- View contract usage
SELECT * FROM SNOWFLAKE.ORGANIZATION_USAGE.CONTRACT_ITEMS;

-- View metered storage usage across accounts
SELECT * FROM SNOWFLAKE.ORGANIZATION_USAGE.STORAGE_DAILY_HISTORY;
```

---

## 8. Use Cases for Account Strategies

### Use Case 1: Environment Isolation (Dev / QA / Prod)

**Recommended Strategy:** Multiple accounts OR single account with strong naming conventions

| Approach | Accounts | When to Use |
|---|---|---|
| Single Account | 1 | Small teams; fast cloning critical; low compliance burden |
| Multiple Accounts | 3 (dev, qa, prod) | Enterprise; strict prod access controls; separate billing per env |

**Pattern:**
```
ORG
├── PROD account  (Business Critical, AWS us-east-1)
├── QA account    (Enterprise, AWS us-east-1)
└── DEV account   (Standard, AWS us-east-1)
```

### Use Case 2: Multi-Tenant SaaS Application

**Strategy:** One Snowflake account per customer (tenant) OR shared account with database-per-tenant

| Sub-Pattern | Isolation | Cost | Complexity |
|---|---|---|---|
| Account per tenant | Maximum | High | High |
| Database per tenant | High | Medium | Medium |
| Schema per tenant | Medium | Low | Low |

**When to use account-per-tenant:** When customers have contractual isolation requirements, separate billing, or compliance mandates.

### Use Case 3: Geo-Distributed / Global Operations

**Strategy:** Primary account in home region + secondary accounts (replicated) in each target region

```
ORG
├── PRIMARY account  (AWS us-east-1) — source of truth
├── EMEA account     (Azure West Europe) — replicated for EU users
└── APAC account     (AWS ap-southeast-1) — replicated for APAC users
```

**Key Features:** Replication Groups, Cross-Region Data Sharing, Cross-Cloud Auto-Fulfillment

### Use Case 4: Disaster Recovery (DR)

**Strategy:** Failover Group from primary to a secondary account in a different region

```sql
-- Initiate failover (run on secondary account)
ALTER FAILOVER GROUP prod_fg PRIMARY;
```

**RPO:** Controlled by replication schedule (minimum: continuous replication with Business Critical)
**RTO:** Minutes (time to promote secondary to primary + reconnect clients)

### Use Case 5: Data Marketplace / Data Sharing Provider

**Strategy:** Single provider account publishes to Marketplace; consumers across any cloud/region

- Use **Cross-Cloud Auto-Fulfillment** for consumers in other regions/clouds
- Use **Private Listings** for controlled sharing with specific consumer accounts
- Use **Replication Groups** for internal sharing across your org's accounts

### Use Case 6: Regulatory Compliance Segmentation

**Strategy:** Separate accounts for different regulatory domains

```
ORG
├── PCI_PROD account   (Business Critical — cardholder data, PCI-DSS scope)
├── HIPAA_PROD account (Business Critical — PHI data, HIPAA BAA)
└── GENERAL_PROD account (Enterprise — non-regulated data)
```

### Use Case 7: Business Unit Autonomy

**Strategy:** Each BU gets its own account with independent billing and administration

- BU accounts connected under the same Organization
- Central IT manages ORGADMIN; each BU manages their own ACCOUNTADMIN
- Cross-BU data sharing via Private Listings or Replication Groups

---

## 9. Quick Reference – Key Commands

```sql
-- ─────────────────────────────────────────────
-- ACCOUNT PARAMETER MANAGEMENT
-- ─────────────────────────────────────────────

-- Set an account parameter (ACCOUNTADMIN)
ALTER ACCOUNT SET <param> = <value>;

-- Unset an account parameter (resets to default)
ALTER ACCOUNT UNSET <param>;

-- View all account-level parameters
SHOW PARAMETERS IN ACCOUNT;


-- ─────────────────────────────────────────────
-- SESSION PARAMETER MANAGEMENT
-- ─────────────────────────────────────────────

-- Set account-level default for session param
ALTER ACCOUNT SET TIMEZONE = 'America/New_York';

-- Set user-level default
ALTER USER john_doe SET TIMEZONE = 'Europe/London';

-- Set for current session only
ALTER SESSION SET TIMEZONE = 'UTC';
ALTER SESSION SET QUERY_TAG = 'ETL_PIPELINE_2025';

-- Unset session parameter
ALTER SESSION UNSET TIMEZONE;

-- Show current session parameters
SHOW PARAMETERS IN SESSION;
SHOW PARAMETERS IN USER john_doe;


-- ─────────────────────────────────────────────
-- OBJECT PARAMETER MANAGEMENT
-- ─────────────────────────────────────────────

-- Time Travel at different levels
ALTER ACCOUNT  SET DATA_RETENTION_TIME_IN_DAYS = 30;
ALTER DATABASE my_db SET DATA_RETENTION_TIME_IN_DAYS = 14;
ALTER SCHEMA   my_db.my_schema SET DATA_RETENTION_TIME_IN_DAYS = 7;
ALTER TABLE    my_db.my_schema.my_table SET DATA_RETENTION_TIME_IN_DAYS = 0;

-- Warehouse object parameters
ALTER WAREHOUSE my_wh SET MAX_CONCURRENCY_LEVEL = 4;
ALTER WAREHOUSE my_wh SET STATEMENT_QUEUED_TIMEOUT_IN_SECONDS = 120;
ALTER WAREHOUSE my_wh SET ENABLE_QUERY_ACCELERATION = TRUE;

-- Show object parameters
SHOW PARAMETERS IN DATABASE my_db;
SHOW PARAMETERS IN WAREHOUSE my_wh;
SHOW PARAMETERS LIKE 'DATA_RETENTION%' IN ACCOUNT;


-- ─────────────────────────────────────────────
-- ORGANIZATION MANAGEMENT
-- ─────────────────────────────────────────────

-- View all accounts in the org (ORGADMIN)
USE ROLE ORGADMIN;
SHOW ACCOUNTS;

-- Create a new account (ORGADMIN)
CREATE ACCOUNT dev_account
  ADMIN_NAME = 'dev_admin'
  ADMIN_PASSWORD = 'Str0ng!'
  EMAIL = 'admin@company.com'
  EDITION = STANDARD
  REGION = 'AWS_US_EAST_1';

-- Enable replication for an account
SELECT SYSTEM$GLOBAL_ACCOUNT_SET_PARAMETER(
  'my_org.target_account',
  'ENABLE_ACCOUNT_DATABASE_REPLICATION',
  'true'
);

-- Create replication group
USE ROLE ACCOUNTADMIN;
CREATE REPLICATION GROUP my_rg
  OBJECT_TYPES = DATABASES, SHARES
  ALLOWED_DATABASES = PROD_DB
  ALLOWED_ACCOUNTS = my_org.dr_account
  REPLICATION_SCHEDULE = '15 MINUTE';

-- Refresh replication manually
ALTER REPLICATION GROUP my_rg REFRESH;
```

---

## 10. Exam Tips & Common Gotchas

### ⚡ High-Yield Exam Topics

1. **Account parameters CANNOT be overridden at any lower level** — this is a hard rule and a frequent exam question.

2. **Session parameter hierarchy:** ACCOUNT → USER → SESSION. Each lower level overrides the higher.

3. **Object parameter hierarchy (database objects):** ACCOUNT → DATABASE → SCHEMA → TABLE/OBJECT.

4. **Warehouses have a flat hierarchy** — only ACCOUNT and WAREHOUSE level; no intermediate steps.

5. **`DATA_RETENTION_TIME_IN_DAYS`** is an **object parameter** (not account-only). It can be set at account, database, schema, and table levels, with lower levels overriding higher.

6. **Standard Edition**: Time Travel max = **1 day**. **Enterprise+**: Time Travel max = **90 days**.

7. **`SHOW PARAMETERS`** without modifiers shows only **session parameters** by default.

8. **ORGADMIN** manages accounts within the organization; **ACCOUNTADMIN** manages a single account's objects.

9. **Cross-account zero-copy cloning is NOT supported** — you need replication; cloning only works within the same account.

10. **Failover Groups** support full account object replication (users, roles, warehouses, network policies, account parameters) — Replication Groups support a subset.

11. **`QUERY_TAG`** is a session parameter, not an object parameter. It's set per session.

12. **`STATEMENT_TIMEOUT_IN_SECONDS`** can be set at account, user/session, AND warehouse levels — warehouse-level overrides others for queries running on that warehouse.

13. **MFA caching (`ALLOW_CLIENT_MFA_CACHING`)** is an **account parameter** — cannot be set at session level by individual users.

14. **Business Critical** is required for: PrivateLink, HIPAA BAA, Tri-Secret Secure (BYOK), PHI data, PCI-DSS compliance.

### 🚫 Common Mistakes

| Mistake | Correct Understanding |
|---|---|
| Assuming cloning works across accounts | Cloning = same account only; use replication for cross-account |
| Assuming `SHOW PARAMETERS` shows all types | Default shows session params only; use `IN ACCOUNT` or `IN <object>` |
| Thinking multiple accounts share users/roles | Each account has independent users, roles, and policies |
| Assuming ORGADMIN can manage objects within accounts | ORGADMIN manages the org and creates accounts; ACCOUNTADMIN manages within an account |
| Setting account parameters with SYSADMIN | Account parameters require ACCOUNTADMIN (or a role with the specific privilege) |
| Assuming warehouse parameters follow DB hierarchy | Warehouses are flat: only account-level and warehouse-level |

---

## 📚 Reference Links

| Resource | URL |
|---|---|
| Snowflake Parameters Reference | https://docs.snowflake.com/en/sql-reference/parameters |
| ALTER ACCOUNT | https://docs.snowflake.com/en/sql-reference/sql/alter-account |
| ALTER SESSION | https://docs.snowflake.com/en/sql-reference/sql/alter-session |
| SHOW PARAMETERS | https://docs.snowflake.com/en/sql-reference/sql/show-parameters |
| Introduction to Organizations | https://docs.snowflake.com/en/user-guide/organizations |
| Account Replication | https://docs.snowflake.com/en/user-guide/account-replication-intro |
| Replication & Failover Groups | https://docs.snowflake.com/en/user-guide/account-replication-config |
| Cross-Region Data Sharing | https://docs.snowflake.com/en/user-guide/secure-data-sharing-across-regions-platforms |
| SnowPro Advanced Exam Guide | https://learn.snowflake.com/en/certifications/snowpro-advanced |

---

*© Study Notes for Snowflake Advanced Certification — For educational use only.*
