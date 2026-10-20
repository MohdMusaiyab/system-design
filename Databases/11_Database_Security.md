# 11. Database Security: Defense-in-Depth Architecture

In modern distributed systems, the database is the crown jewel. Network firewalls, API gateways, and web application firewalls (WAFs) protect perimeter entry points, but the database stores the actual customer records, financial ledgers, and proprietary intelligence. If an attacker breaches the application tier or an insider obtains illicit access, the database's internal security architecture is the final line of defense.

Database security is not a single feature; it is an integrated **defense-in-depth model** spanning authentication, role-based authorization, query execution models, cryptographic storage, dynamic secrets management, sanitization pipelines, and immutable auditability.

---

## 1. Authentication vs. Authorization

Securing a database begins with distinguishing **identity verification** from **permission enforcement**:

* **Authentication (AuthN - "Who are you?"):** Validating that the client connecting to the database socket is genuinely who they claim to be.
* **Authorization (AuthZ - "What are you permitted to do?"):** Determining whether that verified identity has the privilege to execute a specific operation (`SELECT`, `INSERT`, `TRUNCATE`, `ALTER`) on a specific object (table, view, column, row).

```mermaid
flowchart LR
    Client["Client / Application"] --> AuthN{"1. Authentication<br/>Who are you?"}
    AuthN -->|"Success: Identity Verified"| AuthZ{"2. Authorization<br/>What can you touch?"}
    AuthN -->|"Failure: Invalid Creds / Cert"| Reject1["Reject Connection (TCP RST)"]
    AuthZ -->|"Allowed by Role & Policy"| Execute["Execute SQL Operation"]
    AuthZ -->|"Forbidden (Insufficient Privilege)"| Reject2["Error: Permission Denied"]
```

### 1.1 Modern Authentication Mechanisms

Traditional database setups relied on static username/password pairs sent as cleartext or weakly hashed MD5 digests. Production databases enforce much stronger protocols:

| Mechanism | How It Works | Strengths | Failure Modes & Trade-offs |
| :--- | :--- | :--- | :--- |
| **SCRAM-SHA-256** | Salted Challenge Response Authentication Mechanism. Uses cryptographic handshakes where passwords never traverse the wire. | Defends against sniffing and replay attacks; password hashes stored on disk cannot be easily cracked. | Requires modern client driver support; CPU overhead during connection setup handshakes. |
| **Mutual TLS (mTLS)** | Both the database and client exchange X.509 cryptographic certificates to verify each other's identity during the TLS handshake. | Eliminates passwords entirely; connection is encrypted and authenticated at the network transport layer. | Complex Public Key Infrastructure (PKI) required to issue, rotate, and revoke certificates before expiry. |
| **Cloud IAM Auth** (AWS IAM / GCP IAM) | Short-lived ephemeral auth tokens (valid for 15 minutes) signed by cloud identity providers (e.g. AWS STS). | Zero static database passwords; centralized revocation through enterprise IAM policies. | Token generation adds 50 ms–150 ms to connection establishment (requires connection pooling). |
| **Kerberos / Active Directory** | Ticket-granting service (KDC) issues cryptographic tickets to clients for single sign-on (SSO). | Seamless enterprise directory integration; centralized corporate user lifecycle management. | Single point of failure if KDC drops; time drift between servers breaks Kerberos handshakes. |

```sql
-- PostgreSQL pg_hba.conf configuration enforcing SCRAM-SHA-256 and client certificates
# TYPE  DATABASE        USER            ADDRESS                 METHOD
hostssl all             app_user        10.0.0.0/16             scram-sha-256
hostssl all             admin_user      10.0.10.0/24            cert clientcert=verify-full
host    all             all             0.0.0.0/0               reject
```

---

## 2. Database Roles, Permissions, and Row-Level Security (RLS)

### 2.1 The Role Hierarchy: Unified Roles vs. Users
Modern relational engines (like PostgreSQL) unify users and groups into a single primitive: the **Role**. A role can represent an individual human, an automated service, or a group containing other roles.

