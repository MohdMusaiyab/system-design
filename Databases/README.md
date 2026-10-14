# 🗄️ System Design: Databases

Welcome to the **Databases** master module. Understanding how data is structurally stored, queried, partitioned, and replicated is arguably the most critical component of distributed system design.

In this directory, we dive deeply into the internal architectures, B-Tree vs LSM-Tree storage engines, ACID compliance mechanisms, and specific use-cases of the industry's most popular databases.

## 📖 Curriculum

1.  **[Database Fundamentals](01_Database_Fundamentals.md):** What is a DB/DBMS, three-tier database architecture, storage engines, SQL vs NoSQL, OLTP vs OLAP, and workload-based database selection.
2.  **[Data Modeling & Schema Design](02_Data_Modeling_And_Schema_Design.md):** Entities and relationships, keys, constraints, 1NF/2NF/3NF normalization, denormalization trade-offs, zero-downtime migrations, and audit patterns.

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
