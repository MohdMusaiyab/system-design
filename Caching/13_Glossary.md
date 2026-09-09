# 13. Glossary and Terminology

Below is the definitive glossary of caching terminology used throughout this guide. Think of this as your quick-reference sheet for system design interviews and production discussions.

| Term | Category | Definition / Our Mental Model |
| :--- | :--- | :--- |
| **Latency Gap** | Fundamentals | The physical speed difference between CPU operations (nanoseconds) and disk/network I/O (milliseconds). Caching exists primarily to bridge this gap. |
| **Locality of Reference** | Fundamentals | The statistical property that makes caching effective. **Temporal:** we access the same key repeatedly. **Spatial:** we access nearby keys consecutively. |
| **80/20 Rule (Pareto)** | Fundamentals | The observation that roughly 80% of requests hit only 20% of the data. This defines our "working set" and justifies why we don't need to cache everything. |
| **Hit Ratio / Miss Ratio** | Metrics | `Hit Ratio = Hits / (Hits + Misses)`. The primary health metric for a cache. A high ratio (>90%) means the cache is effectively offloading the database. |
| **Working Set** | Sizing | The subset of data actively accessed within a given time window (e.g., the last 5 minutes). The cache must be large enough to hold the working set to maintain a high hit ratio. |
| **Page Cache / Buffer Cache** | OS-Level | The OS-managed cache that stores recently accessed disk blocks in RAM. It operates automatically at the filesystem level, beneath our application. |
| **CDN (Content Delivery Network)** | Infrastructure | A geographically distributed cache that stores static assets (images, CSS, JavaScript) at edge locations close to users to reduce global latency. |
| **DNS Caching** | Infrastructure | The caching of domain-name-to-IP-address mappings at the OS, browser, or resolver level to avoid repeated network lookups. |
| **Reverse Proxy Cache** | Infrastructure | An intermediate server (e.g., Varnish, Nginx) that caches full HTTP responses before they reach our application servers. |
| **Local / In-Process Cache** | Topology | A cache that lives inside our application's heap memory (e.g., Caffeine, Guava). Extremely fast (~100ns) but limited by heap size and inconsistent across instances. |
| **Distributed Cache** | Topology | A centralized, shared cache cluster (e.g., Redis, Memcached) accessible over the network by all application instances. Slower (~1ms) but offers massive capacity and global consistency. |
| **Cache-Aside (Lazy Loading)** | Pattern | **Read:** Check cache; on miss, query DB and populate cache. **Write:** Update DB, then delete/update cache. The most common and resilient default pattern. |
| **Read-Through** | Pattern | The cache library itself is responsible for loading missing data from the DB on a miss. The application treats the cache as the single source of truth for reads. |
| **Write-Through** | Pattern | Writes go to the cache first, which synchronously writes to the DB. Returns success only after the DB is committed. Guarantees strong consistency but increases write latency. |
| **Write-Behind (Write-Back)** | Pattern | Writes go to the cache and return immediately. The cache asynchronously batches and flushes writes to the DB in the background. Maximizes write throughput but risks data loss if the cache crashes. |
| **Write-Around** | Pattern | Writes bypass the cache entirely and go straight to the DB. The cache is only populated on read misses. Prevents cache pollution from cold, bulk writes. |
| **Eviction** | Memory Management | The process of removing existing keys from a full cache to make room for new keys. Driven by policies like LRU or LFU. |
| **LRU (Least Recently Used)** | Eviction Policy | Evicts the key that has gone the longest without being accessed. The general-purpose workhorse; exploits temporal locality. |
| **LFU (Least Frequently Used)** | Eviction Policy | Evicts the key with the lowest access count. Resistant to scan/pollution attacks. Often paired with "decay" to prevent old hot keys from living forever. |
| **TTL (Time-To-Live) / Expiry** | Eviction / Invalidation | A timer attached to each cache entry. When the timer expires, the key is automatically considered stale and eligible for eviction. Serves as both a memory-management and invalidation mechanism. |
| **ARC / TinyLFU** | Eviction Policy | Adaptive policies that dynamically balance between LRU and LFU based on real-time workload. TinyLFU (used in Caffeine) uses a frequency sketch to efficiently track popularity. |
| **Slab Allocator** | Memcached Internals | Memcached's memory management system that divides memory into fixed-size chunks (slabs) to avoid fragmentation. Eviction occurs per-slab-class, not globally. |
| **Invalidation** | Consistency | The explicit act of marking a cache entry as stale or deleting it because the underlying source of truth has changed. Famously one of the hardest problems in CS. |
| **Cache Busting / Versioning** | Invalidation | Changing the cache key itself (e.g., `style.v2.css`) rather than deleting the old one. Guarantees fresh reads without race conditions; ideal for immutable assets. |
| **Event-Based Invalidation** | Invalidation | Publishing a message (e.g., via Kafka) on every database write. All cache nodes listen to this event and evict the specific key. Enables near-instant consistency across services. |
| **Weak / Eventual Consistency** | Consistency Model | Accepting that the cache may serve stale data for an undefined period. The system guarantees only that it will eventually become consistent (usually via TTL). |
| **Strong Consistency** | Consistency Model | Guaranteeing that every read returns the most recent successful write. Achieved via synchronous Write-Through patterns or distributed locks. Expensive but necessary for financial/inventory data. |
| **Bounded Staleness** | Consistency Model | A compromise: guaranteeing the cache will never be more than X seconds stale (e.g., 60-second TTL). Gives us the performance of eventual consistency with a safety guarantee. |
| **Read-Your-Writes** | Consistency Model | Guaranteeing that a user always sees their own writes immediately after performing them. Achieved via "sticky sessions" or pinning reads to the primary node. |
| **Monotonic Reads** | Consistency Model | Guaranteeing that a user never sees "time travel"—i.e., going from a newer version of data back to an older one. |
| **Leaky Cache** | Failure State | The inevitable scenario where the cache and the database get out of sync due to race conditions, failed transactions, or network partitions. Managed via TTLs and versioned writes. |
| **Cache Stampede / Thundering Herd** | Failure Mode | When a key expires and thousands of concurrent requests simultaneously miss the cache, all hammering the database at once. |
| **Dogpile Effect** | Failure Mode | A synonym for Cache Stampede, specifically referring to the moment the cache key expires and everyone rushes to recompute it. |
| **Cache Penetration** | Failure Mode | Requests for non-existent keys that bypass the cache entirely and hit the database every single time. Mitigated via Null Caching or Bloom Filters. |
| **Hot Key** | Failure Mode | A single key (e.g., a celebrity profile) receiving a disproportionately massive number of requests, overwhelming the specific Redis node that owns it. Mitigated via local caching or key replication. |
| **Split-Brain** | Distributed Failure | A cluster split into two partitions, both believing they are the primary and accepting writes, leading to conflicting data when the partition heals. |
| **OOM Killer (Out-Of-Memory)** | Production Failure | The Linux kernel forcibly terminating the Redis process because it exceeded its allowed memory limit. Mitigated via proper sizing and memory alerts. |
| **Connection Pool** | Application Pattern | A managed pool of persistent TCP connections to the cache. Prevents the overhead of opening/closing connections per request. Exhaustion of this pool can hang the service. |
| **Circuit Breaker** | Resilience Pattern | A pattern that "trips" and stops calling the cache if failures exceed a threshold, allowing the system to fail-fast and fall back to the database to prevent cascading failures. |
| **Consistent Hashing** | Sharding | A distribution algorithm using a hash ring. Allows adding/removing nodes with minimal key remapping (~1/N keys), preventing massive cache invalidations during scaling. |
| **Hash Slots** | Redis Sharding | Redis Cluster's specific sharding mechanism. Divides the key space into 16,384 slots. Nodes own a range of slots. Clients use `CRC16(key) % 16384` to route. |
| **Replication** | High Availability | Copying data from a primary node to secondary (replica) nodes. Used for scaling read traffic and providing failover capability. |
| **RDB (Snapshot)** | Redis Persistence | Redis saving a point-in-time binary snapshot of the entire dataset to disk (e.g., `dump.rdb`). Fast restore but loses data since the last snapshot. |
| **AOF (Append-Only File)** | Redis Persistence | Redis logging every write operation to a file. More durable than RDB (configurable fsync), but larger file size and slower restart. Production systems often use both. |
| **Serialization** | Data Encoding | The process of converting an in-memory object (e.g., a Python dict) into a byte stream for storage in Redis. |
| **Protobuf (Protocol Buffers)** | Serialization | A binary serialization format requiring a strict schema. Highly compact and fast. Preferred for internal, high-throughput service-to-service caching. |
| **MessagePack** | Serialization | A binary format similar to JSON but smaller and faster. Schema-less, making it flexible for dynamic data structures. |
| **Bloom Filter** | Penetration Mitigation | A probabilistic data structure that can definitively say "this key does NOT exist." Used to filter out requests for non-existent keys before they hit the DB. |
| **Null Caching** | Penetration Mitigation | Explicitly caching a NULL or NOT_FOUND marker for keys that don't exist (with a short TTL) to prevent repeated DB lookups. |
| **Dual-Writes** | Migration Strategy | Writing to both the old and new cache clusters simultaneously during a migration. Ensures the new cluster is populated before we cut over reads. |
| **Backfill** | Migration Strategy | A background batch process that reads all keys from the old cache/database and writes them to the new cache to "warm it up" before a traffic cutover. |
| **Blue-Green Deployment** | Migration Strategy | Fully provisioning a new cache cluster (Green) alongside the old one (Blue) and atomically switching traffic via DNS or configuration. Allows instant rollbacks. |

---

⬅️ **[Previous: 12. Unseen Production Realities](12_Unseen_Production_Realities.md)** | 🏠 **[Back to TOC](README.md)** | **[Next: 14. Prerequisites ➡️](14_Prerequisites.md)**