```sql
-- 1. Create specialized groups
CREATE ROLE read_only_group;
CREATE ROLE read_write_group;

-- 2. Grant privileges to groups
GRANT SELECT ON ALL TABLES IN SCHEMA public TO read_only_group;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO read_write_group;

-- 3. Create login identities (users) and inherit group memberships
CREATE ROLE order_service WITH LOGIN PASSWORD 'strong_scram_secret';
GRANT read_write_group TO order_service;

CREATE ROLE bi_reporting_user WITH LOGIN PASSWORD 'reporting_secret';
GRANT read_only_group TO bi_reporting_user;
```

### 2.2 Granular Permissions: From Database Down to Column
Database permissions form a strict hierarchical ladder:
$$\text{Database} \longrightarrow \text{Schema} \longrightarrow \text{Table} \longrightarrow \text{Column} \longrightarrow \text{Row}$$

To prevent unauthorized access to sensitive fields (e.g. social security numbers, password hashes), apply **Column-Level Privileges**:

```sql
-- Prevent order_service from reading sensitive billing/tax columns
REVOKE SELECT ON customer_accounts FROM order_service;
GRANT SELECT (id, name, email, status) ON customer_accounts TO order_service;
-- If order_service runs "SELECT ssn FROM customer_accounts", the engine rejects it!
```

### 2.3 Row-Level Security (RLS)
While column-level grants restrict vertical slices of a table, **Row-Level Security (RLS)** restricts horizontal slices. RLS forces the database engine to automatically inject tenant or ownership filters into every incoming query, regardless of what the application requested.

```mermaid
flowchart TD
    App["Application Query:<br/>SELECT * FROM tenant_documents"] --> Engine["Database Engine"]
    Engine --> PolicyCheck{"Evaluate RLS Policy<br/>tenant_id = current_setting('app.current_tenant')"}
    PolicyCheck --> Filter["Inject WHERE Filter Internally"]
    Filter --> DiskScan["Scan Disk: Only rows matching Tenant 42 returned"]
```

#### Implementing Multi-Tenant Isolation via RLS:
```sql
-- Step 1: Enable RLS on the multi-tenant table
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Step 2: Define the security policy
CREATE POLICY tenant_isolation_policy ON documents
    FOR ALL
    USING (tenant_id = NULLIF(current_setting('app.current_tenant_id', true), '')::UUID);

-- Step 3: Application sets session context upon checkout from pool
BEGIN;
SET LOCAL app.current_tenant_id = 'a1b2c3d4-0000-0000-0000-000000000042';

-- Even if a compromised service queries "SELECT * FROM documents",
-- the engine physically returns only rows belonging to Tenant 42!
SELECT * FROM documents;
COMMIT;
```

---

## 3. The Principle of Least Privilege (PoLP)

The **Principle of Least Privilege (PoLP)** dictates that every process, service, and user must possess only the bare minimum set of permissions necessary to execute its legitimate function, and nothing more.

```mermaid
flowchart TD
    subgraph Bad["Antipattern: Monolithic Superuser"]
        App1["App Service"] --> DB1[("Database<br/>Connected as 'postgres' / 'root'<br/>(Can DROP DB, read everything)")]
    end

    subgraph Good["Production Standard: Principle of Least Privilege"]
        App2["API Service (DML)"] -->|"SELECT, INSERT, UPDATE"| AppRole["app_user_role"]
        Migration["CI/CD Runner (DDL)"] -->|"ALTER, CREATE, MIGRATE"| DDLRole["migration_admin_role"]
        Reporting["Analytics Engine"] -->|"SELECT Only (Replica)"| ReadRole["readonly_analytics_role"]
        
        AppRole --> DB2[("Target Tables")]
        DDLRole --> DB2
        ReadRole --> DB2
    end
```

### 3.1 Why Services Must Never Connect as Table Owners
When an application connects as the owner of a table, it has implicit `DROP`, `TRUNCATE`, and `ALTER TABLE` privileges. If the application suffers an SQL injection vulnerability or runs a rogue script, the attacker can wipe out the entire table structure.

