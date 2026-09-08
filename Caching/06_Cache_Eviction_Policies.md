# 6. Cache Eviction Policies

We have designed our interaction patterns (Cache-Aside, Write-Through, etc.) and decided where our cache lives (Local vs. Distributed). But there is an inescapable physical constraint: **memory is finite.**

We only have 10GB, 50GB, or 100GB of RAM allocated to Redis or our local heap. Eventually, we will reach that limit. When a new key arrives and the cache is full, we must decide which existing key to sacrifice to make room for the new one. 

This decision is the eviction policy. The policy we choose directly determines our Cache Hit Ratio in production. A poor eviction policy means we are constantly evicting hot keys and thrashing the database.

Let's walk through the major policies, how they actually work under the hood, their real-world trade-offs, and what we actually see running in production systems.

---

### 6.1 LRU (Least Recently Used)

**The Mental Model:**
We treat the cache like a stack of books on a desk. Every time we use a book, we place it on the very top. When the desk is full and we need a new book, we look at the very bottom of the stack—the one that hasn't been touched in the longest time—and throw it away.

**How it works under the hood:**
To achieve `O(1)` lookups and `O(1)` evictions, we internally maintain a Doubly Linked List and a Hash Map.
- The HashMap points to the node in the list.
- On a `get()` or `set()` of an existing key, we move that node to the head (most recent).
- On an eviction, we simply remove the node at the tail (least recent).

**Why we use it:**
It perfectly exploits **Temporal Locality**. If a key was accessed recently, it is statistically highly likely to be accessed again soon. It is the most robust general-purpose policy and the absolute workhorse of production systems.

**Real-World Tech Example:**
- **Memcached** (default and primary eviction mechanism).
- **Redis** when configured with `maxmemory-policy allkeys-lru` or `volatile-lru`.
- **CPU L1/L2 Caches** (though they use an approximated, hardware-optimized variant).
- **OS Page Cache** uses a variant of LRU (specifically, the "Clock" / Second-Chance algorithm to avoid the overhead of moving nodes in kernel space).

**The Trade-offs & The Hidden Problem:**
- **Pros:** Exceptional hit ratio for standard web workloads (user sessions, product catalogs, news feeds). Implementation is relatively straightforward.
- **Cons (The "Scan" problem):** Imagine we run a nightly ETL batch job that scans 1 million keys from our database. As this batch runs, it reads every single key once, pushing them to the head of the LRU list. By the end of the batch, our LRU cache is now filled with these 1 million cold, one-time-use keys, and our actual hot 20% working set has been pushed out and evicted. When 9:00 AM hits and our users log in, the cache is completely polluted, and we experience 100% cache misses, killing the database.

> **🛠️ What we do in the real world to fix this:**
> To fight the scan problem, we rarely rely on perfect LRU in high-scale distributed systems. Redis does not use a perfect LRU. Perfect LRU requires moving nodes in a linked list on every read, which takes a spinlock and has high overhead at 1 million+ QPS. Instead, Redis uses an **Approximated LRU**. When eviction is needed, Redis randomly samples a small set of keys (e.g., 5 keys), compares their last access timestamps, and evicts the oldest one. This is nearly as good as perfect LRU for random distributions but avoids the heavyweight linked-list pointer manipulation.

---

### 6.2 LFU (Least Frequently Used)

**The Mental Model:**
We count how many times each book has been read. We track the "popularity" score. When the desk is full, we evict the book with the absolute lowest read count.

**Why we use it:**
LFU solves the Scan/Pollution problem decisively. The nightly ETL batch that reads 1 million keys once will increment their counters to "1". Our actual hot keys have counters in the thousands or millions. The cold keys will never evict the hot ones.

**Real-World Tech Example:**
- **Redis 4.0+** introduced `allkeys-lfu` and `volatile-lfu` specifically to handle this scenario.
- **Content Delivery Networks (CDNs)** for globally cached video files often use LFU. A viral video on YouTube is accessed millions of times; a trending news video is accessed hundreds of thousands; an old archived video is accessed once a day. Frequency is the best predictor of global popularity.

**The Trade-offs & The Hidden Problem:**
- **Pros:** Highly resistant to cache pollution. Excellent for workloads with a permanently "hot" set of items (like a global leaderboard or a celebrity profile).
- **Cons (The "Decay" problem):** LFU suffers from lack of temporal decay. Imagine a weekly flash-sale event. Product "A" is hammered 10 million times on Monday. Product "B" becomes the new flash-sale item on Friday. Product "A" has a massive counter. Under strict LFU, "A" will stay in the cache forever, and "B" (which now has a counter of only "1") will keep getting evicted, even though temporal locality says B is now the hot key.

> **🛠️ What we do in the real world:**
> To fix this, we never use a pure counter. We use **Decaying LFU**. Redis handles this beautifully: It stores a counter that is reduced over time. As time passes, old counters logarithmically decrease, so stale "old hot" keys slowly drift down, allowing new "current hot" keys to compete fairly.

---

### 6.3 TTL (Time-To-Live) / Expiration

**The Mental Model:**
Every item we put in the cache comes with an expiration timer, like food in a refrigerator. When the timer hits zero, the item is considered stale and becomes eligible for removal.

**Why we use it:**
TTL is fundamentally about data freshness and implicit invalidation, not just memory pressure. We set a TTL to guarantee that we don't serve data older than, say, 5 minutes. It is our primary defense against the "Cache Invalidation is hard" problem—if we can't reliably push an invalidation event, we just let the time run out and let the cache naturally cool down.

