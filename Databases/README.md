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
