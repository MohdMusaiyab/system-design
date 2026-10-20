# 🗄️ System Design: Databases

Welcome to the **Databases** master module. Understanding how data is structurally stored, queried, partitioned, and replicated is arguably the most critical component of distributed system design.

In this directory, we dive deeply into the internal architectures, B-Tree vs LSM-Tree storage engines, ACID compliance mechanisms, and specific use-cases of the industry's most popular databases.

## 📖 Curriculum

1.  **[Database Fundamentals](01_Database_Fundamentals.md):** What is a DB/DBMS, three-tier database architecture, storage engines, SQL vs NoSQL, OLTP vs OLAP, and workload-based database selection.
2.  **[Data Modeling & Schema Design](02_Data_Modeling_And_Schema_Design.md):** Entities and relationships, keys, constraints, 1NF/2NF/3NF normalization, denormalization trade-offs, zero-downtime migrations, and audit patterns.
3.  **[Database Internals](03_Database_Internals.md):** On-disk pages and slotted storage, buffer pool management (LRU vs CLOCK), B+ Trees, WAL, checkpoints and dirty pages, process architecture, and the read vs. write paths.
4.  **[Transactions & ACID](04_Transactions_And_ACID.md):** Atomicity to Durability, transaction lifecycles, SQL isolation levels, Dirty/Non-Repeatable/Phantom reads, Lost Updates, Write Skew, PCC vs. OCC, MVCC mechanics, deadlocks, and long-running transaction mitigation.
5.  **[Database Concurrency & Locking](05_Database_Concurrency_And_Locking.md):** Granularity (row vs. table), Shared (S) vs. Exclusive (X) locks, Intent locks, lock escalation, race conditions, atomic updates, Optimistic Locking (versioning), Pessimistic Locking (`SELECT FOR UPDATE`, `NOWAIT`, `SKIP LOCKED`), wait-for graphs, and safe concurrency patterns.
6.  **[Database Performance & Optimization](06_Database_Performance.md):** Query latency vs. throughput, connection overhead, pooling architecture & sizing formulas, saturation limits, slow query analysis (EXPLAIN ANALYZE), cache hit ratios, hardware bottlenecks, batch operations, bulk COPY, and capacity planning.
7.  **[Replication & Read Scaling](07_Replication_And_Read_Scaling.md):** Availability vs. scale, Primary-Replica (Leader-Follower), sync vs. async vs. semi-sync replication, physical causes of replication lag, read/write splitting, Read-Your-Own-Writes anomalies & sticky routing, automated failover (Patroni/Raft), and split-brain fencing (STONITH).
8.  **[Sharding & Partitioning](08_Partitioning_And_Sharding.md):** Single-node partitioning vs. multi-node sharding, vertical vs. horizontal partitioning, Range/Hash/List strategies, shard key selection criteria, hot partitions & celebrity problems, cross-shard scatter-gather queries, consistent hashing with virtual nodes, live dual-write resharding migrations, and when sharding is actually necessary.
9.  **[Database Reliability & Recovery](09_Database_Reliability_And_Recovery.md):** RPO and RTO objectives, HA vs. DR scopes, why replication is not a backup, full vs. diff vs. incremental backups, physical vs. logical (pg_dump), Point-In-Time Recovery (PITR) with continuous WAL streaming, automated restore verification pipelines, consensus failover, data corruption defenses (torn pages & checksums), and AZ vs. regional failure blast radius.
10. **[Data Consistency in Distributed Systems](10_Data_Consistency.md):** The spectrum of consistency (Linearizability to Eventual), client-centric models (Read-After-Write, Monotonic Reads), CAP & PACELC theorems, Two-Phase Commit (2PC) blocking flaws, Sagas (Choreography vs. Orchestration), compensating transactions, the Dual-Write Problem & Transactional Outbox pattern, idempotency keys, and cross-service consistency.
11. **[Database Security](11_Database_Security.md):** Defense-in-depth architecture, AuthN (SCRAM, mTLS, IAM) vs. AuthZ (RBAC), Row-Level Security (RLS), Principle of Least Privilege, SQL injection mechanics and AST parameterization, encryption at rest/in transit, envelope encryption (DEK/KEK/KMS), dynamic secrets management, password hashing (Argon2id), data masking (DDM vs. SDM), `pgaudit`, and immutable WORM backup security.
12. **[Backend–Database Integration](12_Backend_Database_Integration.md):** ORM internal architecture (AST, Identity Map, Unit of Work) vs. query builders vs. raw SQL, connection lifecycle and leak prevention, cross-service transaction propagation, N+1 query elimination (joins vs. batch prefetching), keyset/cursor pagination, bulk ingestion via COPY, safe transient retries (full jitter), pool sizing formulas ((Cores * 2) + Spindles), repository patterns, timeout layering, and graceful circuit-breaker degradation.
13. **[Database Design in System Design](13_Database_Design_In_System_Design.md):** Quantitative back-of-the-envelope estimation (QPS, TPS, 5-year storage capacity with index/replication multipliers, network bandwidth), access-pattern-driven schema design, read-heavy vs write-heavy architectures, SQL vs NoSQL decision spectrum, physical database bottlenecks (CPU, RAM, IOPS, Network), evolutionary 7-stage scaling ladder, multi-region replication (Active-Passive vs Active-Active vs Spanner consensus), automated data tiering and archival, Event Sourcing & CQRS, PACELC trade-offs, and case studies (URL shortener, Twitter timeline, Uber driver tracking).
14. **[Advanced Database Topics Worth Knowing](14_Advanced_Database_Topics.md):** Change Data Capture (CDC) via WAL tailing (Debezium), inverted indexes for full-text search (`tsvector`/GIN), vector embeddings and ANN indexing (HNSW vs IVFPQ, `pgvector`), time-series engines (hypertables, delta-of-delta compression), search engines vs databases (dual-store architecture), object storage (S3 presigned URLs) vs DB BLOBs, analytical columnar data warehouses (OLAP/SIMD), materialized views and IVM, database triggers vs application logic, multi-tenant SaaS isolation models (DB-per-tenant vs Schema-per-tenant vs RLS), and Distributed SQL (CockroachDB, YugabyteDB, Spanner).

## 🗄️ Database Architectures Explored

### 1. Relational (SQL) Databases
*   **[PostgreSQL](PostgreSQL/README.md):** The world's most advanced open-source relational database. Deep dives into MVCC, Write-Ahead Logs (WAL), and complex joins.
*   **[SQLite](SQLite/README.md):** The most deployed database engine globally. Exploring embedded serverless architecture and file-locking mechanics.

### 2. NoSQL (Document & Wide-Column)
*   **[MongoDB](MongoDB/README.md):** The leading Document (BSON) database. Exploring Replica Sets, Sharding Architectures, and horizontal scaling mechanics.
*   **[Cassandra](Cassandra/README.md):** The ultimate highly-available Wide-Column store. Understanding the Dynamo paper, decentralized masterless scaling, and tunable consistency.

### 3. In-Memory (Key-Value)
*   **[Redis](Redis/README.md):** Beyond simple caching. Exploring Redis persistence (RDB/AOF), Pub/Sub, and single-threaded event loops. *(Note: Extensive practical applications are also covered in the [Caching](../Caching/README.md) module).*

---
_Select a database engine above to enter its specific architectural deep dive._
