# 🐘 PostgreSQL Architecture & Internals

Welcome to the **PostgreSQL** architectural deep dive. As the world's most advanced open-source relational database, Postgres relies heavily on strict ACID compliance paradigms, complex relational modeling, an extensible catalog-driven runtime, and immense vertical scale capabilities.

---

## 📖 Curriculum

1. **[Introduction to PostgreSQL](01_Introduction_To_PostgreSQL.md):** Object-Relational philosophy, open-source governance, logical hierarchy (Instance $\rightarrow$ Database $\rightarrow$ Schema), multi-process architecture, shared memory vs. local `work_mem`, physical on-disk layout (`$PGDATA`), native JSONB and extensions (`pgvector`, PostGIS), connection multiplexing with PgBouncer, and essential diagnostic catalogs.
2. **Core Storage Architecture & Page Layout:** Physical slotted pages (8 KB), Page Header, Item Identifiers (Line Pointers), Tuple Header (t_xmin, t_xmax, t_ctid), TOAST tables, Free Space Map (FSM), and Visibility Map (VM).
3. **Indexing Mechanics:** B-Trees (Lehman-Yao algorithm), GIN (Generalized Inverted Indexes for JSONB/FTS), GiST (Generalized Search Trees for PostGIS), BRIN (Block Range Indexes for time-series), and HNSW (vector embeddings).
4. **Concurrency Control & MVCC Deep Dive:** Multi-Version Concurrency Control internals, Tuple immutability, Transaction ID wraparound, Dead Tuples, Heap-Only Tuples (HOT), and the internal mechanics of `VACUUM` and `ANALYZE`.
5. **Durability & Crash Recovery:** Write-Ahead Logging (WAL) mechanics, checkpoints, Full Page Writes (FPW), LSN sequencing, and the ARIES recovery process.
6. **Scaling & High Availability:** Physical Streaming Replication (sync vs. async), Logical Replication, Connection Pooling (PgBouncer in transaction mode), automated failover (Patroni/etcd), and zero-downtime schema migrations.

---

⬅️ **[Back to Database Root](../README.md)**
