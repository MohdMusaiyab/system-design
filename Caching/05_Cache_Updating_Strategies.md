# 5. Core Caching Strategies / Patterns (How we interact with the cache)

When designing a caching layer, we must decide how the application, the cache, and the database interact on reads and writes. Here are the five fundamental strategies.

---

### 5.1 Cache-Aside (Lazy Loading)

**What it is:**
The application takes full responsibility. On a read:
1. Check the cache.
2. If hit, return the value.
3. If miss, query the database, store the result in the cache, and return it.

On a write: Update the database, then explicitly delete (or update) the key in the cache.

**Why we use it:**
It's the most intuitive and widely used pattern because it only loads data into the cache when it's actually requested. We don't waste memory pre-populating data that nobody reads. It perfectly exploits the 80/20 rule—only the "20%" of hot data ever makes it into the cache.

**The Trade-offs:**
- **Pros:**
  - Extremely flexible. We control exactly what goes in and out.
  - Memory efficient (only the working set is cached).
  - Resilient to cache failures (if Redis dies, the app just reads from the DB and repopulates it later).
- **Cons:**
  - Higher latency on misses (3 steps: cache check → DB query → cache insert).
  - Stale data window: Between the time we update the DB and the time we delete/update the cache, the cache returns old data. This is the "eventual consistency" penalty.
  - **Cache Stampede (Thundering Herd):** If a key expires and 1,000 concurrent requests arrive simultaneously, all 1,000 see a miss and hammer the database at the exact same instant.

**When to choose it (and why):**
This is the default choice for 90% of read-heavy backend services (e.g., product catalogs, user profiles, session data). We choose it because it is simple to reason about and handles cache failures gracefully. We mitigate the stampede problem separately with mutex locks or "probabilistic early expiration".

---

### 5.2 Read-Through

**What it is:**
The cache sits between the application and the database. The application treats the cache as the primary data source. When the app asks for a key, the cache library itself is responsible for loading the missing data from the database and populating itself.

**Why we use it:**
To offload the "loading logic" from the application code. Instead of writing `if cache.get() == null { db.query(); cache.set(); }` everywhere, we just call `cache.get(key)` and the underlying library (like Caffeine or a Redis module) handles the fallback invisibly.

**The Trade-offs:**
- **Pros:** Cleaner application code. The cache is the single abstraction for reads.
- **Cons:** The cache becomes stateful and more complex. The cache layer now needs database credentials and query logic. It also introduces tight coupling between the cache infrastructure and our database schema.
- **Difference from Cache-Aside:** Functionally identical in terms of data flow (lazy load on miss), but the responsibility shifts from the app to the cache library.

**When to choose it:**
When we're using local caches like Guava's `LoadingCache` or advanced distributed caches that support server-side scripting. We choose this over Cache-Aside when we want to strictly enforce a "single source of truth" abstraction and keep our business logic free of caching boilerplate.

---

### 5.3 Write-Through

**What it is:**
On a write (UPDATE/DELETE), the application writes to the cache first. The cache then synchronously takes responsibility for writing the data to the database. The app only gets a success response after the database has been successfully updated.

**Why we use it:**
To guarantee strong consistency between the cache and the database. If a write succeeds, we know the cache is 100% up-to-date. Subsequent reads will never fetch stale data.

**The Trade-offs:**
- **Pros:** Data consistency is phenomenal. Invalidation logic is baked into the write path.
- **Cons:** Higher write latency. The application now has to wait for the DB commit plus the network round-trip to the cache. It also doubles the writes—every DB write is also a cache write, which wastes memory if the data is rarely read.

**When to choose it:**
We generally use this only for critical, frequently read data where consistency is non-negotiable (e.g., user email changes, financial account balances). However, in practice, we often prefer Write-Through + Cache-Aside together: We write through the cache to keep it hot, but we still use Cache-Aside logic on the read path as a fallback.

---

### 5.4 Write-Behind (Write-Back)

**What it is:**
The application writes to the cache and immediately returns a "success" to the user. The cache then asynchronously batches these writes and flushes them to the database in the background (usually every few seconds or when a threshold is met).

**Why we use it:**
To absorb massive write throughput and decouple the application from database latency. If our DB can handle 5,000 writes/sec but our application receives 50,000 writes/sec, we funnel them through Redis with Write-Behind. The cache batches them (e.g., accumulating counters) and flushes a single aggregated update to the DB.

