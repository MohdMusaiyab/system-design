# 👁️ Apache Cassandra Architecture

Welcome to the **Cassandra** design deep dive. Engineered originally at Facebook (based on Amazon's Dynamo paper and Google's Bigtable), Cassandra is built for one purpose: zero downtime and massive, multi-datacenter horizontal scale.

## 📖 Curriculum

*(Chapters will be populated here during the module build-out)*

1.  **Masterless Ring Architecture:** How peer-to-peer decentralized nodes operate without a single point of failure using Gossip protocols.
2.  **Storage Physics:** Log-Structured Merge (LSM) Trees, Memtables, and SSTable disk flushing.
3.  **Partitioning:** Consistent Hashing algorithms heavily routing data across the global ring.
4.  **Tunable Consistency:** The mathematical formulas balancing Quorum reads/writes against network partition failures (CAP Theorem).
5.  **Tombstones & Compaction:** The mechanical garbage collection required in distributed immutable logs.

---
⬅️ **[Back to Database Root](../README.md)**
