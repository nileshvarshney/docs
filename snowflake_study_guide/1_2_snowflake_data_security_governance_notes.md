# ❄️ Snowflake Advanced Certification Study Notes
## Topic: Data Security, Privacy, Compliance & Governance Architecture

> **Exam Domain:** SnowPro Advanced – Data Engineer / Architect / Administrator
> **Last Updated:** June 2025
> **References:** Official Snowflake Documentation, Snowflake Horizon, Snowflake Engineering Blog

---

## Table of Contents

1. [RBAC – Core Concepts & Mental Model](#1-rbac--core-concepts--mental-model)
2. [Privilege Inheritance](#2-privilege-inheritance)
3. [System-Defined Roles & Best Practices](#3-system-defined-roles--best-practices)
4. [Database Roles](#4-database-roles)
5. [Functional Roles vs. Access Roles](#5-functional-roles-vs-access-roles)
6. [Secondary Roles](#6-secondary-roles)
7. [Data Access & Storage Integrations](#7-data-access--storage-integrations)
8. [Secure Views](#8-secure-views)
9. [Column-Level Security](#9-column-level-security)
   - 9.1 Dynamic Data Masking (DDM)
   - 9.2 External Tokenization
   - 9.3 Conditional Masking
10. [Row-Level Security – Row Access Policies](#10-row-level-security--row-access-policies)
11. [Aggregate Policies](#11-aggregate-policies)
12. [Projection Policies](#12-projection-policies)
13. [How All Four Policies Interact](#13-how-all-four-policies-interact)
14. [Data Lineage & Object Dependencies](#14-data-lineage--object-dependencies)
15. [Object Tagging & Data Classification](#15-object-tagging--data-classification)
16. [Compliance & Snowflake Editions](#16-compliance--snowflake-editions)
17. [Snowflake Horizon – Unified Governance](#17-snowflake-horizon--unified-governance)
18. [Exam Tips & Common Gotchas](#18-exam-tips--common-gotchas)

---

## 1. RBAC – Core Concepts & Mental Model

### What RBAC Solves

In a large data platform, managing access user-by-user becomes unscalable and error-prone. Snowflake's **Role-Based Access Control (RBAC)** solves this by introducing an intermediary layer — the **role** — between users and data objects. Privileges are granted to roles, and roles are granted to users. When access needs change, you update the role, and every user holding that role is immediately affected.

### The Two Frameworks Working Together

Snowflake actually combines two access control models simultaneously:

**RBAC (Role-Based Access Control):** Privileges on objects are assigned to roles. Users get access by being granted a role.

**DAC (Discretionary Access Control):** Every object in Snowflake has an *owner* — the role that created it. The owner can, at its discretion, grant privileges on that object to other roles.

In practice, this means the role that runs `CREATE TABLE` owns that table. Only that owning role (or a higher role in the hierarchy) can initially grant access to others. This creates a natural accountability chain: you always know which role is responsible for which objects.

### The Four Pillars of RBAC in Snowflake

**Securable Objects** are anything in Snowflake that access can be controlled on — databases, schemas, tables, views, warehouses, stages, integrations, and even roles themselves.

**Privileges** are discrete permissions on a securable object. Examples include `SELECT` on a table, `USAGE` on a database, `OPERATE` on a warehouse, or `CREATE TABLE` on a schema. Privileges are always granted *on a specific object*, not globally.

**Roles** are named collections of privileges. A role acts like a job description — it says "whoever holds this role can do these things." Roles can be granted to users or to other roles.

**Users** are the human or service identities that authenticate to Snowflake. A user gets capabilities only through the roles they have been granted.

### Critical Rule: USAGE Is Always Required

A very common exam topic — **having SELECT on a table is not enough**. To query a table, a role must have:
- `USAGE` on the **database** containing the table
- `USAGE` on the **schema** containing the table
- `SELECT` on the **table** itself

Missing any one of these three grants will result in a permission denied error, even if the other two are in place. This layered requirement exists by design — it forces administrators to make conscious decisions at every level of the object hierarchy.

---

## 2. Privilege Inheritance

### How Inheritance Works

Snowflake roles form a **hierarchy** — more precisely, a directed acyclic graph (DAG). When Role A is granted *to* Role B, Role B inherits **all privileges of Role A**. This inheritance flows **upward** through the hierarchy: child roles push their privileges up to parent roles.

Think of it like an org chart in reverse — the privileges "bubble up" from the lowest roles to the highest. `ACCOUNTADMIN`, sitting at the very top, inherits every privilege from every role below it in the hierarchy.

### The Direction of Inheritance Is Frequently Misunderstood

Many people intuitively think that a "higher" role passes privileges *down* to lower roles. In Snowflake, it is the opposite. When you **grant Role A to Role B**, you are making Role B the *parent*. Role B inherits from Role A. The child (Role A) shares upward with the parent (Role B).

```
Role A  →  granted to  →  Role B
Role B now has everything Role A had, PLUS its own direct grants.
```

### Transitive Inheritance

Inheritance is transitive. If Role A is granted to Role B, and Role B is granted to Role C, then Role C inherits the privileges of both Role A and Role B. There is no limit to the depth of this chain — Snowflake evaluates the full graph at query time to determine the combined privilege set.

### What Cannot Be Inherited

- **Account roles cannot be granted to database roles.** This is an absolute constraint. The inheritance model only flows from account role to account role, or from database role to account role. A database role can never hold, inherit from, or grant to an account role.
- **Ownership** is not inherited in the traditional sense. If a role owns an object, a parent role does not "own" that object — but the parent role can still perform all owner-level operations due to privilege inheritance.

### FUTURE GRANTS – Inheritance for New Objects

A critical concept for governance at scale: `FUTURE GRANTS` extend a privilege grant to objects that do not yet exist. For example, granting `SELECT ON FUTURE TABLES IN SCHEMA my_schema TO ROLE analyst` means the analyst role will automatically receive SELECT on every new table added to that schema going forward. Without future grants, administrators must manually re-grant after every `CREATE TABLE`, creating a governance gap.

---

## 3. System-Defined Roles & Best Practices

Snowflake ships with **six built-in system roles** that are automatically available in every account. They cannot be dropped or renamed.

### The Six System Roles

**ACCOUNTADMIN**
This is the most powerful role in Snowflake. It is the only role that can view billing and credit usage, configure account-level parameters, enable replication, and manage organization-level settings. It sits at the top of the hierarchy and inherits everything from all other system roles. Because of its power, it must be treated with extreme caution.

*Best practices:* Assign it to no more than two or three named individuals. Require MFA for all users holding ACCOUNTADMIN. Never set it as anyone's default role — it should only be activated deliberately when truly needed. Do not use it for day-to-day operations. Monitor its usage via `LOGIN_HISTORY` and `QUERY_HISTORY`.

**SECURITYADMIN**
This role is responsible for managing users, roles, and all grants across the account. It can grant and revoke any privilege on any object. It is the parent of USERADMIN in the default hierarchy, and is itself a child of ACCOUNTADMIN.

*Best practices:* Assign to the security or data governance team. Use this role for RBAC management tasks rather than ACCOUNTADMIN. Separating security administration from object administration (SYSADMIN) is a key principle of least privilege.

**USERADMIN**
A more limited role focused exclusively on user and role lifecycle management. It can create users and roles, but cannot grant object-level privileges the way SECURITYADMIN can. It sits below SECURITYADMIN in the hierarchy.

*Best practices:* Ideal for helpdesk teams or HR-integrated provisioning systems that need to create/disable users without having the ability to modify data access.

**SYSADMIN**
The primary role for managing all account-level data objects — databases, schemas, warehouses, stages, and so on. Object creation and configuration flows through SYSADMIN. Critically, all custom roles should ultimately be granted to SYSADMIN so that SYSADMIN (and therefore ACCOUNTADMIN) retains visibility and management capability over all objects.

*Best practices:* Use SYSADMIN as the "administrative parent" for all custom roles. Never let custom roles create objects without granting those custom roles to SYSADMIN.

**PUBLIC**
This role is automatically granted to every user in the account without exception. Any privilege granted to PUBLIC is effectively visible to the entire organization. Because of this, PUBLIC should be treated as a zero-trust boundary.

*Best practices:* Grant only to objects that are intentionally and permanently public to every person in the account. This is almost never the right choice for actual data tables.

**ORGADMIN**
A special role for managing the Snowflake Organization — creating and listing accounts, enabling replication between accounts, and monitoring organization-wide usage. It operates at a level above the individual account.

*Best practices:* Only assign to senior platform administrators responsible for the full Snowflake estate. Never used for workload operations.

### The Critical "Orphan Object" Problem

When a custom role creates an object (e.g., a table) without that custom role being granted to SYSADMIN, SYSADMIN and ACCOUNTADMIN cannot manage that object. The object is effectively orphaned from administrative control. **This is one of the most common RBAC mistakes in production Snowflake environments** and a frequent exam topic. The fix is always: grant every custom role up to SYSADMIN.

---

## 4. Database Roles

### What Database Roles Are

Database roles are role objects that exist and operate **within the scope of a single database only**. They are an evolution of Snowflake's access control model that addresses the scalability problem of managing too many access roles at the account level.

Unlike account roles, database roles cannot be directly activated in a session — a user cannot `USE ROLE my_db.my_db_role`. Instead, database roles must be granted to an account role, which the user then activates. The database role's privileges flow up through that grant.

### Database Roles vs. Account Roles — Key Distinctions

**Scope:** An account role can grant privileges on any object anywhere in the account. A database role can only grant privileges on objects within its home database.

**Session Activation:** Account roles can be set with `USE ROLE` or specified at connection time. Database roles cannot be activated directly in a session.

**Visibility:** Account roles appear in Snowsight's role switcher dropdown. Database roles do not — this is by design to reduce role clutter for end users.

**Grant Hierarchy:** Account roles can be granted to users, to other account roles, or (in one direction) to other account roles. Database roles can only be granted to account roles or to other database roles within the same database. Account roles can never be granted to database roles.

**Cross-Database:** An account role can hold privileges on tables in Database A, Database B, and Database C simultaneously. A database role is permanently bounded to its home database.

### Why Database Roles Matter for Governance

Before database roles, implementing the access role pattern at scale meant creating dozens or hundreds of account-level access roles (e.g., `DB_SCHEMA1_READ`, `DB_SCHEMA2_READ`, `DB_SCHEMA3_WRITE`). These cluttered the account role namespace and confused end users who saw them in their role dropdown.

With database roles, access roles live *inside* the database they govern. They are invisible to users at the account level. The database itself becomes a self-contained, governable unit — its roles, its privileges, its access policies all travel together. This makes database roles the preferred approach for implementing access roles in a modern Snowflake architecture.

**An additional governance benefit:** When a database is shared via Snowflake Data Sharing, the provider can grant a database role to the share. The consumer account then receives that database role with all its associated privileges cleanly packaged — no need to manually enumerate which tables to share.

---

## 5. Functional Roles vs. Access Roles

This is the recommended RBAC design pattern from Snowflake's own documentation. It separates *what data can be accessed* (access roles) from *who the user is in the business* (functional roles).

### Access Roles — The Building Blocks

Access roles represent a **narrow, specific set of privileges on a specific object or schema**. They answer the question: "What can be done with this particular dataset?"

A well-designed access role covers exactly one database object boundary (a schema or a table group) at exactly one permission level (read-only, read-write, or admin). Their names should be self-describing — for example, `FINANCE_DB.ACCOUNTS_SCHEMA_READ` or `FINANCE_DB.ACCOUNTS_SCHEMA_RW`.

Access roles are implemented as **database roles** in a modern Snowflake architecture to avoid account-level role sprawl. They are never granted directly to users — that is the job of functional roles.

### Functional Roles — The Business Layer

Functional roles represent a **job function or persona** in the organization. They answer the question: "What does this type of user need to do their job?"

A data analyst, a data engineer, a compliance officer, and a data scientist all have different access needs across potentially many databases and schemas. Each of these job functions gets its own functional role, which is an account role. The functional role is then populated by granting the appropriate access roles into it.

The key insight is that the mapping between access roles and functional roles is the central governance decision — it is where the organization defines who can see what. Updating that mapping is the only change needed when job responsibilities shift.

### Service Roles — The Automation Layer

Service roles are a third category — functionally similar to functional roles but designed for **non-human principals**: ETL pipelines, BI tool connectors, CI/CD systems, and API service accounts. They follow the same structure as functional roles but with an even stricter least-privilege lens, since automated systems should never have more access than the single task they perform.

### The Full Architecture Stack

```
USERS (humans and services)
    ↓  granted to
FUNCTIONAL / SERVICE ROLES  (what job does this person/system have?)
    ↓  granted to
ACCESS ROLES / DATABASE ROLES  (what objects can they touch, and how?)
    ↓  hold privileges on
SECURABLE OBJECTS  (databases, schemas, tables, views, warehouses)
```

All functional and service roles are granted upward to SYSADMIN to close the orphan object gap.

### Managed Access Schemas — Centralizing the Grant Decision

A **Managed Access Schema** is created with the `WITH MANAGED ACCESS` option. In this mode, the normal DAC rule is suspended: even though object creators own their objects, they cannot grant privileges on those objects to other roles. Only the schema owner (or ACCOUNTADMIN / SECURITYADMIN) can make grant decisions within the schema.

This is essential for governance at scale — it ensures that a data engineer who creates a new table cannot accidentally share it with the wrong roles. All access grants within a managed schema must flow through the designated governance role.

---

## 6. Secondary Roles

### The Problem Secondary Roles Solve

In a standard Snowflake session, a user has exactly one active role — the **primary role**. If a user needs privileges from two different roles simultaneously (for example, they need to read from a table owned by Role A and write to a table owned by Role B), they would traditionally have to switch roles between queries, breaking their workflow.

**Secondary roles** allow one or more additional roles to be active in a session simultaneously alongside the primary role, so the user has access to the union of all those roles' privileges in a single session.

### How Primary and Secondary Roles Divide Responsibility

The division of responsibility between primary and secondary roles is precise and important for the exam:

**Primary role** governs **object creation**. Whenever a `CREATE` statement is executed, the object is owned by the primary role — and only the primary role. Secondary roles play no part in ownership assignment.

**Secondary roles** contribute their privileges to **all other operations** — `SELECT`, `INSERT`, `UPDATE`, `DELETE`, DDL on existing objects, calling procedures, etc. Snowflake evaluates the combined privilege set of the primary role plus all active secondary roles for these operations.

### Secondary Role Modes

A user can set secondary roles to `ALL` (every role currently granted to that user becomes active as a secondary role) or to `NONE` (the default — only the primary role is active). As of early 2025, Snowflake changed the default behavior for new accounts so that `SECONDARY_ROLES = ALL` is active by default, making multi-role workflows more seamless.

### Why This Matters for Governance

Secondary roles introduce complexity in auditing. When a query runs, the `QUERY_HISTORY` view records the primary role but not necessarily the full secondary role context. Organizations with strict compliance requirements should carefully consider whether to allow `SECONDARY_ROLES = ALL` or to design their role hierarchy so that a single primary role is sufficient for each user's needs.

---

## 7. Data Access & Storage Integrations

### Why Storage Integrations Exist

When Snowflake needs to read from or write to external cloud storage (AWS S3, Azure Blob Storage, GCP Cloud Storage), it needs credentials to authenticate with that storage provider. The naive approach — embedding access keys or storage account keys directly in stage definitions — creates serious security risks: keys can expire, leak, or be hard to rotate across many stage definitions.

A **storage integration** is Snowflake's solution. It is an account-level object that establishes a **trust relationship** between Snowflake and a cloud provider using the provider's native identity mechanism (AWS IAM roles, Azure service principals, GCP service accounts). The credentials never need to touch Snowflake configurations — instead, Snowflake is *trusted* by the cloud storage, and the integration object holds the configuration of that trust.

### How the Trust is Established

When a storage integration is created, Snowflake generates an identity on its side — an IAM entity (in AWS terms, an IAM user ARN and an External ID). The Snowflake administrator retrieves these generated values by describing the integration object, then goes to the cloud provider's console and configures the cloud storage's trust policy to allow that Snowflake-generated identity to assume a specific role. From that point on, Snowflake can authenticate as that assumed role without any stored secrets.

### Security Considerations for Storage Integrations

**Allowed and Blocked Locations** are the most important security controls on a storage integration. `STORAGE_ALLOWED_LOCATIONS` restricts which storage paths the integration can access — applying the principle of least privilege. `STORAGE_BLOCKED_LOCATIONS` explicitly denies specific sub-paths even if they fall within an allowed location. This two-layer control prevents a misconfiguration from accidentally exposing sensitive prefixes within an allowed bucket.

**Privilege separation:** Creating a storage integration requires `ACCOUNTADMIN`. But using it — creating external stages that reference it — only requires `USAGE` on the integration. This allows ACCOUNTADMIN to set up and control the trust relationship, while operational roles (like a data engineering role) can create stages and load data without needing elevated privileges.

---

## 8. Secure Views

### The Problem with Regular Views for Security

A regular Snowflake view is transparent — any user with `SELECT` on the view can inspect the view's definition through `SHOW VIEWS` or by querying the `INFORMATION_SCHEMA.VIEWS` table. If a view is designed to filter rows or mask columns based on `CURRENT_ROLE()`, an unauthorized user can read the view's SQL and understand exactly what data is being hidden from them — and potentially use that structural knowledge maliciously.

There is also a subtler risk: Snowflake's query optimizer can sometimes use access patterns (like which partitions it skips for a given user) to inadvertently reveal information about the filtered data. This is called a **side-channel attack**, and regular views are vulnerable to it.

### What Secure Views Provide

A **secure view** addresses both problems:

**DDL confidentiality:** The view definition is hidden from all users except the view's owner role. Non-owners cannot inspect the SQL logic through any metadata query or information schema view.

**Optimizer bypass:** Snowflake intentionally restricts certain query optimizer shortcuts when evaluating a secure view query. The optimizer is forced to evaluate the full view logic rather than taking shortcuts that could leak structural information. This comes at a slight performance cost — an acceptable trade-off for security-sensitive use cases.

### When to Always Use Secure Views

- Any view shared through Snowflake Data Sharing or the Marketplace, because consumers of a shared view can inspect regular view definitions
- Any view that uses `CURRENT_ROLE()`, `CURRENT_USER()`, or similar context functions as part of its filtering logic, where exposing the logic would reveal what is being protected
- Any view over sensitive tables where the view's query structure would itself reveal information about the data's shape or classification

### Secure Views and Data Sharing

Secure views are a requirement when sharing data externally via Snowflake's Data Sharing mechanism. When a data provider creates a share and includes a view, Snowflake enforces that the view must be a *secure* view. This prevents consumers from reverse-engineering the provider's underlying table structure or security logic through view introspection.

---

## 9. Column-Level Security

Column-level security (CLS) is an **Enterprise Edition and higher** feature. It controls what users see within individual columns of a table, independent of row-level filtering.

### 9.1 Dynamic Data Masking (DDM)

#### Core Concept

Dynamic Data Masking uses a **masking policy** — a schema-level object containing SQL logic — to transform column values at **query runtime**. The critical point is that the underlying data in storage is never altered. The transformation happens in the query engine at the moment the data is read. Different users running the same query against the same table at the same time may see completely different values, all determined by the masking policy's conditions.

#### The Masking Policy as a Schema-Level Object

The masking policy lives at the schema level, meaning it is a first-class database object with its own lifecycle — it can be created, altered, dropped, and its assignments can be queried through metadata views. This design has important implications:

One masking policy can be **applied to multiple columns across multiple tables and views**, creating a single point of control for a governance decision (e.g., "all email columns should be masked the same way for non-HR roles").

The policy is attached to a specific column with an `ALTER TABLE ... MODIFY COLUMN ... SET MASKING POLICY` statement. Only one masking policy can be attached to a column at a time.

#### How the Policy Conditions Are Evaluated

The masking policy is a SQL expression — typically a `CASE` statement — that receives the column's value as an argument and returns a transformed value (or the original value) depending on **execution context**. The most commonly used context functions are:

- `CURRENT_ROLE()` — returns the active primary role of the session
- `IS_ROLE_IN_SESSION(role_name)` — returns TRUE if the named role is active as either the primary or any secondary role
- `CURRENT_USER()` — returns the logged-in username
- `INVOKER_ROLE()` — used inside views to capture the role of whoever is calling the view, not the view owner
- `INVOKER_SHARE()` — returns the share name when the query originates from a data sharing consumer

#### Where Masking is Applied

Masking applies **everywhere the column appears in a query** — not just in the SELECT list. If a masked column is used in a WHERE clause, a JOIN condition, an ORDER BY, or a GROUP BY, the masking policy is evaluated at all of those locations. This prevents a user from inferring real values by testing query conditions.

#### Nested Masking Policies

Snowflake supports having a masking policy on a base table column AND a different masking policy on a view's column over that same table. The policies are additive — the view policy applies on top of the table policy. This allows fine-grained governance at both the storage and presentation layers.

#### Who Should Manage Masking Policies

Snowflake recommends creating a dedicated `MASKING_ADMIN` custom role rather than using ACCOUNTADMIN or SECURITYADMIN for daily masking policy work. This role is granted the ability to create and apply masking policies but not the ability to change broad account settings. Typically, the data privacy officer or security engineer holds this role.

---

### 9.2 External Tokenization

#### Core Concept

External tokenization is a more radical form of data protection than DDM. With DDM, the raw sensitive value *does exist* in Snowflake's storage — it is just obscured at query time for unauthorized users. With external tokenization, the **raw sensitive value is never stored in Snowflake at all**.

Instead, the data is **tokenized by a third-party tokenization service before it is loaded** into Snowflake. What enters Snowflake is an opaque, cryptographically secure token — a meaningless string that cannot be reversed without the tokenization service. At query time, authorized users trigger the detokenization by invoking an external function that calls back to the tokenization service, which returns the real value only if the caller is authorized.

#### The Mechanism

External tokenization uses **masking policies with external functions** as its delivery mechanism. The masking policy looks at the caller's role, and if authorized, calls the external function (which is linked to the tokenization service's API). The external function returns the real value; if not authorized, the policy returns the token as-is — still opaque and meaningless to the unauthorized user.

#### DDM vs. External Tokenization — When to Use Which

The choice comes down to the **threat model** and **regulatory mandate**:

Use **DDM** when the requirement is to prevent unauthorized users from seeing sensitive values in query results, but where storing plaintext in Snowflake's encrypted storage is acceptable from a compliance standpoint. DDM is simpler to operate, has no external dependency, and works fully with Snowflake Data Sharing.

Use **External Tokenization** when regulatory requirements explicitly mandate that the sensitive value must never exist in plaintext in any cloud storage — not even encrypted. This is common for PCI-DSS cardholder data (Primary Account Numbers), where the standard requires that full PANs must be stored in tokenized or truncated form.

**Critical Limitation:** External tokenization cannot be used across Snowflake Data Sharing because external functions cannot be invoked in the context of a share. If you need to share data containing tokenized columns, only the token (not the real value) would be visible to the consumer — which may or may not be acceptable depending on your use case.

---

### 9.3 Conditional Masking

Conditional masking is an extension of DDM where the masking policy's decision depends not just on the caller's role, but also on **the value of another column in the same row**.

This is most useful when data has a consent or classification flag built into it. For example, if a customer table has an `opt_out` flag indicating the customer has withdrawn consent for their data to be used, a conditional masking policy can inspect that flag and apply full masking to the email column regardless of the caller's role.

Conceptually, conditional masking moves the masking logic from being purely role-based to being **data-driven**. The column being masked and the conditional column must reside in the same table or view — you cannot reference columns from other tables inside a masking policy.

---

## 10. Row-Level Security – Row Access Policies

### Core Concept

A **Row Access Policy (RAP)** controls which *rows* a user can see when querying a table or view. While masking policies transform column values, row access policies silently **remove rows** from the result set that the querying user is not authorized to see. From the user's perspective, those rows do not exist — they receive no error, no indication that rows were filtered, just a result set containing only their permitted data.

### How the Policy Logic Works

A row access policy is a schema-level object containing a SQL expression that accepts one or more column values from the table as arguments and **returns a Boolean**. TRUE means the row is visible; FALSE means the row is filtered out. The expression is evaluated for every row in the table at query time.

The most common and scalable pattern is the **mapping table approach**: a separate administrative table maps role names (or user names) to the data segments they are permitted to see (regions, business units, cost centers, etc.). The policy expression queries this mapping table, checking whether the current role has an entry for the row's value. This decouples the access decision from the policy code — administrators update the mapping table to change access without ever touching the policy definition itself.

### Key Rules and Behaviors

**Evaluation order with masking policies:** When both a row access policy and masking policies are in effect on the same table, Snowflake always evaluates the **row access policy first**. Rows that are filtered out are never processed by masking policies — which makes sense, since there is no need to mask a row that is invisible.

**One policy per table:** A table or view can have only one row access policy attached at a time. You cannot stack multiple row access policies on the same object. The policy must encapsulate all row-filtering logic for that table.

**Column restriction:** A column that is referenced in a row access policy's signature (its arguments) cannot simultaneously be covered by a masking policy. The two policy types are mutually exclusive for any given column. Plan your policy architecture accordingly.

**Cloning behavior:** When a table with a row access policy is cloned, the clone references the same policy object as the source. The policy is not duplicated — both the source and clone point to the same policy definition, which means any change to the policy affects both. This is important to understand for dev/test cloning workflows.

**Performance impact:** Row access policies add overhead to every query against the protected table because Snowflake must evaluate the policy expression for every scanned row. For large tables with complex policy expressions — especially those that join to a mapping table — this can be significant. Always test with production-scale data volumes before rolling out a row access policy broadly.

**Applicability:** Row access policies can be applied to standard tables, views, materialized views, dynamic tables, external tables, and Iceberg tables. They are a universal row-filtering mechanism across Snowflake's object types.

---

## 11. Aggregate Policies

### Core Concept

An **Aggregate Policy** restricts users to seeing only **aggregated results** rather than individual row-level data. It enforces a minimum group size on query results — if a query returns a group with fewer rows than the configured minimum, that group is silently excluded from the result.

### The Problem It Solves

There are scenarios where an organization needs to provide analytical access to sensitive data (survey responses, clinical trial results, demographic statistics, salary bands) but cannot allow individual record access. Standard masking and row filtering don't fully solve this because a clever analyst could still re-identify individuals by filtering down to very small groups.

An aggregate policy solves this by making it structurally impossible to retrieve individual rows — the query engine enforces that results must represent a minimum number of underlying records, preventing the "group of one" re-identification attack.

### Important Behavioral Detail

When an aggregate policy is in effect and a query produces a group that falls below the minimum size, that group is **silently dropped from the result** — not replaced with NULL or a masked value, just absent. The user receives no indication that data was suppressed. This is intentional: even knowing that a certain combination has fewer than N records is itself a data disclosure.

### Typical Use Cases

Aggregate policies are most relevant for external data sharing (publishing statistical datasets to partners or the Marketplace without exposing source records), privacy-preserving analytics over HR or health data, and compliance with differential privacy requirements in research contexts.

---

## 12. Projection Policies

### Core Concept

A **Projection Policy** controls whether a column can appear in the **output** of a SELECT statement. It does not prevent the column from being used in filtering, joining, or grouping — only from being projected into the result set.

### How This Differs from Masking

With a masking policy, a restricted user sees a column in their results but gets a transformed or obscured value (asterisks, NULL, partial data). With a projection policy, the column simply **cannot appear in SELECT** at all for restricted users. Attempting to project a projection-policy-restricted column results in an error.

However, that same column can still be used in a `WHERE` clause, a `JOIN ON` condition, or a `GROUP BY` clause. This means users can filter or join on the sensitive field (e.g., matching on SSN to find a customer) without the raw SSN value ever appearing in their query results.

### The Security Use Case

Projection policies are ideal when the goal is to enable **functional use of a sensitive identifier** (for joining, matching, deduplication) while ensuring the identifier itself is never exposed in results. This is a common requirement for PCI-scoped columns (cardholder numbers used as join keys in downstream analytics) and certain GDPR scenarios where indirect identifiers must not be returned in output.

---

## 13. How All Four Policies Interact

Understanding how these policies layer together is essential for the exam.

### Evaluation Order

Snowflake applies these policies in a specific sequence:

1. **Row Access Policy** — evaluated first; rows that fail the policy are removed entirely from consideration
2. **Aggregation Policy** — applied next; enforces aggregate-only constraints on the remaining rows
3. **Projection Policy** — controls which columns can appear in the SELECT output
4. **Masking Policy** — applied last; transforms the visible column values for the user

The ordering ensures that the most restrictive, broadest filters (row removal) run before more targeted transformations (column masking).

### What Each Policy Controls

| Policy | What It Filters | User Sees |
|---|---|---|
| **Row Access Policy** | Which rows are returned | Only permitted rows; filtered rows silently absent |
| **Aggregation Policy** | Whether individual rows can appear | Aggregate results only; small groups silently dropped |
| **Projection Policy** | Whether column appears in SELECT | Column blocked from output; usable in WHERE/JOIN |
| **Masking Policy** | What value a column shows | Transformed/obscured value; column always appears |

### Important Constraint

A single column cannot simultaneously be the argument of a **Row Access Policy** and subject to a **Masking Policy**. The two policies are mutually exclusive for any given column. This is a hard platform constraint, not just a best practice.

---

## 14. Data Lineage & Object Dependencies

### Why Lineage Matters for Governance

Data lineage answers the questions that regulators, auditors, and data stewards most frequently ask: *Where did this data come from? What transformations did it undergo? Who accessed it? What would break if we changed this table?* Without lineage, compliance is a manual, error-prone exercise.

Snowflake provides **native lineage** as part of its platform — no third-party tool is required for fundamental lineage tracking.

### The Three Lineage Tools

**Snowsight Lineage Tab** is the visual, interactive lineage graph available in the Snowflake UI for Enterprise Edition and above. It shows upstream and downstream dependencies for tables, views, dynamic tables, tasks, and ML objects. It is the best tool for stakeholder-facing lineage reviews and for quickly understanding what a given object depends on.

**`GET_LINEAGE` function** provides programmatic access to the same lineage data, returning a structured result that can be queried, filtered, and used in custom dashboards or automation. It is limited to a maximum depth of 5 levels in the dependency graph, which covers the vast majority of real-world pipelines.

**`OBJECT_DEPENDENCIES` view** in `ACCOUNT_USAGE` provides a simpler, table-level dependency map — which objects reference which other objects. It is lightweight and does not require the full lineage graph computation. Best for quick automated checks like "which views will break if I alter this table?"

### ACCESS_HISTORY — The Column-Level Audit Trail

The `ACCESS_HISTORY` view in `SNOWFLAKE.ACCOUNT_USAGE` is the gold standard for compliance auditing. It records every query that accessed a table or view, including **which specific columns were read or written**, the user, the role, the timestamp, and the query ID. This column-level granularity is what distinguishes it from `QUERY_HISTORY`, which only records the query text.

For PCI-DSS audits (proving who accessed cardholder data columns), HIPAA audits (proving who accessed PHI), and GDPR data subject access requests (tracing who has touched a specific individual's data), `ACCESS_HISTORY` is the authoritative source. Data is retained for one year in the `ACCOUNT_USAGE` schema.

### Tag Lineage vs. Data Lineage

These two concepts are often confused on the exam:

**Data lineage** tracks how data flows between tables, views, and other objects through SQL transformations — it is about the movement and transformation of the data itself.

**Tag lineage** (sometimes called tag propagation) describes how object tags applied to a parent object automatically propagate to child objects (database tags flow to schemas, schema tags flow to tables, table tags flow to columns). Tag lineage is about metadata inheritance within the object hierarchy, not data movement.

---

## 15. Object Tagging & Data Classification

### What Object Tags Are

**Object tags** are key-value metadata labels that can be applied to Snowflake objects — databases, schemas, tables, columns, warehouses, users, roles, and other objects. A tag is itself a schema-level object with a defined name and an optional set of allowed values. Tags are an **Enterprise Edition+** feature.

Tags serve as the **connective tissue of governance** — they link business meaning (sensitivity level, data classification, regulatory scope) to technical objects, enabling automated policy enforcement and compliance reporting.

### Tag Propagation and Inheritance

When a tag is applied to a database, it automatically propagates to all schemas within that database, and from those schemas to all tables and views, and from tables to all their columns. This propagation means you can classify an entire data domain with a single tagging action rather than tagging every column individually.

This inheritance is not retroactive — it applies to existing child objects at the time of tagging and to new child objects created after the tag is applied. Importantly, a child object can override its inherited tag value with its own explicit tag assignment.

### Tag-Based Masking — The Automation Layer

The most powerful use of object tags in governance is **tag-based masking**: associating a masking policy with a specific tag-value pair. When that association is configured, any column tagged with that tag-value combination automatically receives the masking policy — without any per-column `ALTER TABLE` statement needed.

This transforms governance from a reactive, manual process (remember to tag and mask every new sensitive column) to a proactive, automated one: define the classification taxonomy, define the masking policies, associate them with tag values, and then governance is enforced automatically as new columns are classified.

### Snowflake Data Classification

Snowflake includes an **AI-powered classification engine** that scans table columns and identifies likely sensitive data categories — `NAME`, `EMAIL`, `PHONE_NUMBER`, `SSN`, `CREDIT_CARD`, `IP_ADDRESS`, `DATE_OF_BIRTH`, `POSTAL_CODE`, and others. Classification works by analyzing column names, sample data patterns, and statistical distributions.

Classification can be run on-demand against individual tables or applied to an entire schema. After classification runs, administrators review the recommendations and accept those that are accurate. Accepting a recommendation automatically applies the corresponding system tag to the column.

When combined with tag-based masking, classification creates an end-to-end automated PII governance pipeline: classify new tables → tags applied → masking policies automatically enforced — with minimal manual intervention.

### Using Tags for Compliance Reporting

Tags are queryable through `SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES`. This makes it possible to generate compliance reports like:
- "Show me all columns in the account tagged as `sensitivity = RESTRICTED`"
- "Show me all tables in the EU region tagged with `data_residency = EU`"
- "Show me all objects in scope for PCI by their `pci_scope = TRUE` tag"

These queries replace what would otherwise be manual spreadsheet-based data inventories — they are always current, always accurate, and directly tied to the actual Snowflake objects.

---

## 16. Compliance & Snowflake Editions

### Edition Overview — A Governance Lens

Snowflake's four editions form a security and governance escalation ladder. Each edition includes everything below it.

**Standard Edition** provides the foundational security features available to all accounts: TLS 1.2+ encryption in transit, AES-256 encryption at rest, object-level RBAC, one day of Time Travel, and seven days of Fail-Safe. It lacks all column-level security, row-level security, object tagging, and compliance certifications. It is appropriate for non-sensitive internal analytics.

**Enterprise Edition** adds the full governance toolkit: Dynamic Data Masking, External Tokenization, Row Access Policies, Aggregate and Projection Policies, Object Tagging, Data Classification, Data Lineage, and Time Travel up to 90 days. This is the minimum edition required for any organization with PII governance obligations, though not yet with formal regulatory compliance mandates. Enterprise is the most common production edition for data-mature organizations.

**Business Critical Edition** (formerly "Enterprise for Sensitive Data" or ESD) is the minimum edition for regulated workloads. It adds: AWS/Azure/GCP PrivateLink (private connectivity that never touches the public internet), Database Failover/Failback for disaster recovery, HIPAA/HITRUST eligibility (with a signed BAA), PCI-DSS compliance support, Tri-Secret Secure (customer-managed encryption keys), and additional compliance certifications. This edition is required for healthcare, financial services, and government workloads handling regulated data.

**Virtual Private Snowflake (VPS)** provides the highest isolation tier. All Business Critical features are included, but the entire Snowflake environment — compute, storage, and metadata — runs on **hardware dedicated to a single customer with no sharing with other Snowflake tenants whatsoever**. VPS is reserved for organizations with the most stringent isolation mandates: large financial institutions, intelligence-adjacent workloads, and enterprises where even the perception of shared infrastructure is unacceptable.

### Feature-to-Edition Mapping (Governance Focus)

| Governance Feature | Standard | Enterprise | Business Critical | VPS |
|---|---|---|---|---|
| Basic RBAC & DAC | ✅ | ✅ | ✅ | ✅ |
| Secure Views | ✅ | ✅ | ✅ | ✅ |
| Storage Integrations | ✅ | ✅ | ✅ | ✅ |
| Dynamic Data Masking | ❌ | ✅ | ✅ | ✅ |
| External Tokenization | ❌ | ✅ | ✅ | ✅ |
| Row Access Policies | ❌ | ✅ | ✅ | ✅ |
| Aggregate Policies | ❌ | ✅ | ✅ | ✅ |
| Projection Policies | ❌ | ✅ | ✅ | ✅ |
| Object Tagging | ❌ | ✅ | ✅ | ✅ |
| Data Classification | ❌ | ✅ | ✅ | ✅ |
| Data Lineage (Snowsight) | ❌ | ✅ | ✅ | ✅ |
| Time Travel up to 90 days | ❌ | ✅ | ✅ | ✅ |
| Multi-Cluster Warehouses | ❌ | ✅ | ✅ | ✅ |
| PrivateLink | ❌ | ❌ | ✅ | ✅ |
| DB Failover / Failback | ❌ | ❌ | ✅ | ✅ |
| HIPAA / HITRUST support | ❌ | ❌ | ✅ | ✅ |
| PCI-DSS compliance support | ❌ | ❌ | ✅ | ✅ |
| Tri-Secret Secure (BYOK) | ❌ | ❌ | ✅ | ✅ |
| FedRAMP | ❌ | ❌ | ✅ | ✅ |
| Dedicated hardware (no sharing) | ❌ | ❌ | ❌ | ✅ |

---

### PCI-DSS (Payment Card Industry Data Security Standard)

PCI-DSS is a prescriptive set of security requirements that any organization **storing, processing, or transmitting cardholder data** must satisfy. Cardholder data includes the Primary Account Number (PAN — the 16-digit card number), cardholder name, expiration date, and service code. Sensitive Authentication Data (SAD) — CVV, PIN blocks, full magnetic stripe — has even stricter handling rules.

**Minimum Snowflake Edition:** Business Critical, for PrivateLink, enhanced encryption, and Snowflake's Attestation of Compliance from a Qualified Security Assessor (QSA).

**Key requirements translated to Snowflake controls:**

*Requirement 3 (Protect stored cardholder data):* PAN must never be stored in plaintext. The appropriate Snowflake control is **External Tokenization** — cardholder data is tokenized before entering Snowflake, so only tokens exist in storage. DDM alone is insufficient here because the plaintext PAN still exists in Snowflake's storage layer.

*Requirement 4 (Encrypt transmission):* Snowflake enforces TLS 1.2+ for all connections. Combined with PrivateLink (which eliminates public internet traversal entirely), transmission security is robust.

*Requirement 7 (Restrict access by business need to know):* Strict RBAC with the access role / functional role pattern. Compliance officer roles see full PANs; analyst roles see only the last four digits via masking; ETL roles see only tokens.

*Requirement 8 (Identify and authenticate access):* Multi-Factor Authentication (MFA) must be enforced for all users. Network Policies restrict access to known, controlled IP ranges. Service accounts must use key-pair authentication rather than passwords.

*Requirement 10 (Track and monitor all access):* `ACCESS_HISTORY` provides the column-level audit trail required by PCI-DSS for demonstrating who accessed cardholder data columns and when. This view must be regularly reviewed and the data retained.

*Requirement 12 (Information security policy):* Tri-Secret Secure provides the highest level of encryption control — combined with a documented key management process, this satisfies the encryption key management requirements.

---

### PII and PHI – Privacy and Healthcare Compliance

**PII (Personally Identifiable Information)** is any data that can identify a specific individual, either directly (name, SSN, passport number) or indirectly (a combination of age, ZIP code, and gender that narrows down to a single person). The definition and handling requirements for PII vary by regulation: GDPR (EU), CCPA (California), PIPEDA (Canada), LGPD (Brazil), etc.

**PHI (Protected Health Information)** is PII in the context of healthcare — any information about a person's health condition, treatment, or payment for healthcare services, when that information is created or held by a covered entity or business associate. PHI is governed by HIPAA in the United States and HITRUST CSF.

#### HIPAA Requirements in Snowflake

**The BAA is non-negotiable:** Before any PHI can be stored in Snowflake, a **Business Associate Agreement (BAA)** must be signed between the healthcare organization and Snowflake Inc. The BAA is a legal contract that establishes Snowflake as a "business associate" under HIPAA and specifies each party's obligations for protecting PHI. The existence of technical security features does not make a Snowflake account HIPAA-compliant — only a signed BAA in conjunction with those features does.

**Minimum Edition:** Business Critical. The BAA is only available on Business Critical and VPS accounts.

**Required technical controls under HIPAA:**
- Encryption at rest and in transit (Snowflake provides both; Tri-Secret Secure for maximum control)
- Access controls limiting PHI access to authorized individuals (RBAC with the functional/access role pattern)
- Audit logging of all PHI access (`ACCESS_HISTORY` for column-level PHI access tracking)
- Emergency access procedures (Failover Groups for disaster recovery)
- Person authentication (MFA, SSO via SAML or OAuth)
- Network isolation (PrivateLink to prevent PHI from traversing the public internet)

#### GDPR Considerations

**Data residency:** EU personal data must typically remain within the EU. Snowflake accounts provisioned in EU regions (AWS `eu-west-1`, Azure `westeurope`, GCP `europe-west4`) ensure data never leaves the EU boundary.

**Data minimization:** The principle that only the minimum necessary personal data should be processed. Masking policies enforce this at the query layer — analytics teams work with masked data, never with the full PII unless their role genuinely requires it.

**Right to erasure:** GDPR grants individuals the "right to be forgotten." In Snowflake, this is operationally complex because Time Travel retains historical versions of data. Setting `DATA_RETENTION_TIME_IN_DAYS = 0` for tables containing personal data, combined with database-level deletion, reduces (but does not eliminate) the Time Travel window. Fail-Safe (seven days, not configurable by customers) is a further consideration.

**Data subject access requests:** `ACCESS_HISTORY` can demonstrate to Data Protection Authorities which systems and users have touched a specific individual's data — important for demonstrating accountability under GDPR's accountability principle.

---

### Tri-Secret Secure (Bring Your Own Key – BYOK)

**Tri-Secret Secure** is a Business Critical and VPS feature that gives customers direct control over the encryption of their Snowflake data. Standard Snowflake encryption uses Snowflake-managed keys — Snowflake can, in theory, decrypt customer data. Tri-Secret Secure changes this.

The concept is a **composite encryption key** formed from two components: a Snowflake-managed key component and a **customer-managed key** stored in the customer's own key management service (AWS KMS, Azure Key Vault, or GCP Cloud KMS). Neither component alone can decrypt the data. Both must be present.

The governance implication is significant: if a customer revokes or disables their key in their KMS, Snowflake loses the ability to decrypt the data — even Snowflake's own engineers cannot access it. This is the "Tri" in Tri-Secret Secure, referring to the three parties whose cooperation is required: Snowflake, the customer's KMS, and the customer's own authorization.

This feature is particularly important for organizations subject to regulations requiring **customer control over encryption keys**, such as certain financial services mandates and government security frameworks.

---

## 17. Snowflake Horizon – Unified Governance

**Snowflake Horizon** is Snowflake's umbrella brand for its built-in governance, security, privacy, compliance, and interoperability capabilities. Rather than requiring a separate external governance tool, Horizon embeds governance directly into the data platform.

### The Five Pillars of Horizon

**Compliance** encompasses the audit and observability capabilities: `ACCESS_HISTORY` for column-level access auditing, `OBJECT_DEPENDENCIES` for impact analysis, regulatory-ready views in `ACCOUNT_USAGE`, and the Lineage UI in Snowsight. The goal is that compliance teams can answer any regulatory question directly from Snowflake metadata without assembling data from multiple systems.

**Privacy** encompasses the active data protection policies: Dynamic Data Masking, External Tokenization, Conditional Masking, Row Access Policies, Aggregate Policies, and Projection Policies. Together, these form a layered defense-in-depth model where even authorized-but-limited users see only the minimum data necessary for their role.

**Security** covers the perimeter and identity controls: RBAC, Network Policies, MFA, SSO (SAML/OAuth), PrivateLink, and Tri-Secret Secure. These protect access to the platform itself, not just the data within it.

**Interoperability** addresses Snowflake's ability to govern data that lives outside Snowflake's own storage — Apache Iceberg tables on customer-managed object storage, Open Catalog for cross-engine catalog sharing, and enforcement of Snowflake governance policies (masking, row access) on Iceberg tables queried from Apache Spark.

**Data Quality** is the newer addition: Data Metric Functions that can monitor freshness, completeness, uniqueness, and custom business rules on live data, surfacing data quality scores through `ACCOUNT_USAGE` views.

### Horizon's Place in the Exam

The Advanced certification increasingly emphasizes Snowflake Horizon as the holistic framework. Questions may present a scenario (e.g., "a healthcare organization needs to share clinical data with research partners while ensuring PHI is never exposed") and ask which combination of Horizon capabilities addresses it. The answer almost always involves multiple layers: edition selection (Business Critical), RBAC design, masking policies (DDM or tokenization), row access policies, secure views for sharing, and `ACCESS_HISTORY` for auditing.

---

## 18. Exam Tips & Common Gotchas

### ⚡ High-Yield Conceptual Points

**Inheritance flows upward, not downward.** When Role A is granted to Role B, Role B inherits Role A's privileges. The child pushes privileges to the parent. The terminology is counterintuitive — "granting to" creates an upward inheritance relationship.

**Account roles can never be granted to database roles.** The constraint is absolute. Database roles can only be granted to account roles, moving privileges upward from database scope to account scope.

**Database roles cannot be activated in a session.** They are activated indirectly by granting them to an account role. This is both a limitation and a feature — it keeps the user-facing role namespace clean.

**Object ownership always belongs to the primary role.** No matter how many secondary roles are active, `CREATE` statements assign ownership to the primary role only. Secondary roles only contribute privileges for read/write/DML operations.

**Row Access Policy evaluates before Masking Policies.** The row filtering layer runs first; then masking is applied to the remaining visible rows.

**A column can be in either a RAP signature or a masking policy — never both.** This is a hard platform constraint, not just a design preference.

**External Tokenization cannot be used with Data Sharing.** External functions cannot be invoked in the context of a Snowflake share. Only DDM works in shared data scenarios.

**DDM does not change the stored data.** The underlying value is always the plaintext in storage. Masking is purely a query-time transformation. This is fundamentally different from External Tokenization, where the plaintext never enters Snowflake storage.

**HIPAA compliance requires a signed BAA — features alone are not enough.** The BAA is a legal contract. Without it, storing PHI in Snowflake violates HIPAA regardless of what technical controls are in place.

**Business Critical is the minimum edition for PrivateLink, HIPAA, PCI-DSS, and Tri-Secret Secure.** Enterprise Edition has the full governance toolkit (masking, row access, tagging) but lacks the compliance certifications and private connectivity required for regulated workloads.

**Managed Access Schemas remove the object owner's ability to grant privileges.** Only the schema owner or ACCOUNTADMIN/SECURITYADMIN can make grant decisions within a managed access schema. This is the Snowflake mechanism for centralized access governance.

**PUBLIC is granted to every user automatically.** Any privilege on PUBLIC is a privilege for the entire organization. This is almost never appropriate for real data objects.

**FUTURE GRANTS are essential for preventing governance gaps.** Without them, every new table or column requires a manual re-grant. Governance that relies on humans remembering to grant is governance that will eventually fail.

**Aggregate policy suppresses groups silently — no error, no NULL, just absence.** Users cannot distinguish between "there is no data" and "data was suppressed by policy." This is intentional — even knowing suppression occurred could constitute a data disclosure.

**Projection policies block the column in SELECT but allow it in WHERE and JOIN.** This is the key distinction from masking, where the column always appears in results (just transformed).

### 🚫 Classic Exam Traps

| What the Exam Tests | The Correct Answer |
|---|---|
| "A user has SELECT on a table but can't query it — why?" | Missing USAGE on the database and/or schema |
| "Which role should own all production objects?" | SYSADMIN (or a custom role granted to SYSADMIN) |
| "A developer created a table with a custom role. SYSADMIN can't manage it — why?" | The custom role was never granted to SYSADMIN — orphan object problem |
| "Can a database role be set as a user's default role?" | No — database roles cannot be activated in sessions directly |
| "Can account roles be granted to database roles?" | No — never |
| "Does DDM protect data at rest?" | No — DDM only transforms data at query time; plaintext is in storage |
| "Which column-level security feature is needed for PCI card number storage?" | External Tokenization (not DDM) — PAN must not exist in plaintext |
| "Can External Tokenization be used when sharing data?" | No — external functions cannot be called in share context |
| "Which edition is minimum for HIPAA?" | Business Critical |
| "Is a signed BAA required for HIPAA compliance in Snowflake?" | Yes — always. Features without BAA = non-compliant |
| "What happens when RAP and masking both apply to the same table?" | RAP evaluates first, then masking applies to visible rows |
| "Can the same column have both a RAP argument and a masking policy?" | No — mutually exclusive |
| "Does cloning a table duplicate its row access policy?" | No — clone references the same policy object as the source |
| "What does a projection policy allow that a masking policy doesn't?" | Using the column in WHERE/JOIN without it appearing in SELECT results |
| "Who can grant privileges in a Managed Access Schema?" | Only the schema owner or ACCOUNTADMIN/SECURITYADMIN — not the object creator |

---

## 📚 Reference Links

| Resource | URL |
|---|---|
| Access Control Overview | https://docs.snowflake.com/en/user-guide/security-access-control-overview |
| Access Control Best Practices | https://docs.snowflake.com/en/user-guide/security-access-control-considerations |
| Column-Level Security | https://docs.snowflake.com/en/user-guide/security-column-intro |
| Dynamic Data Masking | https://docs.snowflake.com/en/user-guide/security-column-ddm-use |
| Row Access Policies | https://docs.snowflake.com/en/user-guide/security-row-intro |
| Aggregation Policies | https://docs.snowflake.com/en/user-guide/aggregation-policies |
| Projection Policies | https://docs.snowflake.com/en/user-guide/projection-policies |
| Object Tagging | https://docs.snowflake.com/en/user-guide/object-tagging |
| Data Classification | https://docs.snowflake.com/en/user-guide/classify-intro |
| Data Lineage | https://docs.snowflake.com/en/user-guide/ui-snowsight-lineage |
| Snowflake Editions | https://docs.snowflake.com/en/user-guide/intro-editions |
| Compliance | https://docs.snowflake.com/en/user-guide/intro-compliance |
| Snowflake Horizon | https://www.snowflake.com/en/data-cloud/horizon/ |
| Storage Integrations | https://docs.snowflake.com/en/user-guide/data-load-s3-config-storage-integration |

---

*© Study Notes for Snowflake Advanced Certification — For educational use only.*
