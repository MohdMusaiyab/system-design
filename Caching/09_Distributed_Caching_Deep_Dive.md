# 9. Distributed Caching Deep-Dive (Redis / Memcached)

We have decided to use a distributed cache (Redis or Memcached) because our working set is too large to fit into local heaps, or because we need cross-instance consistency. But now we must actually operate this cluster.

### 9.1 Sharding / Consistent Hashing (How data is distributed across nodes)

**The Problem:**
We have 100GB of cache data, but one Redis instance only has 10GB of RAM. We need to split our data across 10 instances. How do we decide which instance holds `user:123`?

**The Naive Approach (Modulo Hashing):**
We compute `hash(key) % N` where N is the number of nodes. If we have 10 nodes, `user:123` goes to node 3.

**Why we avoid it:**
This approach is a disaster in production. If we add a new node (scaling up from 10 to 11), N changes from 10 to 11. Suddenly, `hash(key) % 10` is different from `hash(key) % 11`. For every existing key, the hash calculation points to a completely different node. This means we effectively invalidate 100% of our cache during a scale-up. Every single request misses, and our database gets hammered.

**The Production Solution (Consistent Hashing):**
Instead of a single hash, we imagine a ring of values from `0` to `2^32 - 1`. We hash each node (e.g., `redis-node-1`) to several points on this ring (these are "virtual nodes"). When we want to store `user:123`, we hash the key to a position on the ring, and we walk clockwise to find the nearest node.

**Why this changes everything:**
When we add a new node (node 11), we only need to map it to a few points on the ring. Only the keys that hash to the segment between the old nodes and the new node will be remapped. If we have 100 virtual nodes per physical node, adding a new node only invalidates roughly `1 / (N+1)` of our cache keys. This is a massive reduction in database load during scaling.

**The Real-World Implementation (Redis Cluster):**
Redis Cluster does not use the classic ring-based Consistent Hashing. Instead, it uses **Hash Slots**. There are 16,384 slots. Each Redis node is responsible for a subset of these slots (e.g., Node A: 0–5000, Node B: 5001–10000, Node C: 10001–16383). To find the node for a key, we compute `CRC16(key) % 16384`.

**The Trade-off:**
- **Consistent Hashing (Ring):** Great for dynamic nodes, but requires a client-side library to handle the ring logic.
- **Redis Cluster (Slots):** Simpler to reason about (exact slot ranges). Resharding is a manual process (`redis-cli --cluster reshard`) where we move slots between nodes. During resharding, the cluster is still available, but the specific keys being moved will return a `MOVED` error, telling the client to redirect to the correct node.

> **💡 What we generally do:**
> We use Redis Cluster with Hash Slots when we need high availability and automatic failover. For simpler setups, we use client-side sharding (like Twemproxy or lettuce with Consistent Hashing) to avoid the operational overhead of a cluster.

---

### 9.2 Replication (Read replicas for scale)

**The Problem:**
We have a primary (master) Redis node that handles all writes and reads. Our application traffic grows, and the primary node's CPU spikes to 90% just serving reads. We need to offload the read traffic.

**The Solution:**
We set up Replicas. The primary node replicates all data asynchronously to one or more replica nodes. Our application sends all writes to the primary, but routes GET requests to the replicas.

**Why we use it:**
It horizontally scales our read throughput. If we have 1 primary and 3 replicas, we can handle 4x the read QPS. It also serves as a disaster recovery mechanism—if the primary fails, we can promote a replica to primary.

**The Trade-offs:**
- **Asynchronous Replication:** Data is replicated to replicas after the primary returns success to the client. This creates a replication lag (usually sub-millisecond, but can grow under heavy write load). This means replicas serve stale data compared to the primary.
- **The "Master-Split" problem:** If our application writes to the primary and immediately tries to read from a replica, it might not see its own write yet. This violates Read-Your-Writes Consistency.

> **💡 What we generally do:**
> We use replicas strictly for heavy read workloads where a few milliseconds of staleness is acceptable (e.g., product listings, analytics dashboards). For critical reads (like checking balance), we force the application to read from the primary.

