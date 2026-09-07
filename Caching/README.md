# Caching

Caching is one of the most effective ways to improve the performance and scalability of a system. By temporarily storing frequently accessed or computation-heavy data in memory, caching reduces the load on databases and decreases response times.

This directory will contain detailed notes, diagrams, and code snippets focusing on various caching mechanisms.

---

## 📌 Proposed Learning Path

To master caching, I recommend proceeding in the following order. You can create a separate markdown file for each of these topics as you learn them:

### [1. Introduction to Caching](01_Introduction.md)
- **What is a Cache?** Principle of Locality (Temporal, Spatial).
- **Why Cache?** Latency reduction, throughput increase, cost savings.
- **Cache Hit vs. Cache Miss.** (Bonus: Cache hit ratio).

### [2. Cache Topologies & Layers](02_Cache_Layers.md)
- **Client Caching:** Browser Cache, OS Cache.
- **CDN (Content Delivery Network):** Caching static assets at the edge.
- **Web Server Caching:** Reverse Proxies (Varnish, Nginx).
- **Application Level Caching:** In-memory caching within the app layer.
- **Database Caching:** Query cache, Buffer pool.
- **Global / Distributed Caching:** Redis, Memcached.

### [3. Cache Updating Strategies](03_Updating_Strategies.md)
How and when do we write data to the cache vs the database?
- **Cache-Aside (Lazy Loading):** App talks to both cache and DB.
- **Read-Through:** App talks to cache; cache talks to DB to get missing data.
- **Write-Through:** App writes to cache; cache synchronously writes to DB.
- **Write-Behind (Write-Back):** App writes to cache; cache asynchronously writes to DB.
- **Write-Around:** App writes directly to DB, bypassing cache.

### [4. Cache Eviction Policies](04_Eviction_Policies.md)
When the cache is full, how do we decide what to remove?
- **LRU (Least Recently Used)** - *Most common!*
- **LFU (Least Frequently Used)**
- **FIFO (First In First Out)**
- **Random Replacement**

### [5. Cache Invalidation & TTL](05_Invalidation.md)
- **TTL (Time to Live):** Setting expiration times for keys.
- **Explicit Invalidation:** When the application actively deletes dirty data from the cache.

### [6. Common Caching Problems](06_Caching_Problems.md)
- **Thundering Herd / Cache Stampede:** What happens when an expensive key expires and 10,000 requests hit the DB at once?
- **Cache Penetration:** Querying keys that don't exist in the DB or Cache over and over. (Solution: Bloom Filters).
- **Cache Breakdown:** Hot keys expiring.
- **Cache Avalanche:** Many keys expiring at the exact same time.

### [7. Real-world Technologies](07_Technologies.md)
- **Redis:** Data structures, persistence (RDB/AOF), Pub/Sub.
- **Memcached:** Simple key-value store, multi-threading.
- Comparing Redis vs Memcached.

---

## 🚀 Next Steps

Start by researching **1. Introduction to Caching** and **2. Cache Topologies**.
Once you learn a concept, document it, perhaps draw a quick diagram, and move to the next. Let me know when you want to dive into the first topic!