#### Hardening the Application Connection:
1. **Migrations Run by a Dedicated CI/CD Role:** The schema migration user creates tables and owns them.
2. **Runtime Services Run as Restricted Non-Owners:** The runtime application user is granted only `SELECT`, `INSERT`, `UPDATE`, and `DELETE`. It is impossible for an application SQL injection attack to drop a table because the user lacks DDL permissions.
3. **Revoke Default Public Grants:** By default, PostgreSQL grants table creation rights to `PUBLIC` in the `public` schema. Revoke this immediately:
   ```sql
   REVOKE CREATE ON SCHEMA public FROM PUBLIC;
   ```

---

## 4. SQL Injection (SQLi): Anatomy and Mechanics

**SQL Injection (SQLi)** occurs when untrusted user input is directly concatenated or formatted into a dynamic SQL statement, allowing an attacker to manipulate the query parser and execute arbitrary database commands.

```
+-----------------------------------------------------------------------------------------+
| Attacker Input:   admin' --                                                             |
+-----------------------------------------------------------------------------------------+
| Naive Dynamic SQL:                                                                      |
| SELECT * FROM users WHERE username = 'admin' --' AND password = 'secret';               |
+-----------------------------------------------------------------------------------------+
| What the Engine Executes:                                                               |
| [ Query Token ]: SELECT * FROM users WHERE username = 'admin'                           |
| [ Comment Token ]: -- (Everything after this is completely ignored by parser!)          |
+-----------------------------------------------------------------------------------------+
```

### 4.1 Primary Attack Vectors

```
1. Tautology Attacks (Always True Condition):
   Input: ' OR '1'='1
   Query: SELECT * FROM items WHERE category = '' OR '1'='1';
   Impact: Returns every row in the table, bypassing authentication or access filters.

2. Piggybacked Queries (Stacked Commands):
   Input: '; DROP TABLE audit_logs; --
   Query: SELECT * FROM users WHERE id = 42; DROP TABLE audit_logs; --';
   Impact: Destroys schemas, truncates audit logs, or inserts unauthorized admin users.

3. UNION-Based Extraction:
   Input: ' UNION SELECT null, username, password_hash FROM admin_credentials --
   Query: SELECT id, title FROM posts WHERE slug = '' UNION SELECT null, username, password_hash ...;
   Impact: Extracts sensitive data from completely unrelated tables via the HTTP response.

4. Blind Time-Based Exfiltration:
   Input: ' UNION SELECT CASE WHEN (SELECT substring(secret, 1, 1) FROM keys) = 'A' 
           THEN pg_sleep(5) ELSE pg_sleep(0) END --
   Impact: Infers private database contents character-by-character by measuring HTTP response latency.
```

---

## 5. Parameterized Queries and Prepared Statements

The industry standard defense against SQL injection is **Parameterized Queries** (implemented via **Prepared Statements**).

### 5.1 Why Parameterization Works: Parser vs. Execution Decoupling
To understand why parameterization is 100% effective against SQL injection, we must examine how relational engines execute queries:

```mermaid
sequenceDiagram
    participant App as Application Driver
    participant Parser as SQL Parser & Optimizer
    participant Exec as Execution Engine

    Note over App,Parser: 1. PREPARE Phase (Structure Compiled)
    App->>Parser: PREPARE GetUser(text) AS SELECT * FROM users WHERE email = $1
    Parser->>Parser: Validate syntax & generate Abstract Syntax Tree (AST)
    Parser->>Parser: Query Plan frozen in memory

    Note over App,Exec: 2. EXECUTE Phase (Data Transmitted Separately)
    App->>Exec: EXECUTE GetUser('malicious_attacker@evil.com\' OR 1=1 --')
    Note over Exec: Input is treated strictly as raw string data literal.
    Note over Exec: It CANNOT alter the compiled AST structure!
    Exec-->>App: Returns 0 rows (No user has that literal email string)
```

1. **Compilation Phase:** The SQL command structure is sent with placeholders (`$1`, `?`, or `:param`). The database parses the query, checks permissions, constructs the Abstract Syntax Tree (AST), and chooses the execution plan.
2. **Execution Phase:** The raw user input parameters are transmitted across the wire in a separate protocol packet. 
3. **The Guarantee:** Because the syntax tree was already compiled, user inputs can **never** alter the query structure. The database treats the input strictly as a literal value (string, integer, date).

### 5.2 Code Comparison: Vulnerable vs. Parameterized