---

### 9.3 Persistence mechanisms (RDB snapshots vs. AOF logs)

**The Mental Model:**
Redis lives in memory. If the server restarts (planned maintenance or crash), we lose all cached data. This is usually fine—it's just a cache—but sometimes we want to speed up cold starts, or we are using Redis as a lightweight primary store (like session storage).

**Option A: RDB (Redis Database) Snapshots**
Redis forks a child process to write the entire dataset to a binary `dump.rdb` file on disk at scheduled intervals (e.g., every 60 seconds if 1000 keys changed).
- **Pros:** Compact, small file size, fast restart (loads a single file).
- **Cons:** Data loss between snapshots. If Redis crashes 30 seconds after the last snapshot, we lose those 30 seconds of writes.

**Option B: AOF (Append-Only File)**
Redis logs every single write operation to a file (`appendonly.aof`) in real-time. We can configure how often we `fsync` to disk (always, everysec, no).
- **Pros:** Minimal data loss (default `everysec` loses at most 1 second of data). Very durable.
- **Cons:** AOF files can grow extremely large and slow down restart time (Redis has to replay the entire log). This also uses more disk I/O.

> **🛠️ The Production Pattern:**
> We enable both RDB and AOF. We use AOF with `appendfsync everysec` as our primary durability mechanism. We use RDB snapshots as a backup for disaster recovery (e.g., backing up to S3 daily). When Redis restarts, it plays the AOF log (which is more complete) rather than the RDB snapshot.

**The Critical Trade-off:**
Do we need persistence at all? If we are purely using Redis as a database cache where a miss just goes to the primary DB, we might disable persistence entirely to maximize performance. For session storage or rate-limiting counters, we enable AOF because losing those counters could cause security or availability issues.

---

### 9.4 Serialization (What format do we store?)

**The Problem:**
We are storing a `User` object (with nested addresses, roles, preferences). We cannot store a Java/Python object directly in Redis. We must convert (serialize) it to a sequence of bytes.

**The Options:**

| Format | Characteristics | Example |
| :--- | :--- | :--- |
| **JSON** | Human-readable, self-describing, no schema needed. | `{"id":1, "name":"Alice"}` |
| **MessagePack** | Binary, compact, schema-less. Similar to JSON but smaller and faster. | Hex bytes. |
| **Protocol Buffers** | Binary, requires a `.proto` schema. Extremely compact and fast. | Predefined fields. |
| **Language Specific** | (e.g. Pickle/Java Ser) Language-specific. Fragile on version mismatch. Slow. | Byte stream with metadata. |

**Why we care about Serialization:**
- **Memory Footprint:** A JSON string takes significantly more bytes than a Protobuf binary. For a 100GB Redis cluster, choosing JSON over Protobuf might waste 30-40GB of memory.
- **CPU Overhead:** Serializing/Deserializing JSON on every read/write costs CPU cycles. Protobuf and MessagePack are significantly faster.
- **Network Bandwidth:** Sending larger payloads over the network increases latency.

**What we generally do in production:**
- **For internal APIs (services talking to services):** We use Protocol Buffers (Protobuf). We define a strict schema, and the serialization is incredibly efficient. The downside is that we must manage schema evolution. 
- **For public-facing APIs:** We use MessagePack. It gives us the efficiency of binary without requiring a strict schema contract.
- **For debugging and small metadata:** We use JSON. It's human-readable, making it easy for engineers to inspect keys via `redis-cli`. However, we set a hard rule: JSON is only for keys under 1KB. For large objects, we force Protobuf.

**The Critical Nuance (Compression):**
Sometimes, even after serialization, the payload is still large. We add a compression layer (like LZ4 or Snappy) after serialization and before storing in Redis. This reduces memory by 30-50% at the cost of CPU cycles. We generally only do this for objects larger than 5KB.

---

⬅️ **[Previous: 8. Cache Consistency](08_Cache_Consistency.md)** | 🏠 **[Back to TOC](README.md)** | **[Next: 10. Production Failure Modes ➡️](10_Production_Failure_Modes.md)**
