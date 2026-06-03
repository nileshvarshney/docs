# ❄️ Snowflake Advanced Certification Study Notes
## Topic: Security Principles and Use Cases

> **Exam Domain:** SnowPro Advanced – Data Engineer / Architect / Administrator
> **Last Updated:** June 2025
> **References:** Official Snowflake Documentation, Snowflake Engineering Blog, Trust Center

---

## Table of Contents

1. [Snowflake Security Philosophy](#1-snowflake-security-philosophy)
2. [Encryption](#2-encryption)
   - 2.1 Encryption at Rest
   - 2.2 The Hierarchical Key Model
   - 2.3 Encryption in Transit
   - 2.4 End-to-End Encryption (E2EE)
   - 2.5 Customer-Managed Keys – Tri-Secret Secure
   - 2.6 Key Rotation and Rekeying
3. [Network Security](#3-network-security)
   - 3.1 Network Policies
   - 3.2 Network Rules
   - 3.3 Network Policy Precedence
   - 3.4 External Access Integrations
   - 3.5 Private Connectivity
   - 3.6 AWS PrivateLink
   - 3.7 Azure Private Link
   - 3.8 Google Cloud Private Service Connect
4. [User, Role, and Grants Provisioning](#4-user-role-and-grants-provisioning)
   - 4.1 User Types
   - 4.2 Manual Provisioning
   - 4.3 SCIM – Automated Provisioning
   - 4.4 Grants Provisioning
5. [Authentication](#5-authentication)
   - 5.1 Authentication Policies
   - 5.2 Federated Authentication and SSO
   - 5.3 OAuth
   - 5.4 Multi-Factor Authentication (MFA)
   - 5.5 Key-Pair Authentication
   - 5.6 Security Integrations
6. [Snowflake's 2025 Authentication Enforcement Timeline](#6-snowflakes-2025-authentication-enforcement-timeline)
7. [Choosing the Right Authentication Method](#7-choosing-the-right-authentication-method)
8. [Exam Tips & Common Gotchas](#8-exam-tips--common-gotchas)

---

## 1. Snowflake Security Philosophy

Snowflake's security model is built around the principle of **defense in depth** — multiple independent layers of protection, each of which must be independently compromised for an attacker to reach data. No single control is relied upon as the only safeguard. The layers from outermost to innermost are:

**Network perimeter:** Network policies and private connectivity restrict which source IPs and network paths can reach Snowflake at all. An attacker who cannot connect to the account cannot even attempt authentication.

**Authentication:** Once connected, the user must prove their identity. Snowflake enforces strong authentication methods — MFA for humans, key-pair or OAuth for machines — making credential theft alone insufficient.

**Authorization:** Even after authentication, RBAC and DAC determine which objects the authenticated user's roles can interact with, and what operations they can perform.

**Data protection:** Even after authorization, governance policies (masking, row access, projection, aggregation) ensure users see only the minimum data appropriate for their role — not everything the object permits.

**Encryption:** Even if an attacker bypassed all prior layers and accessed storage directly, data at rest is encrypted with keys they do not have. Data in transit is encrypted so interception yields nothing usable.

**Audit:** Every action is logged. Even a successful breach is detectable and traceable. Access History, Query History, and Login History create an immutable audit trail.

Understanding this layered model — and which tool addresses which layer — is the foundational mental framework for both the exam and real-world architecture design.

---

## 2. Encryption

### 2.1 Encryption at Rest

All customer data stored in Snowflake is **encrypted by default, automatically, and without any configuration required**. This is not optional and cannot be disabled. Snowflake uses the **AES-256** (Advanced Encryption Standard with a 256-bit key) algorithm, which is the gold standard in symmetric encryption and is compliant with FIPS 140-2, NIST 800-53, and virtually all major regulatory frameworks.

The encryption applies to all data stored in Snowflake's managed storage layer — table data, query result caches, internal stage data, and metadata. Customers do not need to choose an encryption standard, manage keys, or take any action to enable this protection.

### 2.2 The Hierarchical Key Model

The sophistication of Snowflake's encryption lies not just in the algorithm, but in its **four-tier hierarchical key model**. Understanding this hierarchy — what each level protects, and why the hierarchy exists — is essential for the exam.

**Level 1 – Root Key (Hardware Security Module)**
The root key is the anchor of the entire hierarchy. It is stored in a cloud-provider-managed **Hardware Security Module (HSM)** — a tamper-resistant physical device specifically designed to hold cryptographic keys. The HSM is managed by the cloud provider (AWS KMS, Azure Key Vault, GCP Cloud KMS) and never exposes the root key to software processes. This is the most protected key in the hierarchy.

**Level 2 – Account Master Key**
Each Snowflake account has its own Account Master Key. This key is unique per account and is encrypted by the root key — meaning the HSM must participate in any operation that involves the Account Master Key. Account-level isolation is enforced here: one account's key hierarchy is completely separate from another account's hierarchy. This is how Snowflake ensures data isolation in a multi-tenant environment even if two customers' data resides on adjacent physical storage.

**Level 3 – Table Master Key**
Each table has its own Table Master Key, which is encrypted by the Account Master Key. If a Table Master Key is compromised, only the data in that specific table is at risk — not the entire account's data. This narrowing of scope is the primary purpose of the hierarchy: **limiting the blast radius of any individual key compromise**.

**Level 4 – File Key**
Every physical file (micro-partition) in Snowflake's storage has its own File Key, encrypted by the Table Master Key. File keys encrypt the actual data bytes. Because each micro-partition has a unique key, compromising one file key exposes only that single file — not the table, not the schema, not the account.

The architecture insight here is **scope reduction through layering**. Each lower layer encrypts a smaller unit of data with a more ephemeral key. The attacker's prize gets smaller at every layer they descend.

### 2.3 Encryption in Transit

All data moving between clients and Snowflake — and between Snowflake's internal components — is encrypted using **TLS 1.2 or higher** (Transport Layer Security). TLS establishes a cryptographically authenticated, encrypted channel before any data is transmitted, ensuring:

- **Confidentiality:** Data cannot be read by anyone intercepting the network traffic.
- **Integrity:** Data cannot be altered in transit without detection.
- **Authentication:** The client is communicating with the genuine Snowflake service, not an impersonator.

TLS 1.2 is the minimum enforced version. TLS 1.3 is supported and preferred where the client supports it, offering stronger cipher suites and faster handshake performance. Snowflake does not support older, vulnerable protocols (SSL, TLS 1.0, TLS 1.1).

An important nuance for the exam: data is **decrypted in memory** when Snowflake's compute layer processes a query (since computation requires readable data), then **re-encrypted** before being written back to storage. This ephemeral decryption-for-computation is an inherent necessity of any cloud compute environment and is not a security weakness — the compute nodes operate within Snowflake's secure VPC/VNet boundary.

### 2.4 End-to-End Encryption (E2EE)

**End-to-end encryption** in Snowflake refers to the comprehensive protection of data across its entire lifecycle within the platform — from the moment it enters the system to when it leaves. Snowflake achieves E2EE through the combination of encryption at rest and in transit, ensuring no cleartext data is ever exposed outside of controlled compute boundaries.

For external stages (data files in S3, Azure Blob, GCS), Snowflake also supports **client-side encryption** — where data files are encrypted by the client *before* they are uploaded to the external stage, using a client-managed encryption key. When Snowflake reads the file during a `COPY INTO` operation, it decrypts using the provided key. This means the data is encrypted even from Snowflake's own object storage access, providing an additional layer of protection for the most sensitive loading workflows.

For internal stages (Snowflake-managed storage), encryption is handled transparently by Snowflake — no client-side key management is required.

### 2.5 Customer-Managed Keys – Tri-Secret Secure

By default, Snowflake manages the full key hierarchy autonomously. For most organizations, this is sufficient — Snowflake's key management is robust, audited, and compliant with major frameworks. However, some organizations — particularly large financial institutions, government agencies, and those with specific regulatory mandates — require **direct control over their encryption keys**.

**Tri-Secret Secure** is Snowflake's solution, available on the **Business Critical Edition and VPS**. It introduces a **customer-managed key (CMK)** as a second required component alongside Snowflake's own key. The actual encryption key used to protect data is derived from both components combined — neither the Snowflake key nor the customer key alone is sufficient.

The "Tri" in Tri-Secret Secure refers to the three parties whose cooperation is required to decrypt data:
1. **Snowflake** — holds one component of the composite key
2. **The customer's KMS** (AWS KMS, Azure Key Vault, or GCP Cloud KMS) — holds the customer-managed key component
3. **The customer's authorization** — must explicitly allow Snowflake to use the CMK through IAM or key policy

The most important governance implication: if a customer **revokes their CMK** in their KMS, Snowflake immediately loses the ability to decrypt any customer data. Even Snowflake engineers and support staff cannot read the data. This is the ultimate "nuclear option" for data sovereignty — the customer retains absolute final control over data accessibility. Organizations subject to regulations requiring customer control over encryption (certain financial services rules, government mandates, or post-breach "switch off" requirements) leverage this feature specifically for that guarantee.

### 2.6 Key Rotation and Rekeying

**Key Rotation** is the regular replacement of encryption keys. Snowflake automatically rotates all keys in the Snowflake-managed hierarchy when they are more than **30 days old**. The retired key is kept accessible long enough to decrypt data it previously protected, then destroyed when Snowflake determines it is no longer needed. This rotation happens transparently without any customer action.

**Rekeying** is distinct from rotation. Rekeying re-encrypts existing data with a newly rotated key — rather than just rotating the key while old data remains encrypted with the old key. By default, Snowflake does not automatically rekey existing data after key rotation (retiring old keys naturally ages out). For Enterprise Edition and higher, customers can configure **automatic rekeying**, which ensures that all data is periodically re-encrypted under the current active key, eliminating any window where old-key-protected data exists.

Automatic rekeying is a requirement in some compliance frameworks (notably certain interpretations of PCI-DSS and HIPAA) that mandate regular re-encryption of stored data, not merely key rotation.

---

## 3. Network Security

### 3.1 Network Policies

A **network policy** is a Snowflake account-level or user-level configuration that restricts access to the Snowflake service based on the **source IP address** of the incoming connection. Network policies are the first line of defense — an IP address not matching the policy's allowed list is blocked before authentication is even attempted.

A network policy is composed of one or more **network rules** (the modern approach) or explicit IP address lists (the legacy approach). When both are present, the blocked list is evaluated first — if an IP matches the blocked list, access is denied regardless of whether it also appears on the allowed list.

**Where network policies can be applied:**
- **Account level:** Applies to all users and all connections to the account. Any connection from a non-permitted IP is rejected before it reaches authentication. This is the broadest, most comprehensive control.
- **User level:** Applies to a specific user. If a user has a user-level policy, it overrides the account-level policy for that user. This allows exceptions — for example, a DBA who works remotely might have a broader allowed IP range than the standard corporate policy enforces.
- **Security integration level:** Applies to connections governed by specific security integrations (Snowflake OAuth, External OAuth, SCIM). Allows different network restrictions for different authentication methods — for example, allowing OAuth connections from a wider IP range while restricting password-based connections to the corporate network.

The precedence rule for network policies is: **security integration policy overrides user policy, which overrides account policy**. The most specific policy wins.

### 3.2 Network Rules

**Network rules** are schema-level objects that represent a set of network identifiers (IPs, CIDR blocks, hostnames, VPC IDs, or Private Endpoint IDs). They are the building blocks used inside network policies and external access integrations. Network rules replaced the older pattern of specifying IP lists directly in the policy definition, because they are reusable, easier to manage, and support a wider range of identifier types.

**Network rule types by identifier:**

| Type | Identifier Examples | Use Case |
|---|---|---|
| `IPV4` | `192.168.1.0/24` | Standard IP-based filtering |
| `IPV6` | IPv6 CIDR ranges | IPv6 network environments |
| `HOST_PORT` | `api.external.com:443` | Egress filtering to external endpoints |
| `AWSVPC` | AWS VPC endpoint IDs | PrivateLink-specific controls |
| `AZURELINKID` | Azure Private Link service IDs | Azure Private Link controls |
| `GCPPSC` | GCP Private Service Connect IDs | GCP PSC controls |

**Network rule modes:**

| Mode | Direction | Used With |
|---|---|---|
| `INGRESS` | Inbound to Snowflake | Network policies (restricting who can connect) |
| `INTERNAL_STAGE` | Access to Snowflake internal stages | Network policies (restricting stage access) |
| `EGRESS` | Outbound from Snowflake | External access integrations (restricting where UDFs can call) |

This mode distinction is critical: INGRESS rules control *who can reach Snowflake*, while EGRESS rules control *where Snowflake can reach out to*. They serve opposite security purposes and are used with different objects.

### 3.3 Network Policy Precedence

When multiple network policies apply to the same connection, Snowflake resolves them using a strict precedence hierarchy:

1. **Security integration policy** (most specific — overrides all others for connections through that integration)
2. **User policy** (overrides account policy for that specific user)
3. **Account policy** (baseline for all users without a more specific override)

When a network policy includes both allowed and blocked network rules or IP lists, the **blocked list is evaluated first**. If the connecting IP appears on the blocked list, the connection is rejected — even if it also appears on the allowed list. This ensures that explicit blocks are never accidentally circumvented by being present on the allowed list.

### 3.4 External Access Integrations

**External access integrations** solve a specific security challenge: Snowflake UDFs (User-Defined Functions) and stored procedures written in Python, Java, or JavaScript can potentially make HTTP calls to external endpoints. Without controls, this creates a risk of data exfiltration — a malicious UDF could silently send query results to an external server.

An external access integration addresses this by:

1. **Requiring explicit EGRESS network rules** that enumerate exactly which external hostnames and ports UDF/procedure code is allowed to call. Code can only reach destinations explicitly whitelisted in the integration.
2. **Requiring explicit secrets** for any authentication the UDF needs with the external endpoint (stored as Snowflake Secret objects — not hardcoded in the UDF code).
3. **Being attached to the UDF or procedure at creation time** — the integration's permissions are evaluated when the function definition is compiled, not at runtime.
4. **Being governed by `USAGE` privilege** — only roles explicitly granted USAGE on the external access integration can create UDFs that use it.

The external access integration is the gatekeeper for all outbound network calls from Snowflake's compute layer. Without it, UDFs cannot make any external calls. With it, calls are constrained to the declared destinations only.

Administrators can monitor all outbound requests made by UDFs and procedures through the `EXTERNAL_ACCESS_HISTORY` view in `ACCOUNT_USAGE`, providing a complete audit trail of what Snowflake's compute layer called externally.

### 3.5 Private Connectivity

**Private connectivity** allows Snowflake to be accessed from a customer's cloud environment **without any traffic crossing the public internet**. Instead of routing through Snowflake's public endpoint (e.g., `account.snowflakecomputing.com` resolving to a public IP), the customer's network connects to a **private endpoint** that routes directly to Snowflake's VPC/VNet through the cloud provider's backbone network.

The key security benefit is **elimination of the public internet from the data path**:
- No possibility of public internet interception or man-in-the-middle attacks on the network path (though TLS still protects the data regardless)
- Allows organizations with strict security postures to enforce "no public internet egress" rules while still using Snowflake
- Required for PCI-DSS and HIPAA compliance postures that mandate private network connectivity for sensitive data systems

**Private connectivity requires Business Critical Edition or higher.** It is the single most commonly misunderstood edition-gating requirement — many organizations assume network isolation is achievable on lower editions, but PrivateLink and its equivalents are exclusive to Business Critical and VPS.

**Important conceptual distinction:** Private connectivity and network policies are complementary but independent controls. Private connectivity determines the *network path*; network policies determine *which source IPs on that path are permitted*. An organization using PrivateLink should also configure network policies that block connections from outside the expected VPC CIDR ranges — private connectivity alone does not enforce that all connections originate from the expected network.

### 3.6 AWS PrivateLink

AWS PrivateLink creates **VPC Endpoint Services** that allow traffic from an AWS customer VPC to reach Snowflake's VPC without traversing the internet. From the customer's VPC, a VPC Interface Endpoint is created that resolves to a private IP address within the customer's own subnet — effectively making Snowflake appear as a local service within the VPC.

**Key architectural points:**
- Supported for Snowflake accounts hosted on AWS only (cannot use AWS PrivateLink to connect to a Snowflake account hosted on Azure or GCP — the cloud provider must match)
- Cross-region PrivateLink is supported via custom endpoint services, allowing a customer VPC in one AWS region to connect to a Snowflake account in a different region
- On-premises environments can combine **AWS Direct Connect** with PrivateLink — Direct Connect brings the on-premises network into the AWS backbone, and PrivateLink then provides private access to Snowflake from there
- Requires ACCOUNTADMIN to authorize PrivateLink for the Snowflake account (`SYSTEM$AUTHORIZE_PRIVATELINK`)
- After enabling PrivateLink, Snowflake provides a PrivateLink-specific account URL — clients must be configured to use this URL, not the standard public URL, for connections to route through the private endpoint

### 3.7 Azure Private Link

Azure Private Link provides the equivalent private connectivity for Snowflake accounts hosted on Microsoft Azure. The mechanism uses **Private Endpoints** in the customer's Azure Virtual Network (VNet) that connect to Snowflake's VNet through Azure's backbone, bypassing the public internet.

**Key architectural points:**
- Supported for Azure-hosted Snowflake accounts only
- The customer creates a Private Endpoint in their Azure VNet that targets the Snowflake Private Link Service
- Requires approval — when the customer creates the Private Endpoint request, Snowflake (as the service provider) must approve it. The approval process is initiated and managed through Snowflake's side
- The connection uses IPv4 TCP traffic only (Azure limitation for Private Link configurations)
- TCP ports 443 and 80 must be open in the network security group for the Private Endpoint network card
- ARM (Azure Resource Manager) VNets are required — classic Azure networking is not supported
- Once approved and configured, the customer uses a Private Link-specific Snowflake URL

### 3.8 Google Cloud Private Service Connect

**Google Cloud Private Service Connect (PSC)** is the GCP equivalent, allowing GCP customers to connect to Snowflake accounts hosted on GCP through Google's private networking backbone.

**Key architectural points:**
- Supported for GCP-hosted Snowflake accounts only
- The customer creates a PSC endpoint (a forwarding rule in their GCP project) that targets the Snowflake PSC service attachment
- DNS configuration is the customer's responsibility — all requests to Snowflake must be routed through the PSC endpoint, which means configuring DNS so that the standard Snowflake hostname resolves to the PSC endpoint's forwarding rule IP address
- As of December 2025, network rules and policies support GCP PSC endpoint IDs, allowing fine-grained network policy controls specific to PSC connections
- Requires ACCOUNTADMIN to authorize PSC for the account using `SYSTEM$AUTHORIZE_PRIVATELINK` with a GCP project ID and access token

**Cross-cloud constraint (all three providers):** Private connectivity endpoints are cloud-provider-specific and cannot cross cloud boundaries. A customer VPC on AWS cannot use PrivateLink to connect to a Snowflake account on Azure. The cloud provider hosting the client environment must match the cloud provider hosting the Snowflake account.

---

## 4. User, Role, and Grants Provisioning

### 4.1 User Types

Snowflake distinguishes between different **user types**, which governs what authentication methods they can use and how they are managed. This distinction became especially significant with Snowflake's 2025 authentication enforcement changes.

**TYPE = PERSON (or NULL/unset):** Human users who log in interactively through Snowsight, SnowSQL, drivers, or other clients. These users are expected to use MFA (enforced as of 2025) or SSO/SAML. They can enroll in MFA through Snowsight.

**TYPE = SERVICE:** Non-human accounts designed for machine-to-machine authentication — ETL pipelines, BI tool connectors, CI/CD automation, API integrations. Service users cannot use passwords or MFA. They must use key-pair authentication, OAuth, Programmatic Access Tokens (PAT), or Workload Identity Federation (WIF). They cannot log into Snowsight.

**TYPE = LEGACY_SERVICE:** A transitional type for existing service accounts that were created before the SERVICE type was introduced and still used password-based authentication. Snowflake deprecated this type in 2025 and blocked creation of new LEGACY_SERVICE users. Existing ones were forcibly converted to SERVICE users as of November 2025.

Understanding the distinction between PERSON and SERVICE users is fundamental to designing the right authentication strategy — the two populations have completely different security requirements and tooling.

### 4.2 Manual Provisioning

**Manual provisioning** means Snowflake administrators directly create and manage users and roles using SQL commands (`CREATE USER`, `CREATE ROLE`, `GRANT ROLE TO USER`, etc.). This is straightforward for small organizations but becomes problematic at scale:

- Keeping Snowflake users in sync with the corporate directory (HR systems, Active Directory) requires manual work or custom scripting
- Offboarding risk: when someone leaves the company, their Snowflake user must be manually disabled. Any delay creates an access window for a departed employee
- No single source of truth — Snowflake user records may drift from the IdP's records
- Audit burden: proving who had access to what at a given point in time requires correlating multiple systems

For any organization with more than a handful of Snowflake users, manual provisioning is considered an anti-pattern. It is appropriate only for very small teams or for bootstrapping an account before SCIM is configured.

### 4.3 SCIM – Automated Provisioning

**SCIM (System for Cross-domain Identity Management)** is an open standard protocol that allows an Identity Provider (IdP) to automatically create, update, and deactivate user accounts in Snowflake whenever users are added, modified, or removed in the corporate directory. Supported IdPs include Okta, Microsoft Entra ID (formerly Azure AD), and custom SCIM-compatible systems.

SCIM solves the manual provisioning problems entirely:

**Automatic user lifecycle management:** When an employee joins the company and is added to the IdP, SCIM automatically creates their Snowflake user. When they leave, their IdP account is disabled or removed, and SCIM automatically disables or removes their Snowflake user — no manual Snowflake action required, eliminating the offboarding risk window.

**Group-to-role mapping:** SCIM can synchronize IdP groups to Snowflake roles. A user added to the "Data Analysts" group in the IdP automatically gets the `DATA_ANALYST` role in Snowflake. This makes role assignment a byproduct of normal HR processes, not a separate administrative step.

**Single source of truth:** User attributes (names, email addresses, departments) are managed in the IdP and pushed to Snowflake, keeping them consistent.

SCIM in Snowflake is implemented through a **SCIM security integration** — a Snowflake object that provides the SCIM API endpoint and access token that the IdP uses to communicate with Snowflake. A dedicated provisioner role (not ACCOUNTADMIN) is created for the SCIM integration to run as, following least-privilege principles.

**Critical governance point:** If using Okta SCIM, care must be taken with the "Sync Password" setting. If enabled, Okta creates a Snowflake password for each provisioned user, which would allow users to bypass SSO and log in directly with a password. For organizations requiring SSO-only access, this setting must be disabled.

### 4.4 Grants Provisioning

While user provisioning (creating accounts) can be automated via SCIM, **grants provisioning** — assigning privileges to roles, and roles to users — must be managed carefully to avoid privilege accumulation over time.

**FUTURE GRANTS** are essential for maintaining consistent access as databases grow. Without them, every new table, view, or schema added to a database requires a manual re-grant. With future grants, new objects automatically inherit the privileges configured at the schema or database level, closing the gap between object creation and access control.

**Regular privilege audits** should be conducted using the `GRANTS_TO_ROLES` and `GRANTS_TO_USERS` views in `ACCOUNT_USAGE`. Over time, roles accumulate privileges through one-off grants, and users accumulate roles through organizational changes. Without periodic review, the effective privilege set of common roles can expand far beyond what was intended — a phenomenon called **privilege creep**.

**Role recertification** is a governance process where role owners periodically review who holds each role and whether the current privilege set is still appropriate. This is a compliance requirement in several frameworks (SOX, ISO 27001) and is best supported by querying Snowflake's `ROLE_GRANTS` and `ACCESS_HISTORY` views.

---

## 5. Authentication

### 5.1 Authentication Policies

An **authentication policy** is a Snowflake schema-level object that defines the rules governing how users can authenticate to the account. It is the centralized enforcement mechanism for authentication requirements — the declarative specification of what authentication is acceptable.

An authentication policy controls four dimensions of the authentication experience:

**Allowed authentication methods:** Specifies which methods are permitted. Options include `PASSWORD`, `SAML`, `OAUTH`, `KEYPAIR`, and `PROGRAMMATIC_ACCESS_TOKEN`. Any method not listed is blocked, even if the user attempts to use it. For example, a policy that allows only `SAML` and `KEYPAIR` means password-based logins are impossible for users under that policy, regardless of whether a password has been set on their account.

**MFA enrollment and requirement:** Specifies whether MFA is optional, required for password authentication only, or required for all authentication. The `MFA_ENROLLMENT` parameter controls whether users must enroll. Enrollment can only happen through Snowsight (the web UI) — this means if a policy requires MFA, users must be able to access Snowsight at least once to complete enrollment.

**Allowed client types:** Restricts which client applications (Snowsight, SnowSQL, JDBC drivers, ODBC drivers, etc.) can be used to connect. This prevents access through unofficial or unapproved tools. For example, an organization might allow Snowsight and approved JDBC drivers but block direct SnowSQL CLI access for most users.

**Allowed security integrations:** When multiple SAML IdPs are configured, the policy can specify which IdPs are available to users subject to that policy. This allows different user populations to use different IdPs (e.g., employees use Okta, contractors use a different IdP) while both are governed by authentication policies.

**Where authentication policies apply:** A policy can be applied at the **account level** (default for all users) or at the **user level** (overrides account policy for specific users). User-level policies always take precedence. This layering allows a base security standard to be enforced account-wide, with specific overrides for users who have different authentication needs (e.g., service accounts exempted from MFA requirements).

### 5.2 Federated Authentication and SSO

**Federated authentication** is the mechanism by which users authenticate to an external **Identity Provider (IdP)** — such as Okta, Microsoft Entra ID, Ping Identity, or a corporate ADFS instance — and Snowflake trusts that IdP's confirmation of the user's identity. Instead of Snowflake validating a password, the IdP validates it and issues a **security token** (a SAML assertion) that Snowflake accepts as proof of identity.

**Single Sign-On (SSO)** is the user experience outcome of federated authentication. Because the IdP manages authentication centrally, a user who has already authenticated to the IdP (e.g., when they logged into their laptop or accessed another corporate application) can access Snowflake without entering credentials again — the IdP's existing session satisfies Snowflake's authentication request.

The protocol underlying Snowflake's federated authentication is **SAML 2.0** (Security Assertion Markup Language). The SAML flow works as follows:

When a user attempts to access Snowflake without an active session, Snowflake redirects them to the configured IdP. The IdP verifies the user's identity (via their existing session, or by prompting for credentials if no session exists). The IdP then generates a **SAML assertion** — a digitally signed XML document that states "this user has authenticated, here are their attributes." The assertion is sent to Snowflake, which validates the IdP's digital signature and creates a Snowflake session.

**Why federated authentication is preferred for human users:**

- **Centralized control:** Disabling a user in the IdP immediately prevents their Snowflake access, even if their Snowflake account still exists. This is the most reliable offboarding mechanism.
- **MFA centralization:** MFA is enforced at the IdP, not in Snowflake. The IdP's MFA is stronger and more flexibly configurable than Snowflake's native MFA for many organizations.
- **Password management:** Users do not have separate Snowflake passwords to manage or forget. Corporate password policies and rotation requirements are handled by the IdP.
- **Audit consolidation:** Authentication events are logged both in the IdP and in Snowflake's `LOGIN_HISTORY`, providing two independent audit trails.

Federated authentication is configured through a **SAML2 Security Integration** in Snowflake, which specifies the IdP's entity URL, SSO URL, and certificate.

### 5.3 OAuth

**OAuth 2.0** is an authorization framework (not, technically, an authentication protocol — though it is used for authentication in practice) that allows third-party applications to obtain **delegated access tokens** representing a user's or service account's access to Snowflake, without the application ever receiving or storing the user's credentials.

Snowflake supports two distinct OAuth implementations with different use cases:

**Snowflake OAuth** is Snowflake's built-in OAuth authorization server. It is primarily designed to allow **BI tools and client applications** (Tableau, Power BI, Looker, etc.) to authenticate to Snowflake on behalf of the user. When a user accesses a Tableau dashboard, Tableau authenticates to Snowflake using OAuth — it obtains an access token representing the user, and uses that token for all subsequent queries. The token is short-lived and scoped to specific roles, which is more secure than the application holding the user's actual Snowflake password.

**External OAuth** allows Snowflake to accept access tokens issued by a **third-party OAuth authorization server** — Okta, Microsoft Entra ID, or any OIDC-compliant IdP. This enables a broader federation pattern where the organization's existing OAuth/OIDC infrastructure can authorize Snowflake access. External OAuth is particularly important for automated workflows and APIs that need to call Snowflake without storing long-lived credentials.

**Key security properties of OAuth:**

- **Short-lived tokens:** Access tokens typically expire in minutes to hours. Even if a token is compromised, the exposure window is limited.
- **No credential sharing:** The application receives a token, never the user's password or private key. The authorization server remains the only entity that handles credentials.
- **Scope limitation:** Tokens can be scoped to specific roles or operations, applying least-privilege principles to token-based access.
- **Refresh tokens:** OAuth supports refresh tokens (longer-lived tokens that can obtain new access tokens), which enables continuous access without requiring re-authentication while still allowing the authorization server to revoke access by refusing to issue new access tokens.

### 5.4 Multi-Factor Authentication (MFA)

**Multi-Factor Authentication** requires users to prove their identity using two or more independent factors: something they **know** (password), something they **have** (a phone with an authenticator app), or something they **are** (biometric). If an attacker obtains a password through phishing or a data breach, they still cannot authenticate without the second factor.

Snowflake's native MFA uses **Duo Security** (a Cisco product) as the MFA provider. When a user with MFA enrolled logs in with their password, Snowflake triggers a Duo push notification to the user's registered device. The user must approve the notification before the login proceeds.

**2025 MFA enforcement:** Snowflake made a major security policy change in 2025, mandating MFA for all human users. This reflects the industry recognition that password-only authentication is no longer acceptable as a baseline. Password-only authentication for human users was phased out through 2025, with complete enforcement by early 2026. See Section 6 for the detailed timeline.

**MFA token caching** is a feature for scenarios where frequent re-authentication is operationally disruptive (e.g., batch ETL processes that reconnect frequently). When enabled at the account level via the `ALLOW_CLIENT_MFA_CACHING` parameter, the Snowflake client caches the MFA token locally for a limited period, allowing reconnections without triggering a new MFA prompt. This is an account parameter and cannot be overridden at the session level — enabling it is a deliberate administrative decision with security trade-offs.

### 5.5 Key-Pair Authentication

**Key-pair authentication** uses **asymmetric cryptography** — a mathematically related pair of keys where the private key can create a signature that only the corresponding public key can verify. This eliminates passwords from service account authentication entirely.

**How it works:** The user or service account generates a key pair (a private key and a public key). The **public key is registered with Snowflake** on the user's account — it is safe to expose, as it can only verify signatures, not create them. The **private key is kept exclusively on the client** and never transmitted. When authenticating, the client uses the private key to sign a challenge issued by Snowflake. Snowflake verifies the signature using the registered public key. If the signature is valid, authentication succeeds — without any password or secret ever being transmitted across the network.

**Why key-pair authentication is preferred for service accounts:**

- **No shared secrets transmitted:** The private key never leaves the client. Even if the authentication request is intercepted, nothing useful for re-authentication is captured.
- **No password to phish or brute-force:** Service accounts are not vulnerable to the most common human-targeting attacks.
- **Compatible with automation:** Key-pair auth works seamlessly in automated pipelines, CI/CD systems, and ETL tools without human interaction.
- **Scalable rotation:** Private keys can be rotated by registering a new public key on the Snowflake user and updating the key file used by the service, without changing any passwords.

**RSA-2048 minimum, RSA-4096 recommended:** Snowflake requires a minimum key size of 2048 bits. For production security, 4096-bit keys are recommended.

**Encrypted private keys:** Private key files can be stored encrypted with a passphrase. This adds a second protection layer — even if the private key file is stolen from disk, it cannot be used without the passphrase. Snowflake's clients support providing the passphrase when configuring the connection.

**Key rotation:** Each Snowflake user account supports registering two public keys simultaneously (`RSA_PUBLIC_KEY` and `RSA_PUBLIC_KEY_2`). This supports **zero-downtime key rotation** — register the new public key alongside the existing one, update the client to use the new private key, then remove the old public key. The service is never disrupted during the rotation.

### 5.6 Security Integrations

A **security integration** is a Snowflake account-level object that defines how Snowflake interacts with an external security system. It is the configuration container for any external authentication or authorization relationship. Different types of security integrations serve different purposes:

**SAML2 Security Integration:** Configures federated authentication with a SAML IdP. Contains the IdP's metadata — the entity ID, the SSO URL (where Snowflake redirects users for authentication), and the IdP's X.509 certificate (used by Snowflake to verify the signature on SAML assertions). Multiple SAML2 security integrations can exist in one account — useful when different user populations (employees vs. contractors, or different business units) use different IdPs.

**Snowflake OAuth Security Integration:** Configures Snowflake as an OAuth authorization server for client applications (Tableau, Looker, etc.). Defines which client applications are registered, what roles they are allowed to request tokens for, and token lifetime settings.

**External OAuth Security Integration:** Configures Snowflake to accept tokens from a third-party OAuth authorization server (Okta, Entra ID, etc.). Contains the IdP's token endpoint and the rules for mapping IdP user attributes to Snowflake roles.

**SCIM Security Integration:** Provides the SCIM API endpoint and bearer token that the IdP uses to provision and manage Snowflake users. Specifies which role the SCIM operations run as.

**API Authentication Security Integration:** Used with external access integrations to hold OAuth credentials for external APIs that UDFs need to call.

Security integrations are account-level objects that only `ACCOUNTADMIN` can create. Once created, specific `USAGE` or other privileges can be granted to allow operational roles to reference them (e.g., a data engineering role might need `USAGE` on an API authentication integration to create UDFs that call external APIs).

---

## 6. Snowflake's 2025 Authentication Enforcement Timeline

This is a high-priority topic for the exam and for real-world architects. Snowflake made a major platform-wide security upgrade starting in 2024–2025, eliminating password-only authentication across the platform. The driving motivation was a series of credential-stuffing attacks across cloud platforms that demonstrated the inadequacy of passwords alone.

### The Enforcement Phases

**Phase 1 – New Account Defaults (October 2024 onward)**
All newly created Snowflake accounts began enforcing MFA by default for human users. Password-only logins were no longer the default baseline for new accounts.

**Phase 2 – Human User MFA Mandate (April–August 2025)**
All human users on existing accounts who log in with passwords were required to enroll in MFA. After April 2025, the next password-based login prompted mandatory MFA enrollment. By August 2025, password-only logins became structurally impossible for human users — MFA or SSO was required.

**Phase 3 – LEGACY_SERVICE User Elimination (November 2025)**
`LEGACY_SERVICE` users (the transitional type allowing service accounts to use passwords) were forcibly converted to `SERVICE` users. Password-based authentication for all service accounts became impossible. Service accounts must use key-pair authentication, OAuth, Programmatic Access Tokens, or Workload Identity Federation.

**Phase 4 – Full Enforcement (2026)**
Ongoing enforcement milestones extending through 2026 complete the transition. Creation of new LEGACY_SERVICE users was blocked from November 2025. All non-human users must use modern, credential-free authentication methods by mid-2026.

### Implications for Architecture Design

This enforcement timeline has significant implications for system design:

Any BI tool, ETL pipeline, or application that previously used a Snowflake username and password must be migrated to OAuth or key-pair authentication. Many third-party tools updated their Snowflake connectors in 2024–2025 to support these methods, but some legacy integrations may require vendor updates or alternative approaches.

Human users who accessed Snowflake via Snowflake-native authentication (not SSO) must be migrated to MFA enrollment or federated SSO. Organizations that have not deployed SSO will need Snowflake's native Duo-based MFA as the minimum baseline.

---

## 7. Choosing the Right Authentication Method

A critical skill for the Advanced exam is matching the authentication method to the use case. The following framework covers the key decision points:

### For Human Users

**SSO/SAML** is the gold standard for human authentication when the organization has an IdP. It provides centralized control, eliminates Snowflake-specific passwords, enables corporate MFA policies, and ensures Snowflake access is governed by the same identity lifecycle as all other corporate systems. Any organization with Okta, Entra ID, or Ping already deployed should use SAML for all human users.

**Snowflake Native MFA (Duo)** is the fallback for organizations without an IdP or for accounts where SSO is not yet configured. It provides the required second factor without requiring IdP integration but adds operational overhead (managing a separate Snowflake credential alongside corporate credentials).

### For Service Accounts and Automation

**Key-pair authentication** is the primary method for service accounts. It is well-suited to long-running pipelines, scheduled tasks, and integrations where a stable, non-expiring credential is needed. The private key must be securely stored (environment variables, secrets managers, HSMs — never in source code).

**OAuth (External OAuth or Snowflake OAuth)** is preferred when short-lived credentials are acceptable and the integration supports OAuth token-based flows. Particularly appropriate for BI tools, modern data connectors, and APIs. The short-lived nature of OAuth access tokens provides excellent security against token theft.

**Programmatic Access Tokens (PAT)** are a newer Snowflake mechanism providing short-lived, scoped tokens for programmatic access — a middle ground between key-pair (long-lived) and OAuth (requires authorization server infrastructure).

**Workload Identity Federation (WIF)** allows cloud workloads (AWS EC2, Lambda, GCP Cloud Run, Azure VMs, etc.) to authenticate to Snowflake using the cloud platform's native identity mechanism (IAM roles, Workload Identity, Managed Identity) rather than any credential at all. This is the most secure option for cloud-native workloads because there is no credential to store, rotate, or steal — the cloud platform vouches for the workload's identity.

### Authentication Method Selection Matrix

| Scenario | Recommended Method | Why |
|---|---|---|
| Human user, corporate IdP available | SAML / SSO | Centralized control, corporate MFA, no Snowflake-specific credential |
| Human user, no IdP | Native MFA (Duo) | Minimum required second factor |
| ETL/pipeline service account | Key-pair | Stable credential, no human interaction needed |
| BI tool connecting on behalf of user | Snowflake OAuth | Short-lived, scoped, no password sharing |
| Third-party API authorization | External OAuth | Leverage existing OAuth infrastructure |
| Cloud workload (AWS, GCP, Azure) | Workload Identity Federation | No credential to manage at all |
| Heavily regulated, zero-credential mandate | WIF + Network Policy | Maximum security posture |

---

## 8. Exam Tips & Common Gotchas

### ⚡ High-Yield Conceptual Points

**Encryption is always on — there is no "opt in."** All Snowflake data at rest is AES-256 encrypted. All data in transit uses TLS 1.2+. This is not configurable and cannot be disabled. The exam may present scenarios suggesting encryption needs to be "enabled" — the correct answer is that it is always active.

**The four-layer key hierarchy (HSM → Account Master Key → Table Master Key → File Key) limits blast radius.** If a file key is compromised, only that file is exposed. This is the architecture exam answer for "why does Snowflake use a hierarchical key model?"

**Tri-Secret Secure requires Business Critical Edition.** It is not available on Standard or Enterprise. The customer-managed key means Snowflake cannot decrypt data without customer cooperation — revoking the key makes all data permanently inaccessible to everyone.

**Key rotation vs. rekeying are different.** Rotation replaces the key going forward; rekeying re-encrypts existing data with the new key. Automatic rekeying requires Enterprise Edition and above.

**Network policy blocked list is evaluated before the allowed list.** If an IP appears on both lists, it is blocked. The blocked list takes absolute precedence.

**Network rules have three modes: INGRESS, INTERNAL_STAGE, EGRESS.** INGRESS restricts inbound connections; EGRESS restricts where UDFs can call externally. These are used with different objects (policies vs. external access integrations) and serve opposite security purposes.

**Network policy precedence: security integration overrides user, user overrides account.** The most specific policy always wins.

**External access integrations are required for UDFs to make outbound network calls.** Without one, no outbound calls are possible. With one, only the declared destinations are reachable.

**Private connectivity (PrivateLink, Private Link, PSC) requires Business Critical Edition.** It is not available on Standard or Enterprise. This is one of the most commonly misunderstood edition requirements.

**Private connectivity and cloud provider must match.** AWS PrivateLink only works for Snowflake accounts on AWS. Azure Private Link only works on Azure. GCP PSC only works on GCP. Cross-cloud private connectivity is not supported.

**Authentication policies are schema-level objects applied at account or user level.** User-level authentication policies override account-level policies.

**SAML/OAuth-based authentication does NOT require MFA within Snowflake** — because the IdP handles authentication (and MFA) externally. Snowflake's MFA requirement applies only to password-based logins.

**Service accounts (TYPE=SERVICE) cannot use passwords and cannot log into Snowsight.** They must use key-pair, OAuth, PAT, or WIF. This distinction drives the entire service account authentication design.

**SCIM requires a SCIM security integration** — it is not a built-in feature. A dedicated provisioner role (not ACCOUNTADMIN) should run SCIM operations.

**Disabling "Sync Password" in Okta SCIM is critical** for SSO-only environments — if enabled, users get a Snowflake password that bypasses SSO.

**Key-pair rotation uses two simultaneous public key slots (RSA_PUBLIC_KEY and RSA_PUBLIC_KEY_2)** — enabling zero-downtime rotation without service disruption.

**MFA caching (ALLOW_CLIENT_MFA_CACHING) is an account parameter** — it cannot be set at the session level and is not available to individual users to configure themselves.

### 🚫 Classic Exam Traps

| What the Exam Tests | The Correct Answer |
|---|---|
| "Does enabling PrivateLink eliminate the need for network policies?" | No — they are complementary. PrivateLink controls the network path; network policies control which source IPs are permitted. Both should be used together. |
| "Which edition is required for AWS PrivateLink?" | Business Critical (or higher) |
| "Can AWS PrivateLink connect to a Snowflake account on Azure?" | No — cloud provider must match |
| "Is encryption at rest optional in Snowflake?" | No — it is always on, always AES-256, cannot be disabled |
| "What happens when a customer revokes their CMK in Tri-Secret Secure?" | All data becomes inaccessible to everyone, including Snowflake employees |
| "Does key rotation automatically rekey existing data?" | No — rotation replaces the key going forward; rekeying is a separate process |
| "Which network rule mode controls UDF outbound calls?" | EGRESS mode, used with external access integrations |
| "Which network rule mode controls inbound Snowflake connections?" | INGRESS mode, used with network policies |
| "Can a LEGACY_SERVICE user use a password after November 2025?" | No — LEGACY_SERVICE users were forcibly converted to SERVICE users, making passwords impossible |
| "Can a TYPE=SERVICE user log into Snowsight?" | No — service users cannot use the web UI |
| "Is SAML 2.0 or OAuth an 'authentication' or 'authorization' protocol?" | SAML is an authentication protocol; OAuth is technically an authorization framework (though used for authentication via OIDC) |
| "Does federated SSO require MFA within Snowflake?" | No — MFA is handled by the IdP. Snowflake trusts the IdP's authentication. |
| "Can multiple SAML2 security integrations exist in one account?" | Yes — useful for different user populations using different IdPs |
| "What privilege does a role need to create a UDF using an external access integration?" | USAGE on the external access integration |
| "Which two public key slots support zero-downtime key rotation?" | RSA_PUBLIC_KEY and RSA_PUBLIC_KEY_2 |

---

## 📚 Reference Links

| Resource | URL |
|---|---|
| End-to-End Encryption | https://docs.snowflake.com/en/user-guide/security-encryption-end-to-end |
| Encryption Key Management | https://docs.snowflake.com/en/user-guide/security-encryption-manage |
| Network Policies | https://docs.snowflake.com/en/user-guide/network-policies |
| Network Rules | https://docs.snowflake.com/en/user-guide/network-rules |
| External Access Integrations | https://docs.snowflake.com/en/developer-guide/external-network-access/creating-using-external-network-access |
| Private Connectivity (Inbound) | https://docs.snowflake.com/en/user-guide/private-connectivity-inbound |
| AWS PrivateLink | https://docs.snowflake.com/en/user-guide/admin-security-privatelink |
| Azure Private Link | https://docs.snowflake.com/en/user-guide/privatelink-azure |
| GCP Private Service Connect | https://docs.snowflake.com/en/user-guide/private-service-connect-google |
| Authentication Policies | https://docs.snowflake.com/en/user-guide/authentication-policies |
| Federated Authentication (SAML) | https://docs.snowflake.com/en/user-guide/admin-security-fed-auth-overview |
| Snowflake OAuth | https://docs.snowflake.com/en/user-guide/oauth-snowflake-overview |
| External OAuth | https://docs.snowflake.com/en/user-guide/oauth-ext-overview |
| MFA | https://docs.snowflake.com/en/user-guide/ui-snowsight-profile |
| Key-Pair Authentication | https://docs.snowflake.com/en/user-guide/key-pair-auth |
| SCIM (Okta) | https://docs.snowflake.com/en/user-guide/scim-okta |
| SCIM (Entra ID) | https://docs.snowflake.com/en/user-guide/scim-azure |
| MFA Migration Best Practices | https://docs.snowflake.com/en/user-guide/security-mfa-migration-best-practices |
| Tri-Secret Secure | https://docs.snowflake.com/en/user-guide/security-encryption-manage#tri-secret-secure |

---

*© Study Notes for Snowflake Advanced Certification — For educational use only.*