#### Python (psycopg3 / SQLAlchemy)
```python
# VULNERABLE TO SQLi (String Concatenation / F-strings)
cursor.execute(f"SELECT * FROM accounts WHERE iban = '{user_input}'")

# SECURE: Fully Parameterized
cursor.execute("SELECT * FROM accounts WHERE iban = %s", (user_input,))
```

#### Node.js (pg driver)
```javascript
// VULNERABLE TO SQLi
await client.query(`SELECT * FROM products WHERE id = ${req.params.id}`);

// SECURE: Parameterized with $1 placeholder
await client.query('SELECT * FROM products WHERE id = $1', [req.params.id]);
```

### 5.3 The "ORM Fallacy"
Many engineers falsely assume that using an Object-Relational Mapper (ORM) like Prisma, TypeORM, Hibernate, or Django guarantees immunity from SQL injection. This is false.
* Standard ORM query builder methods (`userRepository.find({ id })`) use parameterization safely.
* However, when developers resort to **raw SQL escapes** inside ORMs (e.g., `sequelize.query(...)`, `db.$queryRawUnsafe(...)`, or `.whereRaw(...)`) with string interpolation, SQL injection vulnerabilities return immediately.

---

## 6. Encryption: In Transit and At Rest

Data security requires robust cryptographic protections throughout its lifecycle.

```mermaid
flowchart LR
    subgraph ClientTier["Application Tier"]
        App["App Server"]
    end

    subgraph Transit["Encryption In Transit (TLS 1.3)"]
        Pipe["Network Socket Encrypted<br/>(AES-256-GCM / ChaCha20)"]
    end

    subgraph DBTier["Database Engine (RAM)"]
        Buffer["Buffer Pool Pages<br/>(Decrypted in Memory)"]
    end

    subgraph Rest["Encryption At Rest"]
        TDE["Page Encryption / TDE / EBS KMS"]
        NVMe[("Encrypted NVMe SSD")]
    end

    App <--> Pipe
    Pipe <--> Buffer
    Buffer <--> TDE
    TDE <--> NVMe
```

### 6.1 Encryption In Transit (TLS / SSL)
Encrypting data between application servers, proxies, and database nodes prevents network sniffing, man-in-the-middle (MITM) tampering, and credential theft over private VPC links.

* **TLS Protocol Version:** Enforce **TLS 1.3** (or minimum TLS 1.2). Disable legacy SSLv3, TLS 1.0, and TLS 1.1.
* **Certificate Validation Modes:**
  * `require`: Encrypts traffic, but does not verify server certificate authority (vulnerable to MITM).
  * `verify-ca`: Verifies that the server certificate was signed by a trusted Certificate Authority (CA).
  * `verify-full` (Production Standard): Verifies both the CA and confirms that the database server hostname matches the Common Name (CN) or SAN in the certificate.

### 6.2 Encryption At Rest: The Three Tiers
Encryption at rest ensures that if a physical hard drive is stolen, an unattached SAN volume is accessed, or an unencrypted volume snapshot is leaked, the raw data cannot be read.

| Tier | Architectural Layer | How It Works | Protects Against | Does NOT Protect Against |
| :--- | :--- | :--- | :--- | :--- |
| **Storage / Volume Level** | AWS EBS / LUKS / dm-crypt | The hypervisor or OS kernel encrypts entire physical disk blocks using AES-XTS-256. | Stolen physical SSDs, decommissioned drives, raw cloud snapshot theft. | Rogue database users, SQL injection, OS root compromise. |
| **Transparent Data Encryption (TDE)** | DBMS Engine Level | The database engine encrypts 8 KB / 16 KB data pages before writing to disk and decrypts them when read into the Buffer Pool. | Offline file scraping, direct reading of table heap files and WAL segments. | Authorized database connections reading memory pages. |
| **Application-Level Encryption (ALE)** | Application Service | The application encrypts sensitive column values (e.g. credit card number) before sending them to the database. | Compromised database superusers, rogue DBAs, compromised backups. | Cannot search or run `WHERE` range queries on encrypted ciphertext. |

### 6.3 Envelope Encryption & KMS Architecture
Enterprise databases avoid storing master encryption keys on the database server itself. They rely on **Envelope Encryption**:

