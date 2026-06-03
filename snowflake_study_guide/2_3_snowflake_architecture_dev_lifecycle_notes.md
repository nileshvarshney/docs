# ❄️ Snowflake Advanced Certification Study Notes
## Topic: Architecture Solutions for Development Lifecycles and Workloads

> **Exam Domain:** SnowPro Advanced – Data Engineer / Architect
> **Last Updated:** June 2025
> **References:** Official Snowflake Documentation, Snowflake Engineering Blog, DevOps with Snowflake Guide

---

## Table of Contents

1. [Architectural Thinking for Snowflake Workloads](#1-architectural-thinking-for-snowflake-workloads)
2. [Data Lake and Environment Architecture](#2-data-lake-and-environment-architecture)
   - 2.1 Storage Directory Structure
   - 2.2 Zones and Data Warehouse Layers
   - 2.3 Supporting DevOps and DataOps Principles
   - 2.4 Environment Strategy: Production, Development, and Sandbox
3. [Data Workloads](#3-data-workloads)
   - 3.1 Data Warehouse Workloads
   - 3.2 ELT vs ETL in Snowflake
   - 3.3 Warehouse Isolation by Workload
4. [Development Lifecycle Support](#4-development-lifecycle-support)
   - 4.1 Migration
   - 4.2 Deployment and CI/CD
   - 4.3 Snowflake CLI
   - 4.4 Git Integration
   - 4.5 Rollback Process
5. [AI/ML Pipelines and Applications](#5-aiml-pipelines-and-applications)
   - 5.1 Snowpark Container Services
   - 5.2 Snowflake ML Functions
   - 5.3 Cortex LLM Functions
   - 5.4 Streamlit in Snowflake
   - 5.5 Snowflake Native App Framework
6. [Putting It All Together – Architecture Patterns](#6-putting-it-all-together--architecture-patterns)
7. [Exam Tips & Common Gotchas](#7-exam-tips--common-gotchas)

---

## 1. Architectural Thinking for Snowflake Workloads

Before designing any Snowflake architecture, a practitioner must internalize a few platform realities that shape every decision. These are not general cloud computing principles — they are specific to Snowflake's design and directly affect whether an architecture will be performant, cost-effective, and maintainable.

**Compute and storage are fully decoupled.** In a traditional data warehouse, scaling the database means scaling everything together: the storage, the indexes, the compute nodes, and the associated licenses. In Snowflake, storage and compute are completely independent layers. This means that a heavy ELT transformation job can use a large warehouse without affecting the storage costs for idle data, and a team of analysts can run interactive queries on their own warehouse while the ETL pipeline runs in parallel on a separate one. The architectural implication is that **workload isolation by virtual warehouse** is not a nice-to-have — it is the foundational pattern for every well-designed Snowflake implementation.

**There are no indexes to manage.** Traditional database architects spend significant effort designing index strategies to make queries fast. Snowflake eliminates this cognitive overhead entirely. Instead, performance optimization uses micro-partition pruning (automatic), clustering keys (for large, frequently filtered tables), materialized views, and the Search Optimization Service for highly selective lookups. Designing for Snowflake means designing around these mechanisms, not around index trees.

**The cloud service layer handles metadata.** Query planning, security enforcement, transaction management, and object metadata all live in Snowflake's cloud services layer, which is separate from both compute and storage. This layer is always running, always consistent, and requires no management. It is what makes features like Time Travel, Fail-Safe, and zero-copy cloning possible — they are metadata operations, not data operations.

**Cost is a function of compute time and storage volume.** Unlike licensing models where you pay for capacity whether you use it or not, Snowflake charges for virtual warehouse compute by the second (with a 60-second minimum per credit usage period) and for storage by the terabyte-per-month. Auto-suspend and auto-resume mean idle warehouses cost nothing. Good architecture minimizes idle compute and avoids unnecessary data duplication — but the platform's economics make techniques like zero-copy cloning for environment creation essentially free.

---

## 2. Data Lake and Environment Architecture

### 2.1 Storage Directory Structure

The concept of a "storage directory structure" in Snowflake manifests at two levels: the logical object hierarchy within Snowflake itself, and the organization of files in external cloud storage when a hybrid lake architecture is used.

**Within Snowflake, the object hierarchy is:** Organization → Account → Database → Schema → Table/View/Stage/Procedure. The database and schema levels are the primary organizing boundaries for a zoned architecture. Most enterprise Snowflake deployments use **separate databases per zone** (e.g., a RAW database, a CURATED database, an ANALYTICS database) rather than separate schemas within one database — this separation allows per-database Time Travel settings, per-database access control, and cleaner lineage between zones.

**For external stages (cloud object storage):** When Snowflake is used alongside an external data lake (S3, Azure Data Lake Storage, GCS), the directory structure in cloud storage should reflect the data's source, ingestion date, and format. A common pattern is a path structure like `s3://bucket/source-system/entity-name/year=YYYY/month=MM/day=DD/`, which supports Snowflake's ability to prune external table partitions by the date-based directory structure. This partition pruning on external tables is what makes external table queries performant — without a well-structured directory layout, external table queries scan all files regardless of the date filter in the query.

**Naming conventions are a governance mechanism.** The names of databases, schemas, tables, and stages encode environment context, data domain, and sensitivity level when designed intentionally. A naming convention like `PROD_SALES_RAW`, `DEV_SALES_RAW`, `PROD_SALES_CURATED` makes it immediately clear which environment, which domain, and which zone an object belongs to — critical for a single-account strategy where all environments coexist in the same account.

### 2.2 Zones and Data Warehouse Layers

The **multi-zone architecture** is the standard pattern for organizing data within a Snowflake-based platform. While naming conventions vary across organizations and methodologies, the underlying principle is consistent: data passes through progressively higher stages of quality, transformation, and business-readiness, with each stage serving a distinct purpose.

**Landing / Bronze / Raw Zone**

This is the first place data lands after being extracted from source systems. The core principle of this zone is **immutability and source fidelity** — data should be stored exactly as it was received, with no business transformations applied. If a source system sends a JSON payload, the raw zone stores that JSON payload unchanged. If a source sends a CSV file with inconsistent date formats, the raw zone captures those inconsistencies.

The raw zone is the system of record for "what was actually received." If business logic changes downstream — a different way to interpret a field, a revised definition of a customer — the data can always be reprocessed from the raw zone without going back to source systems. This zone is rarely queried directly by business users. In Snowflake, raw zone tables often use VARIANT columns to accept semi-structured source data without requiring a predefined schema.

**Staging / Silver / Curated Zone**

This zone applies the first layer of technical transformation: cleaning, parsing, type casting, deduplication, and light harmonization. Data in this zone is no longer raw — it has been processed and validated — but business rules have not yet been applied. An address field is normalized to a standard format; inconsistent date representations are all converted to a single timestamp type; clearly erroneous records are quarantined to an error table.

This zone serves data engineers, data quality processes, and the transformation pipelines that feed the next zone. Business users typically do not query here directly. Snowflake Streams on raw zone tables are commonly used to detect new arrivals and trigger curated zone transformations automatically.

**Analytics / Gold / Presentation Zone**

This is where business value is delivered. Data in this zone has been fully transformed, enriched with business rules, modeled dimensionally (star schema, Data Vault information mart, or other presentation formats), and optimized for the query patterns of BI tools and data consumers. Tables in this zone are the source for dashboards, reports, ML training datasets, and self-service analytics.

The gold zone's schema should use business vocabulary, not technical or source-system terminology. A column named `customer_since_date` is gold zone; a column named `acq_dt` is not. Gold zone tables should be read-optimized — which in Snowflake means considering clustering keys for large fact tables and materialized views for common aggregations.

**Operational / Sandbox Zone (optional)**

Some architectures include additional zones for specific purposes: a sandbox for data science experimentation (isolated from production to prevent accidental impact), an operational zone for near-real-time data that hasn't yet been processed through the full pipeline, or a governed data science feature store for ML features.

### 2.3 Supporting DevOps and DataOps Principles

**DevOps** applied to data platforms means treating data infrastructure — Snowflake objects, pipeline definitions, access control configurations — with the same engineering discipline applied to application code: version control, automated testing, continuous integration, and automated deployment.

**DataOps** extends DevOps with data-specific concerns: data quality testing as part of the pipeline, data lineage tracking, automated monitoring of data freshness and completeness, and governance as code. The goal of DataOps is to reduce the cycle time between a data change in a source system and reliable, trusted insight in a downstream consumer — while minimizing the risk of data quality failures along the way.

Snowflake's platform features directly enable these principles:

**Version control for Snowflake objects:** Through Git integration and declarative object management (via Snowflake CLI's DCM projects), Snowflake object definitions (table DDL, view definitions, stored procedures, policies) can be maintained in Git repositories with the same branching, review, and merge workflows used for application code.

**Automated testing:** Data quality tests can run as part of a CI pipeline before changes are promoted. dbt's test framework, Snowflake Data Metric Functions, and custom SQL validation queries can all be orchestrated to run automatically, failing the pipeline if data quality thresholds are breached.

**Infrastructure as code:** Snowflake objects can be defined declaratively using `CREATE OR ALTER` statements, Terraform with the Snowflake provider, or Snowflake CLI DCM projects. Declarative definitions are idempotent — running the same definition multiple times reaches the same state — which is essential for reliable automated deployment.

**Observability:** Snowflake's `QUERY_HISTORY`, `ACCESS_HISTORY`, and Data Metric Functions provide the raw material for observability dashboards. DataOps-mature teams build monitoring pipelines that track pipeline execution times, data freshness (time since last successful load), row count trends (unexpected drops indicate pipeline failures), and data quality scores.

### 2.4 Environment Strategy: Production, Development, and Sandbox

The choice between a **single-account multi-environment strategy** and a **multi-account strategy** is one of the most consequential architectural decisions in a Snowflake deployment. Both are valid, and the right choice depends on the organization's compliance requirements, team structure, and operational maturity.

**Single-Account Strategy**

In a single-account strategy, all environments (production, development, QA, sandbox) coexist within one Snowflake account, separated by naming conventions and RBAC. The production databases might be named `PROD_SALES`, `PROD_MARKETING`, and so on. Development databases are `DEV_SALES`, `DEV_MARKETING`. This approach has significant advantages in Snowflake specifically because **zero-copy cloning makes environment creation essentially free** — a developer can clone the entire production database to a development database in seconds, with no storage cost until they diverge.

Single-account environments also make it trivial for a developer to join development data with production reference data, which is often necessary during feature development. The shared RBAC, shared integrations (SSO, network policies, storage integrations), and shared billing simplify operations considerably.

The primary risks in a single-account strategy are accidental access to production objects (mitigated by RBAC naming conventions and managed access schemas), and the absence of physical billing isolation between environments.

**Multi-Account Strategy**

In a multi-account strategy, production and non-production environments are separate Snowflake accounts. This creates a hard physical boundary between environments — a developer's credentials in the dev account literally cannot be used in the production account. This is often required by compliance frameworks (PCI-DSS cardholder data must be isolated, HIPAA PHI in production must be separated from development environments) or by enterprise architecture standards that mandate environment isolation.

Multi-account deployments require careful CI/CD tooling to promote code changes across accounts, and cross-account data access (for a developer to work with realistic data) requires replication or data sharing — adding complexity and potentially cost. The tradeoff is a higher security posture and cleaner operational boundaries.

**Sandbox Environments**

A sandbox is a free-form experimentation environment with minimal governance overhead — a place where data engineers, data scientists, or analysts can try new approaches without following the full deployment process. In Snowflake, sandboxes are typically implemented as schemas within the development database (or as cloned databases for isolated experiments) with a dedicated virtual warehouse.

The key governance requirement for sandboxes is resource limits. An unconstrained sandbox can generate unexpected compute costs if a user accidentally runs a query without a LIMIT clause against a large table. Resource Monitors on sandbox warehouses prevent runaway spend. Sandbox environments should also have a **data expiration policy** — stale sandbox objects accumulate quickly without one, creating governance noise and potential cost.

---

## 3. Data Workloads

### 3.1 Data Warehouse Workloads

Snowflake serves multiple distinct workload types simultaneously through its multi-cluster warehouse architecture. The key insight is that **different workloads have different performance and concurrency profiles**, and mixing them on a single warehouse leads to resource contention, unpredictable performance, and tangled cost attribution.

**Batch ELT/ETL workloads** need large, bursty compute. They run on a schedule (hourly, daily), process many rows, and can tolerate some latency between when they start and when they finish. These workloads benefit from a large warehouse (L or XL) that auto-suspends between runs. Running a massive transformation on a 2XL warehouse for 10 minutes and then auto-suspending is far more efficient than running it on a Medium warehouse for 40 minutes — because Snowflake bills by the second and scales linearly.

**BI and reporting workloads** need low latency and high concurrency. Many users running many short queries simultaneously. These workloads benefit from multi-cluster warehouses that scale out (add additional compute clusters) when concurrency increases, rather than scaling up (adding more compute per cluster). A Medium multi-cluster warehouse configured to scale from 1 to 4 clusters handles 100 concurrent analyst queries more cost-effectively than a 4XL single-cluster warehouse.

**Data science and ML workloads** need large memory, are often iterative, and involve a mix of Python (Snowpark), SQL, and sometimes external compute. These workloads deserve their own warehouse both to isolate cost and to prevent a long-running ML training job from impacting interactive query response times for analysts.

**Streaming ingestion workloads** (Snowpipe, Snowpipe Streaming) use Snowflake-managed serverless compute rather than virtual warehouses. The architecture implication is that streaming ingestion cost is separated from transformation cost — they are billed independently.

### 3.2 ELT vs ETL in Snowflake

**ETL (Extract, Transform, Load)** is the traditional pattern where data is extracted from source systems, transformed to the target format in an intermediate compute environment (a dedicated ETL server or tool), and then loaded into the data warehouse in its final form. The data warehouse receives clean, transformed data.

**ELT (Extract, Load, Transform)** reverses the transform and load steps. Data is loaded into the data warehouse first, in its raw form, and transformations are executed inside the warehouse using the warehouse's own compute. The data warehouse does double duty as both the transformation engine and the storage layer.

**Why Snowflake strongly favors ELT:**

Snowflake's elastic, scalable compute makes in-warehouse transformation efficient and cost-effective. Rather than paying for a dedicated ETL server running 24/7, the transformation runs on a Snowflake virtual warehouse that auto-suspends when idle. Transformations written in SQL (or Snowpark Python/Java/Scala) run with the full power of Snowflake's distributed engine and benefit from micro-partition pruning, columnar storage, and result caching — none of which are available to an external ETL tool processing data before it enters Snowflake.

ELT also aligns naturally with the multi-zone architecture: the raw zone holds source data exactly as received (the "Load" phase), and all transformations occur as data moves from raw to curated to analytics zones (the "Transform" phase). This separation means transformations can be re-run from scratch if business rules change, without ever going back to source systems.

**When ETL still makes sense in a Snowflake architecture:**

External ETL remains appropriate when source systems require complex API-based extraction that must happen outside the database (e.g., REST API pagination, OAuth token management), when data must be filtered or sanitized before entering Snowflake for compliance reasons (e.g., PII must be tokenized before Snowflake sees it), or when the organization has significant existing investment in an ETL platform that manages hundreds of source connectors. In these cases, the ETL tool handles extraction and loading, while Snowflake handles all transformations within the platform.

**Dynamic Tables as ELT automation:**

Snowflake's Dynamic Tables are a significant advancement in ELT automation. A Dynamic Table is defined by a SELECT query, and Snowflake automatically keeps the Dynamic Table's contents up to date by re-evaluating the query incrementally whenever its upstream tables change. This creates a declarative, self-maintaining ELT pipeline where the engineer defines the desired transformation state and Snowflake manages the refresh orchestration — without requiring Streams, Tasks, or external orchestration tools for many common patterns.

### 3.3 Warehouse Isolation by Workload

The principle of workload isolation through separate virtual warehouses is the single most impactful architectural decision for Snowflake performance and cost management. A minimum production warehouse topology for a mature organization includes:

**Ingestion/Loading Warehouse:** Used by Snowpipe-triggered tasks, bulk COPY INTO operations, and streaming ingestion pipelines. Sized for throughput. Auto-suspends quickly because loading bursts are typically short.

**Transformation Warehouse:** Used for ELT processing — dbt runs, stored procedure executions, Dynamic Table refreshes, and complex SQL transformations. May be a larger size (L or XL) to handle complex multi-table joins and aggregations efficiently. Auto-suspends between scheduled runs.

**Analytics/BI Warehouse:** Used by BI tools (Tableau, Power BI, Looker) and analysts running interactive queries. Benefits from multi-cluster auto-scaling to handle concurrency spikes. Sized conservatively (S or M) because most BI queries hit materialized views or the result cache.

**Data Science Warehouse:** Used by Snowpark notebooks, ML training jobs, and exploratory analysis. Often sized dynamically — started large when needed, suspended otherwise.

**Development Warehouse:** Used by engineers developing new pipelines, testing transformations, and validating changes. Smaller size, strict Resource Monitor to prevent runaway costs.

**Administrative Warehouse:** Used by ACCOUNTADMIN operations, governance tasks, and account-level queries. Small and separate to ensure administrative operations are never delayed by production query contention.

---

## 4. Development Lifecycle Support

### 4.1 Migration

Migration encompasses two distinct scenarios in Snowflake: migrating data from an existing warehouse to Snowflake, and migrating data between environments within Snowflake.

**Migrating from legacy systems to Snowflake** involves translating DDL (table definitions, view definitions, stored procedures), migrating historical data, and re-implementing pipeline logic. Key architectural decisions during migration include:

The **lift-and-shift vs. modernize** decision: Whether to translate existing SQL and procedures as directly as possible (lift-and-shift, minimizing risk but missing the opportunity to improve) or to redesign pipelines using Snowflake's native capabilities (ELT, Dynamic Tables, Streams) while migrating (modernize, higher risk but better long-term outcome). A common approach is lift-and-shift first to establish a working baseline, then modernize incrementally.

**Blue-green migration:** For large, complex migrations where downtime must be minimized, a blue-green approach uses Snowflake's zero-copy cloning to create a parallel green environment that runs alongside the live blue environment. The migration team validates the green environment thoroughly, including data reconciliation between blue and green, before cutting over. If issues arise after cutover, the rollback is as simple as redirecting applications back to the blue environment — no data needs to be restored.

**Schema change management tools** such as Flyway, Liquibase, schemachange, and Snowflake's own DCM (Declarative Change Management) projects manage the versioning of DDL changes and ensure changes are applied consistently and in order across environments. These tools maintain a migration history table that records which scripts have been applied to which environment, preventing the same migration from being applied twice and ensuring idempotent deployments.

**Historical data migration** typically uses the COPY INTO command for bulk loads from staged files, or Snowflake's partner ecosystem tools (Fivetran, Informatica, dbt) for replication from the source system. The raw zone of the new architecture receives historical data first, and transformation pipelines then process it through curated and analytics zones.

### 4.2 Deployment and CI/CD

**CI/CD (Continuous Integration and Continuous Delivery)** applied to Snowflake brings software engineering discipline to data infrastructure. The core workflow is: code changes are committed to a Git branch, automated tests run against those changes in a development environment, and if tests pass, the changes are automatically (or semi-automatically) deployed to production.

**Continuous Integration (CI)** for Snowflake typically involves:
- A developer creates a feature branch in Git and makes changes to SQL, Python, or schema definition files
- A pull request is opened, triggering the CI pipeline
- The CI pipeline clones the current production database to a PR-specific development environment (zero-copy clone makes this instant and free)
- The proposed changes are applied to the cloned environment using the deployment tool
- Data quality tests, unit tests for stored procedures, and integration tests run against the cloned environment
- The pipeline succeeds or fails based on test results; the PR can only be merged if tests pass
- After the PR is closed or merged, the cloned environment is dropped

**Continuous Delivery (CD)** automates the application of merged changes to higher environments:
- A merge to the main/trunk branch triggers the CD pipeline
- The CD pipeline applies changes to the test/QA environment first
- After validation (automated or human), changes are promoted to production
- The deployment applies only the delta — the changes that are new since the last deployment

**Snowflake's first-party CI/CD integrations** (GitHub Actions, GitLab CI, Azure DevOps) install Snowflake CLI on the CI runner and configure authentication. **Workload Identity Federation (OIDC)** is the recommended authentication method for CI/CD pipelines — the CI runner obtains a short-lived identity token from the CI platform that Snowflake validates directly, without any stored credentials. This eliminates the service account private key rotation problem that traditionally plagued pipeline authentication.

**Declarative vs. imperative deployment:**

*Imperative* deployment (schemachange, Flyway, Liquibase) works by applying numbered migration scripts in order. Each migration script describes a change operation (`ALTER TABLE ADD COLUMN`, `CREATE VIEW`, etc.). The tool tracks which scripts have been applied and runs only new ones. Rollback requires explicit rollback scripts.

*Declarative* deployment (`CREATE OR ALTER`, Snowflake CLI DCM projects, Terraform) works by defining the desired state of objects. The deployment tool computes the difference between the current state and the desired state, and applies whatever operations are needed to close that gap. Declarative definitions are idempotent — running the same definition twice has no effect if the object is already in the desired state. This makes declarative deployment safer and easier to reason about.

### 4.3 Snowflake CLI

**Snowflake CLI** (`snow`) is Snowflake's official open-source command-line tool for managing Snowflake resources, executing deployments, and integrating with CI/CD pipelines. It is the successor to older tools like SnowSQL and represents Snowflake's first-party answer to infrastructure-as-code for the platform.

**Key capabilities of Snowflake CLI:**

**DCM (Declarative Change Management) projects:** The primary feature for CI/CD deployments. A DCM project is a directory of SQL and Python files (with a `manifest.yml` configuration) that defines the desired state of Snowflake objects. `snow project deploy` applies the project to the target Snowflake account, computing and executing only the necessary changes. Multiple deployment targets (dev, test, prod) can be defined in the manifest with different account bindings and parameterization via Jinja templates.

**Snowpark function and procedure deployment:** `snow snowpark deploy` packages and deploys Python, Java, or Scala Snowpark functions and procedures, handling dependency management and artifact upload to Snowflake stages.

**Native App deployment:** `snow app run` and `snow app deploy` manage the full lifecycle of Snowflake Native App development, including local testing and Marketplace publication.

**Object lifecycle management:** Commands for creating, listing, describing, and dropping most Snowflake object types — useful for scripting routine administrative tasks.

**Connection management:** Supports multiple named connections (profiles), making it easy to switch between accounts, roles, and warehouses from the command line.

**Authentication support:** Supports key-pair authentication, OAuth token flows, SSO/browser-based flows, and workload identity federation. The `config.toml` file stores connection definitions; secrets are injected via environment variables, not stored in the config file.

### 4.4 Git Integration

Snowflake's **Git integration** allows a Snowflake account to connect directly to a Git repository (GitHub, GitLab, Azure DevOps, Bitbucket), treating the repository as a stage-like object from which code can be read and executed. This enables code to be sourced directly from a repository without requiring a separate staging step.

**How it works:** A Git repository is registered in Snowflake using a `CREATE GIT REPOSITORY` command, which references an API integration (for HTTPS access) and optionally a secret (for private repository authentication). After registration, Snowflake creates an internal stage that mirrors the repository's file structure. Running `ALTER GIT REPOSITORY FETCH` refreshes the local mirror with the latest commits from the remote repository.

**What Git integration enables:**

*Code sourcing for stored procedures and UDFs:* Python or Java code for Snowpark procedures can reference files directly in the Git repository stage rather than being bundled inline in the procedure definition. When the code is updated and pushed to Git, running `FETCH` on the repository and then re-executing the `CREATE OR REPLACE` procedure definition picks up the new code. This eliminates the need to separately manage code files in an internal stage.

*Direct file execution from branches:* SQL scripts in a repository can be executed against Snowflake by referencing the file's path in the Git repository stage. This enables branch-based deployment patterns where the CI pipeline fetches the current branch and applies its scripts directly.

*Notebook integration:* Snowflake Notebooks can be synced to Git repositories, allowing notebook code to be version-controlled alongside other data pipeline code.

**Important limitation:** The Git stage in Snowflake is **read-only** — Snowflake can read from the repository but cannot write back to it. The Git integration does not support pushing code changes from Snowflake to the repository. It is purely a consumption mechanism.

**When Git integration complements CI/CD tooling:** The Git integration is most valuable for tightly coupling code references to specific Git commits, enabling reproducible deployments where the exact code at a specific commit hash is deployed. It reduces the number of artifact management steps in a pipeline. However, it does not replace a CI/CD pipeline — it is one component within a broader DevOps workflow.

### 4.5 Rollback Process

Rollback in Snowflake is more nuanced than in traditional databases because multiple mechanisms provide different rollback capabilities with different scopes and time limits.

**Time Travel – Data Rollback**

Snowflake's Time Travel allows any table to be queried as of a specific past point in time (using `AT` or `BEFORE` clauses with a timestamp, offset, or statement ID). For data rollback, the most powerful operation is cloning a table at a historical point and replacing the current table with the historical clone — or swapping them using `ALTER TABLE SWAP WITH`.

The crucial architectural decision is **Time Travel retention period**. Standard Edition provides a maximum of 1 day. Enterprise Edition allows up to 90 days per table. The retention period must be set at table creation or alteration, and it determines the rollback window. For critical tables where business errors (incorrect transformations, accidental deletes) might not be detected immediately, a longer retention period on the affected tables is essential for operational safety.

**DDL Rollback via Version Control (Code Rollback)**

When Snowflake objects are defined and deployed from a Git repository, rolling back a DDL change is a matter of reverting the commit in Git and re-running the deployment pipeline. If using declarative definitions (`CREATE OR ALTER`), redeploying the previous definition returns the object to its prior state.

For additive changes (adding a column), reverting is straightforward — the `CREATE OR ALTER` without the new column drops it. For destructive changes (dropping a column), reversion is more complex because the column's data is lost. This is why schema migrations should always have explicit rollback procedures written before the forward migration is executed — particularly for changes that cannot be trivially reversed.

**Zero-Copy Clone as Pre-deployment Snapshot**

A best practice before any significant schema migration or data transformation is to **zero-copy clone the affected database or tables before making changes**. The clone serves as an instant point-in-time snapshot that can be used to restore if the change fails. Because the clone is zero-copy, creating it is instantaneous and costs nothing until the original diverges from the clone. If the migration succeeds, the clone is dropped. If it fails, the original is dropped and the clone is renamed — an operation that takes milliseconds and has zero data movement cost.

**Application Code Rollback**

For Snowpark procedures, UDFs, and tasks that are version-controlled in Git, rollback is a Git revert and re-deploy. Snowflake itself does not maintain multiple versions of stored procedure code — the `CREATE OR REPLACE PROCEDURE` pattern means the current definition is the only version. Version history exists only in the Git repository, making Git commit history the rollback mechanism for code changes.

**Task-based pipeline rollback**

For ELT pipelines implemented with Snowflake Tasks, a pipeline can be halted by suspending the root task. If a pipeline has populated tables with incorrect data, the rollback strategy combines Time Travel (to restore affected tables) with task suspension (to stop further incorrect data from flowing downstream). After correcting the transformation logic (re-deploying from Git), the tasks are resumed to re-process from the last clean state.

---

## 5. AI/ML Pipelines and Applications

### 5.1 Snowpark Container Services

**Snowpark Container Services (SPCS)** is Snowflake's managed container execution environment — an OCI (Open Container Initiative) compatible platform built directly into the Snowflake platform. It allows teams to run arbitrary containerized workloads inside Snowflake's security and governance boundary, eliminating the need to extract data from Snowflake into an external compute environment for workloads that cannot run in a standard SQL or Python context.

**The fundamental problem SPCS solves:** Not every computation can be expressed in SQL or even standard Python. Complex ML model serving (TensorFlow, PyTorch), GPU-accelerated workloads, custom API services, multi-container application stacks (a Flask backend with a Vue.js frontend and nginx routing) — these require the flexibility of containers. Traditionally, running these workloads meant extracting data from Snowflake to an external Kubernetes cluster, which broke data governance, added latency, and created security complexity. SPCS brings the containerized compute to the data, keeping everything in Snowflake's governance envelope.

**Key architectural components:**

*Service:* A long-running containerized workload (analogous to a Kubernetes Deployment). Services can be web servers, ML model endpoints, or API backends. They are accessible within Snowflake via service endpoints and can be exposed publicly or privately.

*Job:* A short-lived containerized workload that runs to completion and exits (analogous to a Kubernetes Job). Batch ML training runs, data processing jobs, and report generation tasks are implemented as SPCS Jobs.

*Compute Pool:* The underlying virtual machine infrastructure that backs SPCS workloads. Compute pools can be configured with different instance types — including GPU-enabled instances for ML workloads. Compute pools are separate from virtual warehouses; SPCS costs are not warehouse credits.

*Image Registry:* A Snowflake-managed OCI-compatible container registry where container images are stored. Images are stored in Snowflake-managed cloud storage and are subject to the same governance controls as other Snowflake objects.

**Security model:** Containers in SPCS run within Snowflake's VPC/VNet. Access to Snowflake data from within a container uses the Snowflake connector — the container authenticates to Snowflake using the application owner role. External network access from containers is controlled by network rules and external access integrations, the same mechanism used for UDFs. This means SPCS workloads cannot exfiltrate data to unauthorized external endpoints.

**Use cases:** Long-running ML model serving endpoints, custom API services for Snowflake Native Apps, GPU-accelerated inference workloads, complex data processing pipelines that require external libraries not available in the Snowflake Python sandbox, and web applications with multiple backend components.

### 5.2 Snowflake ML Functions

**Snowflake ML Functions** are SQL-callable machine learning algorithms built natively into the Snowflake platform, executing inside the Snowflake engine without requiring any external infrastructure, Python environment management, or MLOps tooling. They enable end-to-end ML workflows — from feature engineering to model training to inference — using familiar SQL interfaces.

**The design philosophy:** Most organizations' most common ML needs are not exotic deep learning problems. They are classification (will this customer churn?), regression (what will this customer spend?), anomaly detection (is this transaction fraudulent?), forecasting (how much inventory should we order?), and recommendation. Snowflake ML Functions provide production-quality implementations of algorithms for these problems, trained and served entirely within Snowflake.

**Key ML function categories:**

*Snowflake ML Modeling (the Model Registry):* Snowpark ML's Python library provides scikit-learn-compatible estimators and transformers (linear regression, random forest, XGBoost, and others) that train and run models entirely within a Snowflake virtual warehouse using Snowpark. Trained models are stored in the **Snowflake Model Registry** — a governed repository of versioned ML models. Models in the registry can be called from SQL as UDFs, making model inference as simple as a SQL function call.

*Automated ML Functions:* Snowflake provides higher-level AutoML-style functions where the algorithm choice and hyperparameter tuning are handled automatically. `FORECAST` for time-series prediction, `ANOMALY_DETECTION` for identifying unusual patterns, `CLASSIFICATION` for binary and multi-class classification problems, `REGRESSION` for numerical prediction, and `CONTRIBUTION_EXPLORER` / `TOP_INSIGHTS` for dimension-level analysis of metric changes. These are invoked as Python procedures using the `snowflake.ml` module.

*Snowflake ML Jobs:* Generally available as of August 2025, ML Jobs allow longer-running training workloads to be executed as managed, serverless ML compute jobs outside the warehouse execution model. This is particularly relevant for training larger models that would be cost-inefficient on a virtual warehouse.

**Governance advantage:** Because models are trained on data that never leaves Snowflake, and inferences are made inside Snowflake, the full lineage of the ML pipeline — from training data to model to predictions — is traceable within Snowflake's existing governance framework. There is no external model serving endpoint to govern separately.

### 5.3 Cortex LLM Functions

**Snowflake Cortex LLM Functions** are SQL and Python callable functions that provide access to large language models hosted entirely within Snowflake. There are no API keys to manage, no external LLM providers to authenticate with, no data leaving Snowflake to an external API endpoint. The LLMs are deployed and managed by Snowflake, and model invocations run within the same security boundary as every other Snowflake query.

**The data governance implication is significant:** The biggest barrier to enterprise adoption of LLM-based workflows has been the requirement to send potentially sensitive enterprise data (customer emails, contracts, medical records) to external AI APIs. Cortex LLM Functions eliminate this barrier — all text processing happens inside Snowflake, subject to the same encryption, access control, and audit logging as any other query.

**Core Cortex LLM functions:**

`SNOWFLAKE.CORTEX.COMPLETE(model, prompt)` is the general-purpose text generation function. It accepts a model name and a prompt (or a conversation history for multi-turn interactions) and returns the model's response. Available models include Llama, Mistral, Arctic, and others, with Snowflake adding new models as they become available. Snowflake Arctic is Snowflake's own open-source model, optimized for enterprise analytics tasks.

`SNOWFLAKE.CORTEX.SUMMARIZE(text)` generates a concise summary of input text without requiring prompt engineering. Useful for summarizing customer feedback, support tickets, meeting transcriptions, or document content at scale.

`SNOWFLAKE.CORTEX.SENTIMENT(text)` returns a sentiment score (-1 to 1) for input text. Applied across a table of customer reviews, it transforms unstructured feedback into a structured numerical signal for analytics.

`SNOWFLAKE.CORTEX.TRANSLATE(text, source_lang, target_lang)` performs language translation without an external translation API. Useful for globalizing data — translating multilingual customer feedback into a single analysis language.

`SNOWFLAKE.CORTEX.EXTRACT_ANSWER(question, context)` performs extractive question-answering — finding the answer to a question within a provided text passage. Useful for extracting specific information from unstructured documents.

**Cortex Search and Cortex Agents:** Cortex Search provides hybrid vector and keyword search over Snowflake data, enabling semantic search use cases. Cortex Agents (generally available mid-2025) orchestrate multi-step workflows combining LLMs with data retrieval, enabling conversational interfaces over enterprise data. These represent Snowflake's move from individual LLM functions to complete AI application infrastructure.

**The `CORTEX_USER` database role:** To call Cortex functions from code (including from Snowflake Native Apps), the executing role must have the `CORTEX_USER` database role in the `SNOWFLAKE` database granted to it. This is the privilege gate that controls which users and applications can access LLM functionality.

### 5.4 Streamlit in Snowflake

**Streamlit in Snowflake (SiS)** is the hosted Streamlit application platform embedded within the Snowflake platform. Streamlit is a Python library for building interactive web applications without frontend engineering skills — a data engineer or data scientist writes Python code that defines UI components (sliders, dropdowns, charts, data tables) and the framework renders them as a web application.

**Why hosting Streamlit inside Snowflake matters:**

Standalone Streamlit applications connect to Snowflake as an external client — they must manage Snowflake credentials, network access, and the security of an external web server. Streamlit in Snowflake eliminates all of this. The Streamlit app runs inside Snowflake's execution environment, with direct access to Snowflake data through Snowpark without any external connection management. The app inherits the permissions of the role it runs under, and Snowflake handles the web hosting, security, authentication (the user logs in with their Snowflake credentials), and scaling.

**The governance implication:** Because SiS runs inside Snowflake, all data access from the app is governed by Snowflake's RBAC, masking policies, and row access policies — just like any other query. A business user accessing a Streamlit dashboard sees only the data their role permits, with masking applied. There is no need for a separate application security layer.

**Common use cases for Streamlit in Snowflake:**

*Internal dashboards and data products:* Business-facing analytics views that require interactivity beyond static BI reports — filtering by dynamic parameters, drill-down exploration, side-by-side comparisons with adjustable inputs.

*ML model interfaces:* Interactive front-ends for ML models where a user can input parameters and receive predictions. For example, a pricing simulation tool that calls a regression model and displays results in real time.

*Data quality monitoring dashboards:* Real-time views of Data Metric Function results, pipeline health metrics, and data freshness indicators for data operations teams.

*LLM-powered data tools:* Applications that combine Cortex LLM functions with Snowflake data — a document Q&A tool that searches the Cortex Search index, a sentiment trend dashboard that applies SENTIMENT to incoming reviews, a Cortex Agents chatbot interface for business users to ask natural language questions about their data.

**Integration with the broader platform:** SiS integrates with Snowpark (for Python-based data access and transformation within the app), UDFs and stored procedures, the Snowflake Native App Framework (Streamlit can be bundled into a Native App as its UI), and Cortex functions (callable directly from app code after granting `CORTEX_USER`).

**Limitations:** Some Streamlit features available in the open-source library are not supported in SiS (check the documentation for the current unsupported list). SiS is not available by default in Virtual Private Snowflake (VPS) accounts — it must be enabled through Snowflake Support. Streamlit in Snowflake is not suitable for applications requiring heavy custom frontend frameworks or complex multi-page routing beyond Streamlit's native multipage support.

### 5.5 Snowflake Native App Framework

The **Snowflake Native App Framework** is the mechanism for packaging, distributing, and consuming complete data applications — code, data, and UI — within the Snowflake platform. It allows software providers to build applications that run inside the consumer's Snowflake account rather than as external services that call Snowflake from outside.

**The architecture shift:** Traditional SaaS data applications extract data from the customer's data warehouse, process it in the vendor's infrastructure, and return results. This model has significant data governance problems — customer data leaves the customer's security boundary, the vendor must be trusted with sensitive data, and results must be loaded back into the customer's system. The Native App Framework inverts this model: the vendor's application code is installed inside the customer's Snowflake account, where it runs directly against the customer's data without that data ever leaving the customer's account.

**Key components of the Native App Framework:**

*Application Package:* The provider's development artifact — a Snowflake object containing the application's code (stored procedures, UDFs, Streamlit apps, Snowpark Container Services), data (if any is bundled), and configuration. The provider develops and tests the application package in their own account before publishing.

*Application:* The consumer-side instantiation of an Application Package. When a consumer installs a Native App, Snowflake creates an Application object in the consumer's account. The application runs under a controlled set of permissions — the consumer controls which privileges the app receives.

*Setup Script:* A SQL script executed during app installation that creates the application's objects in the consumer's account. This script runs in the consumer's account context, so it can only do what the consumer grants it permission to do.

*Application Role:* A special role type used within Native Apps to control what privileges the app requests from the consumer. The consumer reviews these privilege requests during installation — feature policies (available from the June 2025 Summit release) allow consumers to restrict certain types of objects an app can create, such as preventing the app from creating new warehouses.

**The trust model:** The consumer never sees the provider's underlying source code for stored procedures (they are packaged as encrypted artifacts) unless the provider explicitly marks them as viewable. The consumer can see what privileges the app requests and what objects it creates, but not the implementation details the provider wants to protect as intellectual property.

**Snowflake ML in Native Apps (GA July 2025):** Providers can include ML training algorithms in a Native App, allowing the model to be trained on the consumer's data within their account. This is powerful for use cases where a generic pre-trained model would be less accurate than a model trained on the specific consumer's data — and where the consumer cannot share their training data with the provider.

**Distribution via Marketplace:** Native Apps are published and distributed through the Snowflake Marketplace using the same listing mechanism as data products. A consumer discovers the app on the Marketplace, requests or purchases access, and installs it in their account. The Native App framework with Snowpark Container Services enables fully featured web applications — not just SQL-based logic — to be distributed this way.

**Limitations:** Native Apps do not support failover for business continuity (application packages cannot be added to replication or failover groups). Apps with Snowpark Container Services containers are only supported in specific AWS, Azure, and GCP commercial regions — not all regions support container-based apps. VPS accounts require explicit enablement from Snowflake Support to use Native Apps or Streamlit.

---

## 6. Putting It All Together – Architecture Patterns

Understanding individual components is necessary but not sufficient. The exam frequently presents scenarios requiring the correct combination of features. The following patterns illustrate how the components covered in this topic fit together.

**Pattern 1: Multi-Environment CI/CD with Zero-Copy Cloning**

A team maintains three environments: DEV, QA, and PROD. DEV is a cloned database from PROD, created fresh at the start of each sprint. Developers branch from the main Git repository, make changes, and test in DEV. When a branch is ready for a PR, the CI pipeline clones QA from PROD, applies the branch's changes to the clone, runs automated tests, and reports results. Passing PRs are merged to main, triggering the CD pipeline which applies changes directly to PROD. If a PROD deployment fails, Time Travel on the affected tables allows data rollback, and reverting the Git commit and re-deploying restores the object definitions.

**Pattern 2: LLM-Powered Analytics Application**

An organization wants to let business users ask natural language questions about their sales data. The architecture: customer interaction data (emails, call transcripts) is loaded through Snowpipe into the raw zone. An ELT pipeline (driven by Tasks and Dynamic Tables) cleans and stages the data through curated to analytics zones, applying `SNOWFLAKE.CORTEX.SENTIMENT` to feedback columns as part of the transformation. `SNOWFLAKE.CORTEX.COMPLETE` powers a conversational interface in a Streamlit in Snowflake app that lets business users ask questions. Cortex Search provides semantic retrieval over the document corpus. The entire stack — ingestion, transformation, ML enrichment, and UI — runs inside Snowflake.

**Pattern 3: Native App for SaaS Data Product**

A fraud detection SaaS company wants to offer a model to financial services customers who cannot share transaction data externally. The company develops a Native App containing their fraud detection algorithm as a Snowpark procedure (encrypted, intellectual property protected). The financial institution installs the app, grants it SELECT on their transaction tables, and the app trains a customer-specific model within their account. The model is stored in the consumer's Model Registry. Inference runs on live transaction data in the consumer's account. No transaction data ever reaches the provider's infrastructure.

**Pattern 4: Data Lake with External Tables and Iceberg**

A data engineering team manages a Hadoop-era data lake in S3 with Parquet files partitioned by date. They want to modernize analytics without a full migration. They create Snowflake External Tables pointing to the S3 paths, defining partition specifications that map to the directory structure. For new data, they use Snowflake-managed Apache Iceberg tables in S3, which Snowflake governs natively through the Horizon Catalog. Transformation pipelines use Snowflake compute to transform external and Iceberg table data into standard Snowflake analytics tables in the gold zone. The result is a unified query interface over both legacy lake data and new managed data.

---

## 7. Exam Tips & Common Gotchas

### ⚡ High-Yield Conceptual Points

**Zero-copy cloning is metadata-only.** A cloned database, schema, or table shares the underlying micro-partitions with the original. No data is copied — the clone references the same storage. Storage cost only accrues when the clone diverges from the original (new writes to either the original or the clone). This makes environment creation essentially free.

**Cloning does not copy permissions, but does copy constraints.** When a database is cloned, the cloned objects do not inherit the grants of the original objects — privileges must be explicitly granted on the clone. However, constraints (PRIMARY KEY, FOREIGN KEY, etc.) are copied to the clone.

**Dynamic Tables are declarative ELT.** A Dynamic Table is defined by a SELECT query and automatically refreshes when upstream tables change. They replace the Streams + Tasks pattern for many incremental refresh use cases. The `TARGET_LAG` parameter specifies the maximum acceptable staleness.

**ELT is strongly preferred over ETL in Snowflake.** Snowflake's elastic compute makes in-warehouse transformation efficient. The raw zone holds source data exactly as received; all transformation happens inside Snowflake. This is the architectural foundation of the multi-zone approach.

**Separate virtual warehouses per workload are a foundational pattern.** Mixing ELT, BI, and data science on a single warehouse creates resource contention and makes cost attribution impossible. At minimum: an ingestion warehouse, a transformation warehouse, and an analytics warehouse.

**Snowflake CLI is the first-party tool for CI/CD integration.** It supports DCM projects for declarative deployments, has first-party integrations with GitHub Actions, GitLab CI, and Azure DevOps, and supports OIDC (workload identity federation) as the recommended credential-free authentication method for CI pipelines.

**Git integration in Snowflake is read-only.** Snowflake can fetch from a repository and read files from it; it cannot push changes back. Code is authored in the developer's environment and pushed to Git through normal Git workflows.

**Time Travel enables data rollback, not schema rollback.** If you drop a column and realize it was wrong, Time Travel restores the data (you can clone the table as it was before the column was dropped) but doesn't automatically reverse the schema change. Schema rollback requires version-controlled DDL re-deployment.

**Fail-Safe is NOT accessible by customers.** Fail-Safe (7 days after Time Travel expires) is a Snowflake internal recovery mechanism used only by Snowflake Support. Customers cannot query Fail-Safe data directly. Only Time Travel is customer-accessible for recovery.

**Cortex LLM functions require CORTEX_USER database role.** Roles calling Cortex functions need this role granted. In Native Apps, providers must document that consumers must grant this role to the app.

**SPCS containers run as the application owner role, not the calling user's role.** When a container connects to Snowflake, it runs as the application owner, not as the user who triggered the container invocation. This is a critical security and privilege-scoping consideration.

**The Native App Framework inverts the traditional SaaS trust model.** Code runs inside the consumer's account; data never leaves the consumer's boundary. The provider cannot access consumer data. This is fundamentally different from traditional SaaS where customer data flows to the vendor's infrastructure.

**Feature policies in Native Apps (June 2025) allow consumers to restrict what apps can create.** A consumer can, for example, create a feature policy that prevents a Native App from creating new warehouses. This gives consumers governance control over app behavior.

**Workload Identity Federation (OIDC) is the recommended authentication for CI/CD.** Short-lived tokens from the CI platform are validated directly by Snowflake, eliminating the need for stored service account credentials. This is more secure than key-pair or password-based service accounts.

**Multi-cluster auto-scaling scales out (adds clusters), not up (doesn't increase cluster size).** When a multi-cluster warehouse is configured with MAX_CLUSTERS > 1, Snowflake adds additional clusters of the same size when concurrency demand increases. This is different from resizing a single warehouse to a larger size.

### 🚫 Classic Exam Traps

| What the Exam Tests | The Correct Answer |
|---|---|
| "Does zero-copy cloning copy data to a new storage location?" | No — it creates a metadata pointer to the same micro-partitions. Data cost only accrues on divergence. |
| "Does a cloned database inherit the grants of the original?" | No — privileges must be re-granted on the clone. Constraints ARE copied; grants are NOT. |
| "What is the maximum Time Travel retention on Standard Edition?" | 1 day. Enterprise Edition allows up to 90 days. |
| "Can customers directly access Fail-Safe data?" | No — Fail-Safe is internal to Snowflake Support. Customers access historical data only through Time Travel. |
| "Is Git integration in Snowflake bidirectional?" | No — it is read-only. Snowflake reads from the repository; it cannot push back to it. |
| "What authentication method is recommended for CI/CD pipelines with Snowflake CLI?" | Workload Identity Federation (OIDC) — short-lived tokens, no stored credentials. |
| "What happens to a Virtual Warehouse when it auto-suspends?" | It stops consuming credits. Auto-resume re-starts it on the next query. There is no data loss. |
| "Does Snowflake enforce primary key constraints on standard tables?" | No — primary key constraints are informational/optimizer hints on standard tables. Only NOT NULL is enforced on standard tables. Hybrid Tables enforce PK constraints. |
| "Can a Snowflake Native App access consumer data directly without consumer authorization?" | No — the consumer controls which privileges the app receives during installation. The app can only access what it is explicitly granted. |
| "Where does a Native App's code execute?" | Inside the consumer's Snowflake account. The vendor's code runs in the consumer's environment, not on the vendor's infrastructure. |
| "Does Streamlit in Snowflake require a separate web server?" | No — Snowflake hosts the Streamlit app. Users authenticate with their Snowflake credentials. |
| "What role controls access to Cortex LLM functions?" | The CORTEX_USER database role in the SNOWFLAKE database must be granted to any role that needs to call Cortex functions. |
| "Can a multi-cluster warehouse scale to a larger per-cluster size automatically?" | No — auto-scaling adds more clusters of the same size (scale out). Changing the per-cluster size is a manual resize operation. |
| "What is the difference between a SPCS Service and a Job?" | A Service is long-running (persistent endpoint). A Job runs to completion and exits. |
| "Does ELT or ETL better align with Snowflake's architecture?" | ELT — data is loaded raw first, transformed inside Snowflake using its elastic compute. ETL (transform before loading) is generally an anti-pattern for new Snowflake implementations. |
| "What does CREATE OR ALTER do vs CREATE OR REPLACE?" | CREATE OR ALTER modifies an existing object to match the definition without replacing it (preserving grants, data). CREATE OR REPLACE drops and recreates the object (losing grants and table data). |
| "Can a Snowflake Native App be added to a replication/failover group?" | No — Native Apps do not support failover for business continuity. Application packages cannot be in replication or failover groups. |

---

## 📚 Reference Links

| Resource | URL |
|---|---|
| DevOps with Snowflake | https://docs.snowflake.com/en/developer-guide/builders/devops |
| Snowflake CLI Documentation | https://docs.snowflake.com/en/developer-guide/snowflake-cli/index |
| Integrating CI/CD with Snowflake CLI | https://docs.snowflake.com/en/developer-guide/snowflake-cli/cicd/integrate-ci-cd |
| Git Integration | https://docs.snowflake.com/en/developer-guide/git/git-overview |
| Zero-Copy Cloning | https://docs.snowflake.com/en/user-guide/object-clone |
| Time Travel | https://docs.snowflake.com/en/user-guide/data-time-travel |
| Dynamic Tables | https://docs.snowflake.com/en/user-guide/dynamic-tables-about |
| Snowpark Container Services | https://docs.snowflake.com/en/developer-guide/snowpark-container-services/overview |
| Snowflake ML Functions | https://docs.snowflake.com/en/developer-guide/snowflake-ml/overview |
| Cortex LLM Functions | https://docs.snowflake.com/en/user-guide/snowflake-cortex/llm-functions |
| Cortex Agents | https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agent |
| Streamlit in Snowflake | https://docs.snowflake.com/en/developer-guide/streamlit/about-streamlit |
| Snowflake Native App Framework | https://docs.snowflake.com/en/developer-guide/native-apps/native-apps-about |
| Native Apps Limitations | https://docs.snowflake.com/en/developer-guide/native-apps/limitations |
| Getting Started with Snowflake DevOps (Quickstart) | https://www.snowflake.com/en/developers/guides/getting-started-with-snowflake-devops/ |
| Snowflake ML in Native Apps | https://docs.snowflake.com/en/developer-guide/native-apps/snowflake-ml-na-about |

---

*© Study Notes for Snowflake Advanced Certification — For educational use only.*