**Real-World Tech Example:**
- **HTTP Caching:** The `Cache-Control: max-age=3600` header tells CDNs and browsers to evict the asset after 1 hour.
- **DNS Records:** TTLs are set to control how long resolvers cache IP addresses.
- **Session OTPs:** A 2-Factor Authentication code has a 5-minute TTL; we evict it automatically to enforce security.
- **Redis:** We use `EXPIRE key 300` on every write to ensure we don't serve stale user profiles after a configuration change.

**The Trade-offs & Critical Nuance:**
- **Pros:** Trivial to implement. Guarantees eventual consistency. Perfect for ephemeral data.
- **Cons:** TTL is **not** a sufficient eviction policy for memory management. If we set a 1-hour TTL but write 10GB of new data every 30 minutes, our 5GB Redis instance will run out of memory and crash before the 1-hour TTL expires. We must combine TTL with LRU/LFU (called `volatile-lru` in Redis) so that Redis evicts the least recently used expirable keys when memory is tight, even if their TTL hasn't expired yet.

---

### 6.4 FIFO / LIFO (First In, First Out / Last In, First Out)

**The Mental Model:**
FIFO is a queue. The first key we put into the cache is the first one we evict. LIFO is a stack. The most recently added key is evicted first.

**Why we (almost never) use these:**
They completely ignore access patterns. FIFO evicts the oldest data, even if that oldest data is the CEO's profile being accessed every 5 seconds. LIFO evicts the newest data, even if that newest data is the breaking news article that millions are about to read.

**Where they show up in real life:**
FIFO is used deep inside database transaction logs (WAL) or streaming buffers where order is preserved, but never for application performance caching. We generally avoid these for Redis/Memcached because they destroy the hit ratio.

---

### 6.5 Advanced / Adaptive (ARC and TinyLFU)

**The Mental Model:**
Instead of picking LRU or LFU, we use a policy that dynamically balances between them based on the current workload. If we notice the workload is scan-heavy, we lean toward LFU. If we notice the workload is time-shifting rapidly (flash sales), we lean toward LRU.

**Real-World Tech Example:**
- **ARC (Adaptive Replacement Cache):** Widely used in file systems (ZFS, PostgreSQL) and some storage appliances. It maintains 4 lists (recent, frequent, ghost lists) to track what we evicted and learn whether LRU or LFU would have performed better.
- **Caffeine (Java):** This is the de-facto local caching library used in Spring Boot and modern microservices. It uses **Window-TinyLFU**. It divides the cache into a small "admission window" (handling LRU behavior) and a large "main segment" (handling LFU based on a frequency sketch - a compact Count-Min Sketch to save memory). This gives us the best of both worlds: it adapts instantly to new hot keys (via the window) and protects the cache from long-term pollution (via the LFU main segment).

**The Trade-off:**
- **Pros:** The highest possible hit ratio without manual tuning. Handles unpredictable traffic patterns beautifully.
- **Cons:** Implementation is extremely complex. Managing ghost lists and frequency sketches consumes extra memory (though Caffeine optimizes this heavily).

> **🛠️ What we do in the real world:**
> - For **Distributed caches (Redis/Memcached)**, we stick with LRU or LFU because the memory per key is precious, and we don't want the overhead of maintaining complex ghost lists across a cluster.
> - For **Local/In-Process caches (within the JVM)**, we use Caffeine as our default, because we have more heap flexibility, and the adaptive nature significantly boosts our local hit ratio without any configuration changes.

---

### 6.6 Implementation Details: Redis vs. Memcached vs. Caffeine

We need a quick glance at how these actually work under the hood to make better choices.

- **Redis (Distributed):**
  Redis stores all keys in a global hash table. To avoid `O(N)` overhead on evictions, Redis uses Random Sampling. For LRU/LFU, it doesn't scan the entire 10GB dataset. It grabs a random sample of ~5 keys (configurable), compares their last access time (or counter), and evicts the worst one from the sample. This is `O(N)` where `N = sample size`, making it blazing fast even at 100GB of data. We set the policy in `redis.conf` via `maxmemory-policy`.

- **Memcached (Distributed):**
  Memcached uses a **Slab Allocator** to avoid memory fragmentation. It divides memory into slabs of specific sizes (e.g., 1KB, 2KB, 4KB). Eviction is per-slab-class. If the 2KB slab is full, it evicts the LRU item from only that slab. This means an overpopulated 2KB slab won't evict items in the 1KB slab. This is crucial to remember—sometimes a key is evicted not because it's globally cold, but because its specific size class is under pressure.

- **Caffeine (Local/In-Process):**
  Uses a ring-buffer to record access events, which are processed asynchronously by a dedicated maintenance thread. The frequency filter (Count-Min Sketch) is probabilistic and uses tiny memory footprints. We use it when we want to scale local caches to 5,000+ items without GC pressure.

---

## 🏆 The Real-World Engineering Choice (Our Rule of Thumb)

- **If we are running Redis:** We start with `volatile-lru` (evict the least recently used among keys that have a TTL). If we don't use TTLs, we use `allkeys-lru`. If we see scan-heavy nightly jobs, we upgrade to `allkeys-lfu`.
- **If we are running Memcached:** We accept the LRU structure and ensure our slab sizes are tuned so that keys of similar sizes don't cannibalize each other.
- **If we are running a Local JVM cache:** We always use Caffeine with `.maximumSize()` and let it use Window-TinyLFU. It outperforms manual LRU implementations by a significant margin and requires zero tuning from our side.
- **If we are running a CDN / Static Assets:** We lean heavily on LFU (or vendor-specific adaptive policies) because global popularity is more static than dynamic.

---

⬅️ **[Previous: 5. Core Caching Strategies](05_Cache_Updating_Strategies.md)** | 🏠 **[Back to TOC](README.md)** | **[Next: 7. Cache Invalidation ➡️](07_Cache_Invalidation.md)**
