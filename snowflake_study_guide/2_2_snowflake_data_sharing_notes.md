# ❄️ Snowflake Advanced Certification Study Notes
## Topic: Data Sharing Solutions

> **Exam Domain:** SnowPro Advanced – Data Engineer / Architect
> **Last Updated:** June 2025
> **References:** Official Snowflake Documentation, Snowflake Engineering Blog, Snowflake Horizon

---

## Table of Contents

1. [The Snowflake Data Sharing Philosophy](#1-the-snowflake-data-sharing-philosophy)
2. [The Share Object – The Foundation of All Sharing](#2-the-share-object--the-foundation-of-all-sharing)
3. [Use Cases by Sharing Scope](#3-use-cases-by-sharing-scope)
   - 3.1 Sharing Within the Same Snowflake Account
   - 3.2 Sharing Within the Same Cloud Region (Different Accounts)
   - 3.3 Sharing Across Cloud Regions
   - 3.4 Sharing Between Different Snowflake Accounts (Cross-Organization)
   - 3.5 Sharing to a Non-Snowflake Customer
   - 3.6 Sharing Across Cloud Providers
   - 3.7 Sharing Using Snowflake Data Clean Rooms
4. [Snowflake Marketplace](#4-snowflake-marketplace)
5. [Data Exchange](#5-data-exchange)
6. [Data Sharing Methods – Deep Dive](#6-data-sharing-methods--deep-dive)
   - 6.1 Direct Share
   - 6.2 Listings (Private and Public)
   - 6.3 Reader Accounts
   - 6.4 Cross-Cloud Auto-Fulfillment
7. [Configuring Shares, Account Parameters, and Privileges](#7-configuring-shares-account-parameters-and-privileges)
8. [Security Patterns for Data Sharing](#8-security-patterns-for-data-sharing)
9. [Sharing Method Selection Guide](#9-sharing-method-selection-guide)
10. [Exam Tips & Common Gotchas](#10-exam-tips--common-gotchas)

---

## 1. The Snowflake Data Sharing Philosophy

Snowflake's approach to data sharing is architecturally different from every traditional data distribution model. In legacy systems, sharing data meant making a copy — extracting data, transforming it, transmitting it, and loading it into the recipient's system. This process is slow, expensive, creates data versioning problems, and requires the recipient to re-secure and govern their own copy. Each copy immediately starts drifting from the source the moment it is created.

Snowflake's **Secure Data Sharing** eliminates the copy entirely. When a provider shares data with a consumer, the consumer does not receive a copy of the data. Instead, they receive **read-only access to the provider's actual data storage layer** — the same micro-partitions that the provider's own queries read. There is no data movement, no ETL, no synchronization lag, and no additional storage charge to the provider for the share. The consumer always sees the current state of the shared data, the same as if they were querying the provider's own tables.

This architecture is possible because of Snowflake's separation of storage and compute. Both the provider's and consumer's virtual warehouses can be pointed at the same underlying cloud storage objects simultaneously. The storage layer enforces that the consumer's queries are read-only — they can SELECT but cannot INSERT, UPDATE, or DELETE in shared databases.

**The three principles that govern all Snowflake data sharing:**

**Zero-copy sharing:** No data is duplicated for the share. Shared data is served from the provider's storage directly, with the consumer's compute paying for the query execution. This means the consumer always has fresh, live data.

**Live data:** The consumer always sees the most recent committed state of the shared data. There is no replication lag for same-region sharing. The consumer's "shared database" is a logical pointer to the provider's objects, not a physical copy.

**Provider-controlled access:** The provider retains full control at all times — which accounts can access the share, which objects are exposed, and which governance policies (masking, row access) apply. A provider can revoke a consumer's access at any moment, and that consumer immediately loses access to all data in the share.

---

## 2. The Share Object – The Foundation of All Sharing

At the technical heart of Snowflake's data sharing model is the **share object** — an account-level container that defines what data is being shared and with whom. Understanding what a share is and how it is composed is foundational to understanding all sharing methods.

A share is created in the **provider's account**. It acts as a descriptor: it names the objects (databases, schemas, tables, secure views, secure UDFs) that are being made available, and it names the **consumer accounts** that are granted access to those objects. The share itself contains no data — it is purely metadata.

When a consumer account is added to a share, the consumer can create a **shared database** in their own account that references the share. This shared database is a logical construct with no local storage cost. When a user in the consumer account queries the shared database, their virtual warehouse executes the query, but the actual data is read from the provider's storage. The result is returned to the consumer's session.

**What can be added to a share:**

A share can include databases, schemas, tables, external tables, secure views, secure materialized views, and secure UDFs. The restriction on what can be shared is important: **regular (non-secure) views cannot be shared**. This requirement exists because a regular view's SQL definition is visible to users who have SELECT on the view, which would expose the provider's underlying schema design. Only secure views, which hide their definitions, can be shared — ensuring consumers cannot reverse-engineer the provider's data architecture.

An important capability for complex sharing scenarios is the ability to **share data from multiple databases** in a single share. A secure view in Database A can reference tables in Database B. To include that view in a share, the provider must grant `REFERENCE_USAGE` on Database B to the share — this allows the share to read from Database B without including Database B's tables in the share directly.

**The two approaches to adding objects to a share:**

The first approach is to grant privileges directly to the share object — the provider grants `SELECT ON TABLE` or `USAGE ON SCHEMA` directly to the share. The second approach, considered the best practice for modern implementations, is to create a **database role** that encapsulates the access privileges, and then grant that database role to the share. The database role approach is preferred because it is cleaner, more auditable, and allows the same database role to be granted to both the share and to account roles within the provider's organization — creating a single, coherent access control definition.

---

## 3. Use Cases by Sharing Scope

Different business scenarios require different sharing architectures. Understanding which Snowflake mechanism applies to which scope — and why — is a primary exam topic.

### 3.1 Sharing Within the Same Snowflake Account

When sharing happens within the same Snowflake account — for example, between departments, teams, or business units that all operate under one account — no share object is needed at all. This is the simplest sharing scenario, and the solution is **RBAC with database roles**.

Within a single account, all data objects are in the same namespace. Access control is entirely a matter of granting the right privileges to the right roles. A Finance database can share a schema with an HR team by granting the appropriate database role to the HR team's functional role. The data is not replicated — both teams query the same underlying storage through their respective virtual warehouses.

For intra-account sharing where one team wants to present a curated, clean view of their data to another team without exposing raw tables or transformation logic, **secure views** serve as the right tool. A data engineering team might create secure views over raw ingestion tables, exposing only the cleaned, business-ready columns to analyst teams while hiding the underlying schema complexity.

The governance tools that apply — Dynamic Data Masking, Row Access Policies, Object Tags — all work transparently within a single account. A masking policy applied to a table protects that table equally whether the querying role belongs to the data owner's team or a shared consumer team.

**Key characteristics:**
- No share object required
- No network boundary crossed
- Pure RBAC with database roles and secure views
- Governance policies apply uniformly
- Appropriate for inter-departmental, inter-team sharing

### 3.2 Sharing Within the Same Cloud Region (Different Accounts)

When the provider and consumer are in different Snowflake accounts but within the same cloud region — for example, both on AWS `us-east-1` — this is where **direct sharing with a Share object** is native and most efficient.

Because both accounts are in the same region, they share the same underlying cloud storage infrastructure. The consumer's account can reference the provider's storage objects directly, without any replication. Data access is truly instantaneous and always current.

This is the foundational use case for which Snowflake's data sharing was originally designed. A data provider with a Snowflake account in AWS East can share a product catalog, a reference dataset, or a curated analytics dataset with any number of partner accounts in the same region, with zero data movement and zero synchronization overhead.

**Key characteristics:**
- Requires Share object in the provider's account
- Consumer must be in the same cloud provider and region
- Zero data movement — consumer reads from provider's storage
- Data is always current (no replication lag)
- Managed via Direct Share (in the same region) or Listing

### 3.3 Sharing Across Cloud Regions

Sharing across regions — for example, from AWS `us-east-1` to AWS `eu-west-1` — requires **data replication**, because the consumer's account cannot directly access storage objects in a different region. This is a fundamental cloud infrastructure constraint, not a Snowflake limitation.

Snowflake's mechanism for cross-region sharing is to replicate the provider's shared data to a location accessible to the consumer. This replication introduces considerations that do not exist in same-region sharing:

**Replication lag:** The shared data in the consumer's region is a replica of the provider's primary data. The replica is updated on a schedule (which can range from minutes to hours, or can be configured for continuous replication on Business Critical Edition). The consumer always sees data as of the last successful replication — not necessarily the most current state in the provider's account. Understanding this lag and communicating it to consumers is important for use cases where data freshness matters.

**Data residency and sovereignty:** Replicating data across regions means data physically moves to a different geographic location. For organizations subject to GDPR, data protection laws, or industry-specific regulations that restrict where data can reside, cross-region sharing requires careful compliance analysis before it can be enabled.

**Cost:** Cross-region data replication incurs storage costs in the destination region and data transfer costs for the replication itself. These are ongoing costs that scale with data volume and update frequency.

Snowflake's **Cross-Cloud Auto-Fulfillment** (covered in detail in Section 6.4) automates the replication aspect of cross-region sharing for Listings, eliminating the need for providers to manually manage replication pipelines.

**Key characteristics:**
- Requires data replication to the consumer's region
- Replication lag exists — data freshness depends on refresh frequency
- Data residency compliance must be evaluated
- Cross-region transfer costs apply
- Manageable via manual replication or Cross-Cloud Auto-Fulfillment for Listings

### 3.4 Sharing Between Different Snowflake Accounts (Cross-Organization)

This covers the scenario of sharing between two organizations that each have their own independent Snowflake accounts — for example, a manufacturer sharing inventory data with a logistics partner, or a financial institution sharing market data with a client.

The mechanism depends on whether the organizations are in the same region:

- **Same region:** A Direct Share is the most direct mechanism — the provider creates a share and adds the partner's account identifier. The partner creates a shared database from the share.
- **Different regions:** Cross-Cloud Auto-Fulfillment via a Listing is the recommended mechanism, as it automates replication to the partner's region.
- **If the relationship involves a broader ecosystem of partners:** A Data Exchange or the Snowflake Marketplace may be more appropriate than managing individual direct shares.

A critical point for cross-organization sharing: data sharing is **only supported between Snowflake accounts**. A consumer must have a Snowflake account (or be given access through a Reader Account) to receive shared data.

### 3.5 Sharing to a Non-Snowflake Customer

The scenario of sharing with a recipient who does not have a Snowflake account is handled through **Reader Accounts** (formerly called "read-only accounts"). A Reader Account is a special type of Snowflake account that is created and fully managed by the **data provider**, not by the consumer.

The provider creates a Reader Account, creates users within it, and shares their data with it exactly like any other consumer account. Users of the Reader Account can log in to Snowflake and query the shared data using the provider's virtual warehouses — which means the **provider pays all compute costs** for the Reader Account's query execution. The consumer cannot create their own objects, load their own data, or run any workload that isn't querying the provider's shared data.

Reader Accounts serve as a bridge for consumers who are not yet Snowflake customers and not yet ready to sign a Snowflake license agreement. They provide immediate access to shared data without requiring the consumer to make any commercial commitment to Snowflake. For providers, Reader Accounts are a way to demonstrate the value of the data to prospective customers before those customers commit to their own Snowflake accounts.

**Key limitations of Reader Accounts:**
- The provider bears all compute costs — Reader Account queries run on provider-managed warehouses
- The consumer cannot load their own data or create their own databases
- The consumer cannot join the shared data with their own datasets (as they have no datasets)
- Snowflake Data Clean Rooms do not support Reader Accounts as participants
- Not suitable as a long-term arrangement for sophisticated data consumers who need to enrich shared data

**Key characteristics:**
- Provider creates and fully manages the Reader Account
- Consumer does not need a Snowflake account or license
- Provider pays all compute costs
- Consumer can only read shared data, not create objects
- Appropriate for one-directional data distribution to unsophisticated or non-committed consumers

### 3.6 Sharing Across Cloud Providers

Sharing across cloud providers — for example, from a Snowflake account on AWS to a consumer with a Snowflake account on Azure or GCP — is the most architecturally complex sharing scenario. It requires both cross-region and cross-cloud data movement.

Snowflake supports this through the combination of **data replication** and the **Listings / Cross-Cloud Auto-Fulfillment** mechanism. The provider's data is replicated to a secure share area in the consumer's cloud provider and region. The consumer then accesses this replicated data through their normal Snowflake account.

Without Cross-Cloud Auto-Fulfillment, a provider who wanted to reach consumers on all three cloud providers would need to manually manage a secondary account in each cloud provider's region, replicate their data to each, and manage separate shares. This is operationally complex and error-prone.

Cross-Cloud Auto-Fulfillment eliminates this complexity — the provider enables auto-fulfillment for their Listing, selects which cloud regions and providers the listing should be available in, and Snowflake's internal machinery handles all replication automatically. The consumer simply requests access to the listing in their region, and the data is already there (or is replicated on demand when the consumer requests it).

**Important constraints for cross-cloud sharing:**
- Private connectivity (PrivateLink) does not work across cloud providers — private connectivity requires the client and Snowflake to be on the same cloud
- The cross-cloud replication incurs cross-cloud egress costs (typically higher than same-cloud cross-region costs)
- Data residency rules must be evaluated carefully when data crosses both regional and cloud-provider boundaries

### 3.7 Sharing Using Snowflake Data Clean Rooms

**Snowflake Data Clean Rooms** address a fundamentally different problem than all other sharing scenarios. In every other scenario, the provider is willing to expose some or all of their data to the consumer — the data flows, subject to governance controls. In a Data Clean Room scenario, the data must **never leave the control of its owner**, even to a trusted partner.

A Data Clean Room is a secure computational environment where two or more parties can bring their data and run **pre-approved analyses** against the combined dataset, without any party being able to see the other's raw data directly. The results are aggregated or otherwise privacy-preserving outputs — no participant can query the other's raw rows.

**The canonical use case:** Two competing retailers want to understand customer overlap — which customers shop at both stores. Neither is willing to share their full customer list with the other (it's commercially sensitive). In a Data Clean Room, both parties contribute their hashed customer identifiers. Pre-approved analyses count the overlap, compute affinity scores, or identify co-marketing opportunities. Neither party ever receives the other's raw customer data — only the agreed-upon analytical outputs.

**How Snowflake Data Clean Rooms work technically:**

Snowflake Data Clean Rooms are built on the Snowflake Native App Framework (following Snowflake's acquisition of Samooha in December 2023). The provider creates a clean room environment, defines **templates** — which are SQL queries that have been pre-approved and locked for execution in the collaboration — and invites consumer accounts as participants. Consumers can only run the approved templates against the combined data. They cannot modify the templates or write ad-hoc SQL that would expose raw data.

The key roles in a Data Clean Room collaboration:
- **Owner:** Creates the clean room and controls the collaboration specification
- **Data provider:** Links their data into the clean room for analysis
- **Analysis runner:** Executes the pre-approved templates against the linked data
- Multiple parties can play multiple roles in the same collaboration

**Differential privacy** can be optionally enforced within clean rooms, adding statistical noise to outputs to prevent re-identification of individuals even from aggregated results — an important capability for healthcare, advertising, and financial services use cases.

**Limitations to be aware of:**
- All participants must have a Snowflake account — Reader Accounts are not supported in clean rooms
- By default, clean rooms are limited to participants in the same cloud region. Cross-region collaboration requires Cross-Cloud Auto-Fulfillment to be enabled
- Cross-region replication introduces latency in the collaboration — requests and data between provider and consumer are subject to the configured refresh frequency
- An account cannot simultaneously act as both provider and consumer in the same cross-cloud clean room collaboration (due to replication type conflicts)
- Data must be aggregated in outputs to protect raw data — this limits the granularity of insights that can be derived

---

## 4. Snowflake Marketplace

The **Snowflake Marketplace** is a global data storefront built directly within the Snowflake platform, operated by Snowflake Inc., where data providers publish datasets, data services, and Snowflake Native Apps that consumers can discover and access from within their own Snowflake accounts.

The Marketplace is the most open-ended sharing channel: providers can make their listings available to any Snowflake account globally. Consumers browsing the Marketplace can discover datasets from across the data ecosystem — financial data, geospatial data, weather data, economic indicators, consumer behavior data, and thousands of other categories — and gain access in minutes, with no ETL, no infrastructure, and no data movement.

### The Provider Experience on the Marketplace

Any Snowflake account can become a data provider on the Marketplace, subject to approval of a **provider profile** — a business identity and legal agreement reviewed by Snowflake. The provider profile establishes the provider's public identity in the Marketplace, their terms of service, and their contact information.

Once a profile is approved, a provider can publish **listings** to the Marketplace. A listing packages a share (or a Native App) with metadata: a descriptive title, data dictionary, sample SQL queries, use cases, and data freshness information. Metadata quality is important — well-documented listings with clear use cases attract more consumers than bare data shares with no context.

**Listing types on the Marketplace:**

**Free listings** make data available to any Snowflake consumer at no charge. The consumer clicks "Get" and immediately has access to a shared database in their account. Free listings are used by providers who want broad adoption — data vendors building network effects, organizations sharing public or semi-public reference data, or companies that monetize downstream rather than directly from data access.

**Paid listings** allow providers to charge consumers for data access. Snowflake handles the billing — the consumer is charged via their Snowflake account, and Snowflake remits payment to the provider. This turns data assets into a revenue stream without the provider needing to manage billing infrastructure. Pricing models can be based on flat monthly fees, per-row usage, or other structures.

**Limited trial listings** offer a preview of a dataset — a subset of rows or a time-limited window — allowing consumers to evaluate the data before committing to a full paid subscription.

**Personalized listings** require the consumer to provide additional information (contact details, intended use case) before access is granted. The provider reviews the request and manually approves or rejects it. This is appropriate for sensitive datasets where the provider needs to know who is accessing the data.

### The Consumer Experience on the Marketplace

From the consumer's perspective, the Marketplace is a library of live datasets accessible within their existing Snowflake account. A consumer discovers a dataset, reviews the listing's documentation and sample queries, and clicks "Get." Within seconds, they have a new shared database in their Snowflake account. They can immediately join that dataset with their own internal tables — no API calls, no file downloads, no data integration pipelines.

Crucially, the data is not copied to the consumer's account — they are reading from the provider's storage (or a regional replica managed by auto-fulfillment for cross-region consumers). The consumer's virtual warehouse pays for query compute, but there is no storage cost for the shared data.

### Provider Visibility and Analytics

Providers with Marketplace listings have access to **usage analytics** in the `SNOWFLAKE.DATA_SHARING_USAGE` schema. This schema provides views on consumer engagement: how many accounts have requested the listing, how many active queries have run against shared data, which queries were most common, and usage trends over time. This data helps providers understand the value and adoption of their data products and refine their offerings.

---

## 5. Data Exchange

A **Data Exchange** is a private, curated version of the Snowflake Marketplace that an organization creates and controls for a specific community of accounts. Where the Marketplace is open to all Snowflake accounts globally, a Data Exchange is invite-only — the Exchange administrator determines which accounts can participate as providers and which can participate as consumers.

### The Governance Model

The key structural difference from other sharing methods is the **three-role governance model** of a Data Exchange:

**Data Exchange Administrator:** The Snowflake account that hosts the Data Exchange is the administrator. The ACCOUNTADMIN in that account configures the exchange — setting membership policies, approval workflows, and visibility rules. The administrator can delegate exchange management privileges to other roles, allowing a Data Exchange to be governed by a dedicated team rather than requiring ACCOUNTADMIN for all management tasks. For Snowflake Marketplace, Snowflake Inc. is the administrator. For a private Data Exchange, the hosting organization is the administrator.

**Data Providers within the Exchange:** Member accounts designated as providers can publish listings to the exchange — making their data available to the consumer members. Providers can require consumer approval before granting access, or make data freely available to all exchange members. Providers need the `CREATE LISTING` privilege within the exchange to publish.

**Data Consumers within the Exchange:** Member accounts designated as consumers can browse the exchange, discover listings, and request or directly access data. Consumers need the `IMPORT SHARE` privilege to create a database from a shared dataset.

### When a Data Exchange is the Right Choice

A Data Exchange is appropriate when an organization needs controlled, discoverable sharing within a defined community — situations where the Marketplace is too open (any Snowflake account could access the data) and Direct Shares are too manual (managing individual share-to-account relationships at scale becomes unwieldy).

**Industry consortiums:** A group of companies in the same vertical — banks, insurers, pharmaceutical companies — that want to share reference data, compliance datasets, or benchmarks among themselves, without exposing those datasets to the broader Snowflake ecosystem.

**Enterprise data products:** A large organization with many business units, each operating their own Snowflake account, that wants to create an internal data marketplace where teams can publish and discover data products in a governed way. The central data team administers the exchange, sets publication standards, and curates the catalog.

**Partner ecosystems:** An organization that distributes data to a defined set of partners — resellers, franchisees, subsidiaries — where each partner has their own Snowflake account and there is a need for a structured discovery and access mechanism rather than individual direct share management.

**Key distinction from Marketplace:** The Marketplace is operated by Snowflake and open to all Snowflake customers. A Data Exchange is operated by a specific organization and accessible only to invited member accounts. An organization can participate in both — exposing some data publicly on the Marketplace while reserving other data for a private exchange.

---

## 6. Data Sharing Methods – Deep Dive

### 6.1 Direct Share

**Direct Share** is the most primitive and foundational sharing method in Snowflake. A provider creates a share object in their account, adds database objects to it, and explicitly adds one or more consumer account identifiers to the share. There is no discovery mechanism — the provider must know the consumer's account identifier and explicitly authorize that specific account.

**What Direct Share is best for:** It is the right tool for one-to-one or one-to-very-few sharing within the same cloud region, where the relationship is already known and trusted, and where no additional metadata, approval workflow, or monetization is needed. A parent company sharing a data product with a single subsidiary, or a data team sharing a curated dataset with one specific analytics partner.

**The critical constraint:** Direct Shares function only within the same cloud provider and region. A Direct Share created in an AWS `us-east-1` account can only be consumed by accounts also in AWS `us-east-1`. If the consumer is in a different region or cloud, a Listing with Auto-Fulfillment must be used instead.

**How it works operationally:** The provider account requires the `CREATE SHARE` privilege (ACCOUNTADMIN by default, but can be delegated). The consumer account requires the `IMPORT SHARE` privilege (ACCOUNTADMIN by default) to create a database from the share. Once the consumer creates their shared database, any role granted `IMPORTED PRIVILEGES` on that database can query it.

Converting a Direct Share to a Listing is supported — a provider can upgrade an existing direct share relationship to a listing without disrupting the consumer's access. This is the natural path as a sharing relationship matures and requires broader distribution or commercial terms.

### 6.2 Listings (Private and Public)

A **Listing** is the evolved, metadata-rich form of a share. While a Direct Share is a bare technical connection between a provider's data and a consumer's account, a Listing wraps that share in a data product experience: a descriptive title, business context, sample queries, data documentation, refresh frequency, and access management.

**Private Listings** are listings published to specific named consumer accounts (identified by their `<orgname>.<account_name>` identifier). The provider knows who the consumers are and deliberately grants access to those specific accounts. Private Listings support:
- Sharing to specific accounts in other regions (with Auto-Fulfillment handling cross-region replication automatically)
- Optional approval workflows where the consumer must request access and the provider approves
- Usage analytics showing consumer query patterns

**Public Listings** are published to the Snowflake Marketplace and discoverable by any Snowflake account globally. The provider has no prior relationship with the consumers — anyone can find and request the listing. Public listings support the full range of monetization options (free, trial, paid).

**Key advantage of Listings over Direct Shares:** Listings provide a consistent consumer experience regardless of where the consumer's account is located — same region, different region, or different cloud. The Auto-Fulfillment mechanism makes regional differences transparent to both provider and consumer. Direct Shares can only serve same-region consumers.

**A share can only be attached to one listing.** A provider cannot attach the same share to multiple listings simultaneously. If a share is already attached to a listing, it must be detached (which deletes that listing) before it can be attached to a new listing.

### 6.3 Reader Accounts

Reader Accounts are provider-created Snowflake accounts that give non-Snowflake customers access to shared data. They are the bridge for sharing to the non-Snowflake world.

A Reader Account is a fully functional Snowflake account in terms of authentication and querying capability, but it is structurally limited:
- It cannot be the source of any share — Reader Accounts cannot share data
- Users can only query data shared with the account by its provider parent
- It cannot join as a member of a Data Exchange
- Users in a Reader Account cannot load their own data or create databases, schemas, or tables independent of the share

The **provider bears all costs** associated with Reader Account usage. When a Reader Account user runs a query, the compute runs on a virtual warehouse created and paid for by the provider. This means providers must budget for Reader Account usage and size the shared virtual warehouses appropriately.

**The lifecycle of a Reader Account relationship:** Reader Accounts are typically a transitional mechanism. As a non-Snowflake consumer begins to see value in the shared data and wants to enrich it with their own data, they will naturally want their own Snowflake account. At that point, the relationship transitions from Reader Account access to a normal Direct Share or Listing — with the consumer now having their own warehouse, their own storage, and the ability to join shared data with their own data.

### 6.4 Cross-Cloud Auto-Fulfillment

**Cross-Cloud Auto-Fulfillment** (also called Listings Auto-Fulfillment, or LAF) is the mechanism that allows a provider's Listing to be consumed by accounts in any Snowflake region and on any cloud provider — without the provider manually managing replication to each destination region.

Before Auto-Fulfillment existed, cross-region sharing required providers to:
1. Set up a secondary Snowflake account in each target region
2. Manually configure database replication from the primary account to each secondary
3. Manage the replication schedules, monitor for failures, and handle schema changes
4. Create separate shares in each regional account

This was operationally expensive, error-prone, and scaled poorly with the number of target regions. A provider supporting consumers across five regions on three cloud providers would need to manage fifteen separate regional accounts and replication pipelines.

**Auto-Fulfillment automates all of this.** When a provider enables Auto-Fulfillment for a Listing, Snowflake's internal infrastructure manages the following on the provider's behalf:

1. Creates an internal **Secure Share Area** in each target region — a Snowflake-managed regional storage location where the replicated data product is stored
2. Replicates the provider's data product (the share's tables, views, and other objects) to the Secure Share Area in each region where there is consumer demand
3. Keeps the replicated data fresh according to the configured refresh interval
4. When a new region gets its first consumer, replication to that region is initiated automatically

From the consumer's perspective, the experience is identical regardless of region — they find the listing, click Get, and immediately have a shared database. The fact that the data was replicated from a provider in a different cloud is entirely transparent.

**Cost model for Auto-Fulfillment:**
- Storage costs are incurred in each region where data is replicated (charged to the provider)
- Cross-cloud and cross-region data transfer costs apply for the replication
- Critically, **costs are only incurred where there is actual consumer demand** — Snowflake does not pre-replicate to every possible region. Replication to a region is triggered when a consumer in that region requests the listing
- For paid listings, the minimum refresh interval is 8 days (to avoid excessive replication costs)

**The privilege architecture for Auto-Fulfillment is hierarchical:**
- Only `ORGADMIN` can enable or disable Auto-Fulfillment at the account level (this is a one-time organizational enablement)
- ORGADMIN can delegate the `MANAGE LISTING AUTO FULFILLMENT` privilege to other roles
- Those roles can then enable or configure Auto-Fulfillment for specific listings

**What gets replicated:** Auto-Fulfillment replicates the objects included in the share — tables, secure views, UDFs, and other shareable objects. If a share includes objects from multiple databases (via `REFERENCE_USAGE`), those referenced objects are also replicated. The internal objects that Snowflake creates for auto-fulfillment management (replication groups, task databases, internal roles) appear in `SHOW DATABASES` and `SHOW ROLES` — these must not be modified or granted to users.

**Limitations:**
- Auto-Fulfillment is not available on trial accounts
- Personalized listings (which require consumer-provided information) do not support Auto-Fulfillment for the Snowflake Marketplace
- Auto-Fulfillment for Snowflake Native Apps with Snowpark Container Services is only supported on AWS and Azure (not GCP)
- Auto-Fulfillment enforces a 10TB size limit on the database being replicated
- An account cannot be both a provider and consumer in the same cross-cloud clean room collaboration

---

## 7. Configuring Shares, Account Parameters, and Privileges

### Share Configuration

The structure of share configuration involves two distinct layers: what objects are shared (the content of the share) and who receives access (the consumers of the share).

**Adding objects to a share:** There are two approaches. The direct approach grants privileges on individual objects directly to the share — granting SELECT on a table, USAGE on a schema, USAGE on the database, all directly to the share. The database role approach creates a database role, grants the required object privileges to that role, and then grants the database role to the share. The database role approach is strongly preferred in modern implementations because it creates a reusable, auditable permission unit that can be consistently applied.

**Consumer account management:** Consumers are added to a share by specifying their Snowflake account identifier (the combination of their organization name and account name). Within the same organization, internal account names suffice. For external accounts, the full `<orgname>.<accountname>` format is required.

**Multi-database shares:** When a shared view references tables in a different database than the view itself, the provider must grant `REFERENCE_USAGE` on the referenced database to the share. This allows the share to traverse the database boundary for the purpose of resolving the view's query, without adding the referenced database's tables to the share directly.

### Account Parameters Relevant to Sharing

`DATA_RETENTION_TIME_IN_DAYS` affects how long Time Travel history is retained on shared objects. Consumers can query shared objects as of any point within the provider's Time Travel window — meaning a provider with 30-day retention gives consumers the ability to query historical states of shared data up to 30 days back.

`ENABLE_DATA_SHARING` must be set to TRUE (the default for all accounts except some trial accounts). If an account has this parameter set to FALSE, it can neither create shares nor consume shares.

`REQUIRE_STORAGE_INTEGRATION_FOR_STAGE_CREATION` does not directly affect shares, but is relevant when external stages that reference shared data are part of a sharing architecture.

### Key Privileges for Data Sharing

**Provider-side privileges:**

| Privilege | Purpose | Default Role |
|---|---|---|
| `CREATE SHARE` | Allows creation of share objects | ACCOUNTADMIN |
| `CREATE LISTING` | Allows creation of listings in a Data Exchange or Marketplace | ACCOUNTADMIN |
| `MANAGE LISTING AUTO FULFILLMENT` | Allows enabling/configuring Auto-Fulfillment for listings | ORGADMIN (delegatable) |

**Consumer-side privileges:**

| Privilege | Purpose | Default Role |
|---|---|---|
| `IMPORT SHARE` | Allows viewing inbound shares and creating a database from a share | ACCOUNTADMIN |
| `IMPORTED PRIVILEGES` | Grants access to query objects within a shared database | Granted by ACCOUNTADMIN after creating shared database |

**On the provider side:** An ACCOUNTADMIN can delegate share creation by granting the `CREATE SHARE` privilege to a custom role. This allows a data engineering team to manage shares without needing ACCOUNTADMIN — following least-privilege principles. Similarly, `CREATE LISTING` can be delegated for listing management.

**On the consumer side:** After an ACCOUNTADMIN creates the shared database from a share, they must explicitly grant `IMPORTED PRIVILEGES` on that database to the roles that should query it. No user can automatically query a shared database just because it was created — the ACCOUNTADMIN controls who within the consumer organization can access it.

---

## 8. Security Patterns for Data Sharing

Security in shared data is a multi-layered concern. The provider must think not just about who can access the share, but about what those consumers see within the shared data.

### Secure Objects Are Mandatory for Shares

As noted in Section 2, only **secure views** (not regular views) can be included in a share. This requirement is enforced by Snowflake — an attempt to add a non-secure view to a share will fail. The reason is that regular views expose their SQL definition to any user with SELECT on the view, which could expose the provider's schema structure, business logic, or security filtering conditions to consumers.

Secure views hide their definition from non-owners. When a consumer's analyst queries a shared secure view, they receive query results but cannot inspect the SQL that generated those results. This protects the provider's data modeling intellectual property and prevents consumers from understanding what data they are not seeing (if the view filters rows based on security logic).

For the same reason, **secure UDFs** (user-defined functions) must be used in shares rather than standard UDFs — the function code is hidden from consumers.

### Applying Governance Policies to Shared Data

Masking policies and row access policies applied to shared tables and views are honored when consumers query the shared data. This is a powerful governance capability: the provider can apply different masking policies to the same column, and the `INVOKER_SHARE()` context function allows the masking policy to detect which share (and therefore which consumer) is executing the query.

This enables **differential masking by consumer**: Consumer A (a trusted analytics partner) might see unmasked email addresses, while Consumer B (a research partner) sees only masked values — even though they both query the same underlying table through the same share mechanism, or through different shares pointing to the same table.

The `INVOKER_ROLE()` function works similarly — it captures the role of the user executing the query within the consumer account, allowing masking policies to be role-aware even for consumer accounts.

**Row access policies** on shared tables filter the rows a consumer can see, applied transparently at query time. A provider can share a single customer table but use a row access policy to ensure Consumer A only sees rows for customers in the Americas region, while Consumer B only sees European customers — without maintaining separate physical tables per consumer.

### What Consumers Cannot Do

The security model for shared data enforces read-only access as an absolute constraint. Consumers cannot INSERT, UPDATE, DELETE, MERGE, or TRUNCATE data in shared databases. They cannot CREATE objects within shared schemas. They cannot alter the structure of shared tables. These restrictions are enforced at the storage layer, not just at the SQL parsing layer — a consumer with ACCOUNTADMIN in their own account still cannot write to a shared table.

### External Tokenization Caveat

One important constraint: **external tokenization does not work with data sharing**. External tokenization uses masking policies that call external functions. External functions cannot be invoked in the context of a share — the security boundary of the share prevents outbound calls to external systems. If a provider needs to share tokenized data, they must use Dynamic Data Masking (which performs transformation within Snowflake's engine) rather than external tokenization.

### Provider Revocation

The provider retains absolute power to revoke access at any time. Removing a consumer account from a share immediately terminates that account's ability to query the shared data — there is no grace period, no data export window. The consumer's shared database still exists as an object in their account, but querying it returns an error. This gives providers full control over the data relationship at all times.

---

## 9. Sharing Method Selection Guide

| Sharing Scenario | Recommended Method | Key Reason |
|---|---|---|
| Same Snowflake account, different teams | RBAC with database roles + secure views | No share object needed; pure access control |
| Same region, known external account | Direct Share | Simplest mechanism; zero-copy, always current |
| Same region, one-to-many known accounts | Private Listing | Better metadata, discovery, and usage analytics than managing many direct shares |
| Different regions, known accounts | Private Listing + Auto-Fulfillment | Automates cross-region replication; consumer gets transparent experience |
| Public data product for any Snowflake account | Snowflake Marketplace (Public Listing) | Maximum reach; built-in discovery, monetization, and usage analytics |
| Private community of curated providers/consumers | Data Exchange | Controlled membership, governed publishing, approval workflows |
| Consumer has no Snowflake account | Reader Account | Provider creates and manages the account; consumer queries shared data only |
| Sensitive collaboration where raw data must never be exposed | Data Clean Room | Both parties' raw data is never visible to the other; only approved query outputs |
| Cross-cloud sharing at scale | Listing + Cross-Cloud Auto-Fulfillment | Automated multi-region/cloud replication without manual pipeline management |
| Sharing data with governance policies (masking, row filters) | Secure Views + Masking/RAP policies | Governance applies at query time on the provider's side; consumer sees governed output |

---

## 10. Exam Tips & Common Gotchas

### ⚡ High-Yield Conceptual Points

**Secure Data Sharing is zero-copy.** No data is duplicated for the share. The consumer queries the provider's storage directly (for same-region shares) or a Snowflake-managed replica (for cross-region). There is no ETL, no synchronization, and no versioning problem.

**Data sharing is only supported between Snowflake accounts.** Non-Snowflake consumers must use Reader Accounts, which the provider creates and manages.

**Direct Shares only work within the same cloud provider and region.** A Direct Share created in AWS `us-east-1` cannot be consumed by an account in AWS `eu-west-1` or in Azure. For cross-region consumers, a Listing with Auto-Fulfillment is required.

**Regular views cannot be shared — only secure views.** This is enforced by Snowflake and cannot be worked around. Secure views hide their definition from consumers.

**External tokenization does not work with data sharing.** Masking policies in a share context cannot call external functions. Dynamic Data Masking is supported; External Tokenization is not.

**The consumer pays for compute, not the provider.** When a consumer runs a query against shared data, the consumer's virtual warehouse executes the query and the consumer's account is billed for the compute credits. The provider pays only for storage of the shared data. The exception is Reader Accounts, where the provider pays all costs.

**Reader Account users cannot create their own objects.** A Reader Account is read-only from the consumer's perspective — they can only query data shared with them by the provider.

**A share can only be attached to one listing.** Once a share is attached to a listing, it cannot be attached to another — even if the original listing is deleted.

**ORGADMIN must enable Cross-Cloud Auto-Fulfillment at the account level** before any role can configure it for listings. This is an organizational enablement that flows downward — ORGADMIN enables the account, then delegates the `MANAGE LISTING AUTO FULFILLMENT` privilege to provider roles.

**Auto-Fulfillment costs are incurred only when there is consumer demand in a region.** Snowflake does not pre-replicate to every possible region — replication is triggered by consumer activity in that region.

**Data Clean Rooms do not support Reader Accounts.** All participants in a clean room collaboration must have full Snowflake accounts.

**In a Data Exchange, the hosting account is the administrator.** For the Snowflake Marketplace, Snowflake Inc. is the administrator. This is the key distinction between the two.

**Governance policies (masking, row access) apply to shared data at query time.** A masking policy on a shared table is evaluated when the consumer queries it — the consumer sees governed output, not raw data. The `INVOKER_SHARE()` function allows policies to be share-aware.

**The consumer's ACCOUNTADMIN must grant IMPORTED PRIVILEGES** to roles within the consumer account after creating the shared database. Simply creating the shared database does not automatically expose it to all roles in the consumer account.

**Cross-region sharing introduces replication lag.** Same-region sharing is always current. Cross-region sharing through Auto-Fulfillment is subject to the configured refresh interval — typically measured in hours. Use cases requiring real-time freshness across regions must account for this lag.

**Time Travel on shared data uses the provider's retention window.** A consumer can query a shared table as of any point within the provider's Time Travel retention period. The consumer does not control this — the provider's `DATA_RETENTION_TIME_IN_DAYS` setting determines what historical states are accessible.

### 🚫 Classic Exam Traps

| What the Exam Tests | The Correct Answer |
|---|---|
| "Does data sharing create a copy of data for the consumer?" | No — zero-copy sharing; consumer reads from provider's storage (or Snowflake-managed replica for cross-region) |
| "Can a regular view be included in a share?" | No — only secure views can be shared |
| "Who pays compute for queries in a shared database?" | The consumer pays (exception: Reader Accounts, where the provider pays) |
| "A consumer in AWS us-east-1 and provider in Azure West Europe — what sharing method is required?" | Cross-Cloud Auto-Fulfillment via a Listing — Direct Shares cannot cross cloud providers or regions |
| "Can external tokenization be used in a shared masking policy?" | No — external functions cannot be called in a share context; only DDM is supported |
| "A company wants to share data with a partner who has no Snowflake account. What is the mechanism?" | Reader Account — created and managed by the provider |
| "Who administers the Snowflake Marketplace?" | Snowflake Inc. |
| "Who administers a private Data Exchange?" | The Snowflake account that hosts the Exchange (the customer's own account, not Snowflake) |
| "Can a share be attached to multiple listings?" | No — one share can only be attached to one listing at a time |
| "What must happen before ACCOUNTADMIN can enable Auto-Fulfillment for a listing?" | ORGADMIN must first enable Auto-Fulfillment at the account level |
| "After a consumer creates a shared database, can all roles in that account query it?" | No — the ACCOUNTADMIN must explicitly grant IMPORTED PRIVILEGES to the roles that should query it |
| "Can a Reader Account be a participant in a Snowflake Data Clean Room?" | No — Reader Accounts are not supported in clean rooms |
| "In a Data Clean Room, can a consumer write ad-hoc SQL against the provider's raw data?" | No — consumers can only run pre-approved templates; ad-hoc SQL is not permitted |
| "What privilege does a consumer-side account need to create a database from an inbound share?" | IMPORT SHARE (ACCOUNTADMIN by default, but can be granted to other roles) |
| "Does Auto-Fulfillment pre-replicate data to all regions when a listing is published?" | No — replication to a region is triggered only when there is actual consumer demand in that region |
| "Can cross-region sharing guarantee real-time data freshness?" | No — there is always a replication lag for cross-region sharing; same-region sharing is always current |
| "What is the INVOKER_SHARE() function used for?" | It identifies which share is executing a query, allowing masking policies to apply different governance rules per consumer |
| "Can a consumer account in a shared database create objects or load data?" | No — shared databases are read-only; no DML or DDL from the consumer side |

---

| Sharing Scenario | Regional & Cloud Boundary | Best Option (Recommended) | Alternate Option | Why the Best Option Wins |
| :--- | :--- | :--- | :--- | :--- |
| **Same Organization** *(Internal Accounts)* | **Same Cloud + Same Region** *(e.g., AWS East to AWS East)* | **Direct Share (SQL)** | Private Listing | **Immediate & Free:** Zero data copying required. Sharing metadata maps instantly across accounts via pure SQL with absolutely zero data transit latency or network egress costs. |
| **Same Organization** *(Internal Accounts)* | **Cross-Cloud or Cross-Region** *(e.g., AWS East to Azure Asia)* | **Direct SQL Replication Groups** | Private Listing with Auto-fulfillment | **Full Automation:** Can be managed completely as infrastructure-as-code (Terraform) and programmatically triggered via orchestration tools (like Airflow or Prefect) directly after your ETL run finishes. |
| **Different Organizations** *(External Companies)* | **Same Cloud + Same Region** *(e.g., AWS East to AWS East)* | **Direct Share (SQL)** | Private Listing | **Simplicity:** The fastest, zero-latency way to grant direct read-only access to an external partner's database view if you happen to sit in the exact same cloud datacenter region. |
| **Different Organizations** *(External Companies)* | **Cross-Cloud or Cross-Region** *(e.g., AWS East to Azure Asia)* | **Private Listing with Auto-fulfillment** | Legacy Multi-Account Replication + Direct Share | **Security & Governance Boundary:** You do not have permissions to run direct administrative SQL sync scripts across an external enterprise's infrastructure. Private listings act as a clean, automated delivery broker managed safely by Snowflake. |
| **Sharing with Non-Snowflake Users** *(External Vendors)* | **Any Cloud or Region** | **Managed Reader Accounts** | Traditional ETL Export (S3/Blob Storage) | **Keeps Data Governed:** You provision a restricted, read-only Snowflake cluster engine inside your own account infrastructure for them. They query your live data via web UI/BI tools without you ever losing control of the physical data files. |
| **Commercial Monetization** *(Public Market)* | **Global Scale (All Clouds/Regions)** | **Public Marketplace Listing** | Private Listing with Custom Contracts | **Mass Scale & Billing Built-in:** Exposes your dataset as a commercial product catalog visible to every Snowflake consumer worldwide, with automated usage tracking and unified procurement/billing handled natively by Snowflake. |

## 📚 Reference Links

| Resource | URL |
|---|---|
| About Secure Data Sharing | https://docs.snowflake.com/en/user-guide/data-sharing-intro |
| Data Sharing and Collaboration Overview | https://docs.snowflake.com/en/guides-overview-sharing |
| Create and Configure Shares | https://docs.snowflake.com/en/user-guide/data-sharing-provider |
| About Listings | https://docs.snowflake.com/en/collaboration/collaboration-listings-about |
| About Snowflake Marketplace | https://docs.snowflake.com/en/user-guide/data-marketplace |
| About Data Exchange | https://docs.snowflake.com/en/user-guide/data-exchange |
| Configure and Use a Data Exchange | https://docs.snowflake.com/en/user-guide/data-exchange-using |
| Reader Accounts | https://docs.snowflake.com/en/user-guide/data-sharing-reader-create |
| Configure a Reader Account | https://docs.snowflake.com/en/user-guide/data-sharing-reader-config |
| Share Data Across Regions and Clouds | https://docs.snowflake.com/en/user-guide/secure-data-sharing-across-regions-platforms |
| Auto-Fulfillment for Listings | https://docs.snowflake.com/en/collaboration/provider-listings-auto-fulfillment |
| Set Up Auto-Fulfillment | https://docs.snowflake.com/en/collaboration/provider-listings-auto-fulfillment-setup-steps |
| Auto-Fulfillment Costs | https://docs.snowflake.com/en/collaboration/provider-understand-cost-auto-fulfillment |
| Data Clean Rooms Overview | https://docs.snowflake.com/en/user-guide/cleanrooms/overview |
| Installing Data Clean Rooms | https://docs.snowflake.com/en/user-guide/cleanrooms/installing-dcr |
| Clean Rooms Auto-Fulfillment | https://docs.snowflake.com/en/user-guide/cleanrooms/v1/enabling-laf |
| Grant Privileges for Sharing | https://docs.snowflake.com/en/user-guide/data-exchange-marketplace-privileges |
| DATA_SHARING_USAGE Schema | https://docs.snowflake.com/en/sql-reference/data-sharing-usage |

---

*© Study Notes for Snowflake Advanced Certification — For educational use only.*