```mermaid
flowchart TD
    KMS["Cloud KMS / HSM (Hardware Security Module)"] -->|"Stores Unexportable"| KEK["Key Encryption Key (Master Key)"]
    KEK -->|"Encrypts"| DEK["Data Encryption Key (Plaintext DEK)"]
    
    subgraph Host["Database Server"]
        EncDEK["Encrypted DEK (Stored on Disk)"]
        RAM_DEK["Plaintext DEK (Held in volatile RAM only)"]
        Pages["Database Disk Pages"]
    end

    EncDEK -->|"Startup: Send to KMS"| KMS
    KMS -->|"Decrypts with KEK & Returns"| RAM_DEK
    RAM_DEK -->|"Encrypts/Decrypts Pages"| Pages
```

1. **Data Encryption Key (DEK):** A symmetric AES-256 key used locally by the database engine to rapidly encrypt and decrypt table pages.
2. **Key Encryption Key (KEK / Master Key):** Stored securely inside a centralized Hardware Security Module (AWS KMS, HashiCorp Vault, Google Cloud KMS).
3. **The Workflow:** At server startup, the database asks the KMS to decrypt its local encrypted DEK. The plaintext DEK is held only in RAM and never written to disk. If the database server is stolen or backed up, the data is unreadable without KMS access.

---

## 7. Dynamic Secrets Management & Credential Lifecycles

### 7.1 The Anti-Pattern: Static Passwords
Historically, applications stored static credentials in configuration files or container environment variables:
`DATABASE_URL=postgres://app_user:SuperSecretPassword123@db.prod.internal:5432/orders`

**Why Static Passwords Fail in Production:**
* **Credential Sprawl:** Passwords get copied into Git histories, CI/CD pipeline logs, docker layers, and developer machines.
* **Zero Rotation:** Rotating a static password requires synchronized deployments across dozens of microservices, leading to teams avoiding rotations for years.
* **Leak Impact:** A leaked static password remains valid indefinitely until manual remediation occurs.

### 7.2 The Solution: Dynamic, Short-Lived Ephemeral Credentials
Modern zero-trust architectures use credential brokers (such as **HashiCorp Vault** or **AWS IAM Database Authentication**) to issue on-demand, unique credentials that automatically expire:

```mermaid
sequenceDiagram
    participant App as Application Service
    participant Vault as HashiCorp Vault / Secret Broker
    participant DB as Database Engine

    App->>Vault: 1. Authenticate (via Kubernetes ServiceAccount / IAM)
    Vault->>Vault: 2. Authorize policy for "orders-service"
    Vault->>DB: 3. CREATE ROLE ephemeral_user WITH LOGIN PASSWORD 'temp_pwd' VALID UNTIL '1 hour'
    Vault->>DB: 4. GRANT app_role TO ephemeral_user
    Vault-->>App: 5. Return temporary username & password (TTL: 1 hour)
    App->>DB: 6. Connect & execute queries
    Note over Vault,DB: 1 hour later: Vault automatically issues DROP ROLE!
```

### 7.3 Zero-Downtime Dual-User Credential Rotation
If using traditional credentials, automate rotation using the **Dual-User Pattern**:

```
State 1: Application uses User_A.
Step 1:  Provision User_B with identical roles and permissions in the database.
Step 2:  Deploy new secrets to configuration manager; App gracefully switches pool to User_B.
Step 3:  Verify User_A has zero active connections via pg_stat_activity.
Step 4:  Drop User_A.
Result:  100% automated rotation without a single millisecond of downtime.
```

---

## 8. Sensitive Data Handling: PII, PCI-DSS, and Hashing

Not all data should be treated equally. Regulatory standards (GDPR, HIPAA, PCI-DSS, SOC 2) enforce strict operational boundaries for Personally Identifiable Information (PII) and financial tokens.

### 8.1 Passwords: One-Way Cryptographic Hashing
Passwords must **never** be encrypted with reversible encryption. They must be transformed using dedicated, slow, memory-hard one-way **password hashing functions**:

