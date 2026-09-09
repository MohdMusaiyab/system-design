# 12. The "Unseen" Production Realities

We end with the edge cases that separate senior engineers from junior ones. These are the silent killers that hit us at 3 AM.

---

### 12.1 Cache Penetration

**The Problem:**
A malicious user or a bug queries `GET /user/999999999` (a user that doesn't exist). We use Cache-Aside, so we check the cache (miss), then check the DB (miss), and never populate the cache. Every single request for this non-existent user hits the database, bypassing the cache entirely.

**Mitigation 1 (Null Caching):**
We explicitly cache the `NULL` result with a short TTL (e.g., 60 seconds). `cache.set("user:999999", "NULL", 60)`.

**Mitigation 2 (Bloom Filter):**
We keep a Bloom Filter in memory that contains all existing user IDs. Before we query the cache, we check the Bloom Filter. If it says "Definitely doesn't exist," we return a 404 immediately, never hitting the DB at all.

---

### 12.2 Cache Breakdown / Hot Keys

**The Problem:**
A single key, like `superstar_celebrity_profile`, receives 1 million QPS. Even though it's a cache hit, the single Redis node responsible for that hash slot is overwhelmed by the sheer volume of network packets. The CPU of that node hits 100%, and the node becomes unresponsive.

**Mitigation:**
We replicate the hot key client-side. We explicitly store the same value on multiple Redis nodes (e.g., `hotkey:1`, `hotkey:2`, `hotkey:3`). Our application randomly chooses one of the three copies to read. This distributes the load across nodes.

**Alternatively (and preferably):**
We use **Local Caching** for this specific key. We cache this hot object in the application's local heap for 5 seconds using Caffeine or Guava. This stops the network calls to Redis entirely and absorbs the spikes locally before they hit the wire.

---

### 12.3 Monitoring & Observability

We cannot manage what we cannot measure. Our production dashboards must reliably show:

- **Hit Ratio (Daily, Hourly):** A sudden drop indicates eviction pressure or a failure in invalidation.
- **Miss Ratio:** Specifically tracking *why* it missed (Did the TTL expire? Was it explicitly evicted? Did it never exist?).
- **Latency Percentiles (p99):** Redis p99 latency should be under 1ms. If it spikes to 10ms, we are experiencing network congestion or CPU contention.
- **Memory Usage:** We set alerts when memory exceeds 80% of `maxmemory`. Eviction rates spike dramatically near capacity.
- **Eviction Rate / Sec:** If we evict thousands of keys per second, our hit ratio will plummet.
- **Slow Log:** Monitoring the Redis slow log (`slowlog get 100`) to catch specific commands taking >1ms.

---

### 12.4 Cache Sizing

How do we estimate the correct memory size for a new cluster?

**Formula:**
`Total Memory = (Number of Expected Active Keys * Average Value Size) + Overhead`

**Redis Overhead:**
Each key in Redis has a `redisObject` overhead of roughly 100-150 bytes (dict entry, expiration timestamp, etc.). If we have 10 million keys, the overhead alone is ~1.5GB! 

**Our Approach:**
We run a Benchmark in staging. We pull a sample of production traffic (e.g., 1 hour of requests) and replay it against a small Redis instance. We observe the memory used and extrapolate linearly.

> **💡 The Golden Rule of Thumb:**
> We never fill Redis above **75%** of its maxmemory. We leave 25% headroom for memory fragmentation (especially with high churn). If we estimate we need 8GB, we provision a 12GB instance. This prevents the OOM Killer from silently ending our cache.

---

⬅️ **[Previous: 11. Migration Strategies](11_Migration_Strategies.md)** | 🏠 **[Back to TOC](README.md)** | **[Next: 13. Glossary ➡️](13_Glossary.md)**