**The Trade-offs:**
- **Pros:** Lowest write latency for the user. Drastically reduces the load on the primary database (batching is far more efficient than single row updates). Great for high-frequency metrics, logs, or social media "like" counters.
- **Cons:** Data durability is at risk. If the cache node crashes before flushing the write to the database, that data is permanently lost. The system achieves Eventual Consistency—if the cache flushes every 5 seconds, the DB is up to 5 seconds stale.

**When to choose it:**
We choose this strictly when we can tolerate data loss or when the data is inherently ephemeral (e.g., clickstream analytics, view counts, session activity pings). We never use this for financial transactions, authentication data, or user profile updates where losing the write is unacceptable.

---

### 5.5 Write-Around

**What it is:**
On a write, the data goes directly to the database. It completely bypasses the cache. The cache is only populated on read misses.

**Why we use it:**
To prevent cache pollution. If we're writing a massive volume of data that we know will never be read again (e.g., bulk ingestion of historical logs, heavy ETL job outputs), we don't want to waste precious Redis memory storing cold data that pushes out the hot keys.

**The Trade-offs:**
- **Pros:** Maximizes cache efficiency. Only "hot" keys that are actually requested occupy memory.
- **Cons:** A "read-after-write" scenario fails. If we write a new record, and then immediately try to read it via the cache, it will miss and hit the DB. This creates an inconsistent experience for the user right after they submit data.

**When to choose it:**
We use this specifically for batch jobs and heavy data imports. For user-facing APIs, we rarely use this alone. We usually pair it with Cache-Aside: writes go to the DB (Write-Around), but the read path uses Cache-Aside, so the first read after a write populates it.

---

## 6. Comparison & Selection Criteria (The Decision Matrix)

Now that we understand the mechanics, how do we decide in a real system design session? We stack them against four axes: Read/Write Ratio, Consistency Requirement, Performance Requirement, and Operational Complexity.

| Pattern | Read Path | Write Path | Consistency | Performance Impact | Complexity | When We Choose It |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Cache-Aside** | Check cache, fetch DB on miss | Update DB, invalidate cache | Eventual (stale until invalidated) | Low latency on hits, high on misses | Low | **Default choice.** Read-heavy APIs with moderate write frequency (User profiles, Product pages). |
| **Read-Through** | Cache loads DB on miss | Update DB, invalidate cache | Eventual | Same as Cache-Aside | Medium | When we want to hide caching logic behind a library/abstraction. |
| **Write-Through** | Check cache (usually hit) | Write to cache, synchronously to DB | Strong (DB and cache are in sync) | High write latency (doubles the I/O) | Medium | When consistency is more important than write speed (Auth tokens, Account details). |
| **Write-Behind** | Check cache | Write to cache, return. Async flush to DB | Weak / Eventual (risk of data loss) | Extremely low write latency, DB load is smoothed | Very High | High-frequency writes where data loss is acceptable (Analytics, logging, counters). |
| **Write-Around** | Check cache, fetch DB on miss | Write directly to DB, bypass cache | Eventual (cache lags immediately after write) | Read misses are slow, writes are fast | Low | Bulk data loads. Or when we know writes won't be re-read soon. |

### The "Real System" Mix (The Production Reality)

We very rarely pick just one pattern for an entire system. Usually, we mix them:
- For hot data reads, we use **Cache-Aside**. It's simple, resilient, and handles failures gracefully.
- For critical writes (like changing an email), we couple it with a **Write-Through** behavior specifically for that key to keep it fresh.
- For analytics pipelines, we use **Write-Behind** to absorb 100k events/sec, accepting that losing the last 2 seconds of events is acceptable.
- We usually leave **Write-Around** for our data-warehouse ETL jobs, not for real-time user traffic.

> **💡 The Fundamental Engineering Takeaway:**
> Always prioritize the read path for latency and the write path for durability.
> 
> If our service is read-heavy (which most are), we optimize the hit ratio (Cache-Aside/Read-Through) and accept slower writes. If our service is write-heavy, we never let the cache degrade the writer's experience—we either write around it or write behind it.
> 
> The worst thing we can do is blindly pick "Write-Through" for a social media newsfeed (which is 95% reads, 5% writes) and triple our write latency for no benefit, or pick "Write-Behind" for a banking ledger and lose customer money on a node crash.

---

⬅️ **[Previous: 4. Fundamental Topologies](04_Fundamental_Topologies.md)** | 🏠 **[Back to TOC](README.md)** | **[Next: 6. Cache Eviction Policies ➡️](06_Cache_Eviction_Policies.md)**