| Algorithm | Status | Architectural Suitability | Why It Is Chosen |
| :--- | :--- | :--- | :--- |
| **Argon2id** | **Gold Standard** | User password storage in modern applications. | Winner of the Password Hashing Competition; resistant to both GPU and ASIC brute-force attacks via configurable memory, time, and parallelism parameters. |
| **bcrypt** | **Industry Standard** | Legacy and widespread production systems. | Adaptive work factor (cost parameter) allows scaling computation time as CPUs get faster. |
| **PBKDF2** | **Compliant Standard** | FIPS-140 compliance environments. | Widely approved by government standards; relies on iterative HMAC (SHA-256). |
| **MD5 / SHA-1 / SHA-256** | **PROHIBITED** | NEVER use for passwords. | General-purpose fast digest algorithms. GPUs can calculate billions of SHA-256 hashes per second, making rainbow table and brute-force attacks trivial. |

```python
# Secure Password Hashing Example using Argon2id
from argon2 import PasswordHasher

ph = PasswordHasher(time_cost=3, memory_cost=65536, parallelism=4)
# Generates a salted, memory-hard hash string containing version and parameters
stored_hash = ph.hash("UserSecretPassword")

# Verification
ph.verify(stored_hash, "UserSecretPassword") # Returns True
```

### 8.2 Application-Level Field Encryption (Deterministic vs. Randomized)
When storing sensitive PII that must be recovered in plaintext (e.g. Tax IDs, Bank Account IBANs), use **AES-256-GCM** authenticated encryption:

* **Randomized Encryption (Standard):** Every encryption of the same value uses a unique random Initialization Vector (IV / Nonce).
  * Encrypting `"123-45-6789"` produces a completely different ciphertext every time.
  * *Advantage:* Highly secure; immune to frequency analysis attacks.
  * *Disadvantage:* The database cannot run `WHERE ssn = 'ciphertext'` index lookups.
* **Deterministic Encryption (Searchable):** Uses a pseudo-random IV derived from the plaintext and a dedicated blind index key.
  * Encrypting `"123-45-6789"` always produces the exact same ciphertext.
  * *Advantage:* Enables equality searches (`WHERE ssn_hash = '...'`).
  * *Disadvantage:* Vulnerable to statistical frequency analysis if the key space is small.

---

## 9. Data Masking and Redaction

Engineers frequently need realistic data to debug issues in staging environments or run analytical reporting. Exposing production PII to non-production environments violates compliance laws and dramatically expands your attack surface.

```mermaid
flowchart LR
    Prod[("Production Database<br/>Contains Real PII")] --> Pipeline["Sanitization Pipeline<br/>(Static Data Masking)"]
    Pipeline --> Stage[("Staging Database<br/>Anonymized / Faked Data")]
    
    Dev["Developers / QA"] --> Stage
    Analytics["Analytics Team"] -->|"Dynamic Masking"| ReadonlyProd[("Reporting Replica")]
```

### 9.1 Static Data Masking (SDM)
Used when copying data from production to lower environments (Dev, QA, Staging). The sanitization process permanently destroys sensitive data before loading it into target databases:

```sql
-- Sanitization ETL script for staging databases
UPDATE users 
SET 
    email = 'user_' || id || '@staging.internal',
    full_name = 'Test User ' || id,
    phone_number = '+1-555-01' || LPAD((id % 100)::text, 2, '0'),
    password_hash = '$argon2id$v=19$m=65536...dummy_test_hash...',
    ssn = NULL;
```

### 9.2 Dynamic Data Masking (DDM)
Used in real-time on live production or reporting databases. The data remains fully intact on disk, but queries executed by unprivileged roles have sensitive fields obscured on the fly:

```sql
-- PostgreSQL Dynamic Masking using Custom Views or Security Extensions
CREATE VIEW secure_customer_view AS
SELECT 
    id,
    name,
    CASE 
        WHEN pg_has_role(current_user, 'billing_admin', 'MEMBER') THEN credit_card_num
        ELSE 'XXXX-XXXX-XXXX-' || RIGHT(credit_card_num, 4)
    END AS credit_card_num,
    CASE 
        WHEN pg_has_role(current_user, 'support_agent', 'MEMBER') THEN email
        ELSE LEFT(email, 2) || '****@' || SPLIT_PART(email, '@', 2)
    END AS email
FROM customer_records;
```

---

## 10. Comprehensive Audit Logging & SIEM Integration

An audit trail provides forensic accountability. If a breach occurs, audit logs answer: *Who accessed what data, from where, and at what exact millisecond?*

### 10.1 What to Audit vs. What NEVER to Audit

```
+---------------------------------------+---------------------------------------+
| MUST Audit (Security Events)          | NEVER Audit (Sensitive Leaks)         |
+---------------------------------------+---------------------------------------+
| - Failed authentication attempts      | - Passwords submitted in queries      |
| - Schema modifications (DDL)          | - Plaintext Credit Card Numbers       |
| - Permission changes (GRANT/REVOKE)   | - Sensitive API Tokens or Private Keys|
| - Superuser / administrative actions  | - Health check queries every 500 ms   |
| - Access to restricted tables (PII)   |   (Destroys disk storage)             |
+---------------------------------------+---------------------------------------+
```

### 10.2 Database Auditing Tools: `pgaudit`
Standard database logs are designed for debugging performance, not compliance. Production PostgreSQL databases deploy **`pgaudit`** (PostgreSQL Audit Extension):

```ini
# postgresql.conf Hardening
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'role, ddl, write'
pgaudit.log_catalog = off
pgaudit.log_level = 'log'
pgaudit.log_parameter = on
```

When an application updates a record, `pgaudit` outputs an immutable structured log record:
```json
{
  "timestamp": "2026-10-18T20:55:01Z",
  "audit_type": "SESSION",
  "statement_id": 4012,
  "substatement_id": 1,
  "class": "WRITE",
  "command": "UPDATE",
  "object_type": "TABLE",
  "object_name": "public.orders",
  "user": "app_user",
  "database": "production_db",
  "client_ip": "10.0.12.44",
  "statement": "UPDATE orders SET status = $1 WHERE id = $2"
}
```

### 10.3 Immutable Log Forwarding (WORM Storage)
Local log files can be modified or deleted by an attacker who gains root access. Production architectures stream logs immediately off the node to central SIEMs (Datadog, Splunk, Elastic) and archive them to **WORM (Write Once, Read Many)** object storage:

```mermaid
flowchart LR
    Engine["Database Engine"] -->|"Writes Audit Stream"| LocalLog["Local Log Pipe"]
    LocalLog --> Agent["FluentBit / Vector Agent"]
    Agent -->|"TLS Stream"| SIEM["SIEM (Splunk / Datadog)"]
    Agent -->|"Archive (Object Lock)"| S3[("WORM S3 Bucket<br/>(Immutable for 7 Years)")]
```

---

## 11. Backup Security and Access Controls

The most secure database in the world can be completely undermined by a single unsecured backup file. Attackers frequently bypass hardened database servers by finding **unencrypted, world-readable `.sql` dumps in unprotected cloud buckets**.

```mermaid
flowchart TD
    subgraph SecureProd["Hardened Production Database"]
        DB["PostgreSQL Node<br/>- Strict Firewall<br/>- SCRAM Auth<br/>- mTLS"]
    end

    subgraph TheLeak["The Forgotten Vector"]
        Dump["pg_dump output: backup.sql"]
        S3["Public / Unencrypted S3 Bucket"]
        Hacker["Attacker Downloads Backup Dump<br/>(Extracts entire company database in 10s!)"]
    end

    DB -->|"Nightly Backup"| Dump
    Dump --> S3
    S3 -.-> Hacker
```

### 11.1 The Golden Rules of Backup Security:
1. **Client-Side Encryption Before Transmission:**
   Never stream an unencrypted backup across the network. Encrypt backups on the fly using asymmetric GPG keys or KMS client-side encryption:
   ```bash
   pg_dump -Fc mydb | gpg --encrypt --recipient ops-backup-key@company.com > mydb_encrypted.dump
   ```
2. **Dedicated Backup Vault Account:**
   Store database backups in a completely **isolated cloud account** separate from the primary production environment. If the primary cloud account is compromised, the attacker cannot delete or view the backups in the vault account.
3. **S3 Object Lock (WORM Immutability):**
   Enable Object Lock in Compliance Mode on backup buckets. This physically prevents anyone (including root cloud administrators) from deleting or overwriting backups for a predefined retention period (e.g. 30 to 90 days), providing guaranteed immunity against ransomware destruction.
4. **Scrubbed Backups for Non-Production:**
   Never restore raw production backups into staging or test environments. Always run a sanitization pipeline to mask PII prior to saving snapshots intended for developer usage.

---

## 12. Master Security Reference: Threat Modeling & Defenses

Review this master threat-modeling reference when auditing database architectures:

| Threat / Attack Vector | Real-World Impact | Primary Defense Layer | Concrete Implementation |
| :--- | :--- | :--- | :--- |
| **SQL Injection (SQLi)** | Arbitrary data exfiltration, schema destruction, remote code execution. | Query Compilation Layer | Prepared statements with parameterized inputs; zero string concatenation. |
| **Credential Sniffing / MITM** | Cleartext password and query interception over network. | Network Transport Layer | Enforce TLS 1.3 with `sslmode=verify-full` and strict CA validation. |
| **Credential Sprawl / Static Leaks** | Permanent breach entry point if passwords leak to Git or logs. | Identity & Secrets Layer | Dynamic short-lived credentials via HashiCorp Vault or Cloud IAM DB Auth. |
| **Accidental Table Drop (DDL Bug)** | Production outage, mass data deletion. | Authorization / PoLP | Runtime application connections granted only DML (`SELECT`, `INSERT`); DDL restricted to CI/CD. |
| **Cross-Tenant Data Bleed** | Tenant A queries and views Tenant B's confidential records. | Data Isolation Layer | Row-Level Security (RLS) policies enforcing `tenant_id` session filters. |
| **Stolen Disk / Volume Snapshot Leak** | Raw database files read offline by malicious actors. | Storage Layer | Volume-level AES-256 encryption (EBS KMS) combined with Transparent Data Encryption (TDE). |
| **Database Superuser Compromise** | Attacker reads sensitive customer credit cards or tax IDs. | Application Cryptographic Layer | Application-Level Encryption (ALE) with AES-256-GCM before writes; DB holds only ciphertext. |
| **Ransomware Backup Deletion** | Attackers encrypt primary DB and erase historical backups. | Backup Storage Layer | S3 Object Lock (Compliance Mode) in an isolated, multi-factor protected vault account. |
| **Insider Data Tampering** | Malicious employee covertly modifies balances or permissions. | Audit & Forensic Layer | `pgaudit` extension streaming structured logs to an append-only, immutable SIEM. |

---

## 13. Summary & Production Security Checklist

Audit your database deployment against this production readiness checklist:

* [ ] **Network Isolation:** Database binds to private VPC subnets only; public internet access (`0.0.0.0/0`) is blocked at the security group and firewall level.
* [ ] **Transport Encryption Enforced:** Database server requires TLS connections (`hostssl`); clients connect with `verify-full` certificate checking.
* [ ] **Modern Auth Enforced:** Legacy MD5/plain passwords are disabled; SCRAM-SHA-256, mTLS, or Cloud IAM authentication is mandated.
* [ ] **Least Privilege Applied:** Application services do not own tables; DDL (`CREATE`, `ALTER`, `DROP`) is strictly segregated from application DML accounts.
* [ ] **Public Schema Hardened:** Default permissions on `SCHEMA public` revoked from `PUBLIC`.
* [ ] **Zero Dynamic SQL:** Codebases strictly enforce parameterized queries; raw SQL string interpolation is flagged and blocked by static analysis linters (SAST).
* [ ] **Row-Level Security Active:** Multi-tenant databases enforce RLS policies verifying tenant identifiers on every query.
* [ ] **Encryption at Rest Active:** Disk volumes are encrypted via KMS; keys are rotated automatically on a recurring schedule.
* [ ] **Audit Logging Enabled:** `pgaudit` captures DDL, role modifications, and sensitive table access; logs ship to an immutable remote SIEM.
* [ ] **Backups Encrypted & Locked:** Backups are encrypted before egress and stored in an isolated vault account with WORM Object Lock enabled.

---

⬅️ **[Previous: 10. Data Consistency in Distributed Systems](10_Data_Consistency.md)** | 🏠 **[Back to TOC](README.md)** | **[Next: 12. Backend–Database Integration ➡️](12_Backend_Database_Integration.md)**
