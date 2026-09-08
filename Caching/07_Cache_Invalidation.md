# 7. Cache Invalidation

### 7.1 Why Invalidation is Famously Difficult

There is a famous quote: *"There are only two hard problems in Computer Science: cache invalidation, naming things, and off-by-one errors."* This is not a joke. It is a profound recognition of a distributed systems reality.

**Why is it so hard?**

- **The Time-Space Trade-off:** Caching is betting on temporal locality (we think we will read this again soon). Invalidation requires us to predict the exact instant that the source of truth changes, and then propagate that change to *every single cache layer*—including CDNs, reverse proxies, local in-process heaps, and distributed Redis clusters—within milliseconds. We are essentially fighting against the physics of network propagation.
- **Distributed State (The CAP Theorem):** Our cache and our database are two distinct, independent state machines. When we update the database, we cannot atomically update the cache in the same transaction (unless we use complex 2PC, which we generally avoid). This creates a consistency window—a moment in time where the database has the new value, but the cache still has the old one.
- **The "Ghost" Problem:** We don't just cache keys that exist. We often cache the absence of a key (e.g., `user:999` does not exist, so we cache a `NULL` to prevent cache penetration). Invalidation requires us to know about all possible variants of a key. If we rename a product ID, do we invalidate the old one? The complexity scales combinatorially.

> **🧠 The Mental Model:**
> We should think of invalidation as a race condition between reads and writes. If a read happens immediately after a write, does it see the stale cache or the new database value? The answer depends entirely on our invalidation strategy.

---

### 7.2 Invalidation Strategies

We have four primary weapons in our invalidation arsenal. In production, we rarely use just one; we layer them based on the criticality of the data.

#### 7.2.1 Time-Based (TTL / Expiry)

**What we do:**
We associate every cache entry with an expiration time (`EXPIRE key 300` in Redis, or `Cache-Control: max-age=600` in HTTP). When the timer runs out, the cache entry is automatically evicted or considered stale. This is the only invalidation mechanism that requires zero explicit action from our application code on a write.

**Why we use it:**
It is the simplest and most resilient invalidation strategy. It acts as a safety net. Even if our event-driven invalidation fails (due to a network partition or a bug), the cache will eventually self-heal and fetch fresh data.

**Real-World Tech Example:**
- **Public REST APIs:** We set a 60-second TTL on a product catalog endpoint. Even if an admin updates a product price, the cache serves stale data for at most 60 seconds. This is perfectly acceptable for e-commerce browsing, and it prevents the database from being hammered.
- **DNS Records:** A TTL of 300 seconds means our users will hit an outdated IP for at most 5 minutes during a failover.
- **Session OTPs:** We use a 5-minute TTL not just for eviction, but for security. We explicitly want the data to disappear.

**The Trade-offs:**
- **Pros:** Trivial to implement. Handles all edge cases automatically. Requires no coordination between services.
- **Cons:** We cannot choose a TTL that perfectly balances freshness and performance. A 1-second TTL means we hit the database constantly; a 1-hour TTL means we serve stale data for an hour. We are essentially giving up on strong consistency and embracing eventual consistency.

> **🚨 The Critical Nuance We Must Remember:**
> TTL is **not** a replacement for explicit invalidation on critical writes. If a user updates their password, we cannot wait 5 minutes for the TTL to expire—they would be locked out of their account. For security-critical data, TTL is a fallback, not the primary invalidation mechanism.

#### 7.2.2 Event-Based / Write-Triggered (Push Invalidation)

**What we do:**
Whenever a write occurs to the source of truth (the database), we publish an "invalidation event" to a message bus (e.g., Kafka, RabbitMQ, or Redis Pub/Sub). Any service that holds a local or distributed cache listens to this event and, upon receiving it, immediately deletes or updates the specific key.

**Why we use it:**
This is the gold standard for strong consistency. It allows us to invalidate the cache synchronously (or near-synchronously) with the write, minimizing the stale-data window to milliseconds.

**Real-World Tech Example:**
- **E-commerce Product Page:** When a seller updates the price, a `ProductUpdatedEvent` is published. The search service, the recommendation service, and the product detail service all listen to this event and evict the specific `product:123` key from their local JVM caches and the global Redis cluster.
- **Social Media Feeds:** When a user posts a new status, an event invalidates the `feed:user:456` cache across all edge nodes.

**The Trade-offs:**
- **Pros:** Near-instant consistency. Excellent for fast-changing data where staleness is unacceptable.
- **Cons:** Complex to implement. We must handle out-of-order events. If an event arrives late and a newer event was processed, we cannot blindly delete the cache, or we risk re-populating it with an old value. We also introduce a dependency on the message broker.

> **🏆 The Production Pattern (Dual-Layer Invalidation):**
> **Primary:** Synchronous `cache.delete(key)` on the write path for the local service.
> **Secondary:** Asynchronous event published to Kafka to invalidate all other services that might have cached this key.
> **Fallback:** A short TTL (e.g., 60 seconds) to guarantee eventual cleanup if either of the above fails.

#### 7.2.3 Manual Invalidation (Explicit Cache.delete)

**What we do:**
Our application code explicitly calls `cache.delete(key)` at the exact moment we update the database. This is the simplest form of write-triggered invalidation, but it happens within the same service process, not across a message bus.

**Why we use it:**
When we operate a single-service architecture or a tightly coupled monolith, we don't need Kafka. We just embed the invalidation logic directly inside the transaction.

```python
# The classic Cache-Aside write path
def update_user(user_id, new_data):
    db.execute("UPDATE users SET ... WHERE id = ?", user_id)
    redis.delete(f"user:{user_id}")  # Manual invalidation
```

**The Trade-offs:**
- **Pros:** Extremely low latency. No external dependencies. Predictable behavior.
- **Cons:** Tight coupling. If we update the database via a stored procedure, a background worker, or a direct SQL script, we might forget to invalidate the cache. 

> **💡 Critical Nuance (Delete vs. Update):**
> We generally prefer `delete` over `set` on writes. Why? Because if we set the cache to the new value, and the database transaction fails (rollback), we are now serving a value that never actually committed. Deleting the key ensures that the next read will fetch the current committed state from the DB.

#### 7.2.4 Versioning / Cache Busting

**What we do:**
We change the cache key itself to include a version or a timestamp. Instead of storing `user:123`, we store `user:123:v2`. When the user profile is updated, we bump the version to `v3`. The old key (`:v2`) naturally rots away, and all reads fetch `:v3`, which is guaranteed fresh.

**Why we use it:**
This eliminates the race condition entirely. Invalidation via delete has a "lost update" problem: Read A checks the cache, finds a miss, starts querying the database. While Read A is fetching, Write B comes in, deletes the old key and writes new data. Read A finishes and writes the old data back into the cache, resurrecting the stale value. Versioning completely avoids this because Write B changes the key name; Read A can only write to the old version, which is never read again.

**Real-World Tech Example:**
- **Static Assets (CDN):** We append a hash of the file content to the URL (e.g., `style.a1b2c3.css`). When we deploy new CSS, we generate a new hash. The CDN never needs to invalidate the old asset; browsers fetch the new URL automatically.
- **Immutable Data Stores:** In event-sourced systems, we use versioned snapshots. The cache key is `snapshot:aggregate:456:103`. We only request the latest version.

**The Trade-offs:**
- **Pros:** No invalidation coordination is required. No race conditions. Extremely safe.
- **Cons:** We suffer from cache churn. The old keys sit in memory, wasting space, until the eviction policy eventually removes them. 

---

### 7.3 The Real-World Engineering Choice (Our Rule of Thumb)

We layer these strategies based on the data classification:

| Data Classification | Primary Invalidation | Fallback | Why |
| :--- | :--- | :--- | :--- |
| **Security / User Identities** (Passwords, Roles) | Manual Delete + Event-Based | TTL (30s) | Stale data can cause privilege escalation. Must invalidate instantly, but keep a short TTL as a failsafe. |
| **Product Catalog / Reference Data** | Event-Based / TTL | TTL (5m) | Stale prices are bad, but the business can tolerate a 5-minute window. We rarely use manual delete here. |
| **Static Assets** (CSS, JS, Images) | Versioning (Cache Busting) | TTL (1y) | We serve old assets indefinitely. Invalidation is never needed; we simply issue new URLs. |
| **Analytics / Counters** | TTL Only | None | We accept eventual consistency. The business doesn't care if the "view count" is 5 seconds stale. |

> **⚠️ The Critical Misconception (The Trap):**
> *"We will just invalidate the cache on every write."*
> 
> This is the most common failure in distributed design. If we invalidate `user:123` on every profile update, and an admin bulk-updates 10,000 users, we suddenly send 10,000 invalidations to Redis and Kafka. This creates a write amplification problem. The cache can become slower than the database because of the sheer volume of delete operations. 
> 
> We avoid this by implementing coalescing (batching invalidation events) or by simply opting for TTL during bulk operations. Sometimes, the best invalidation is waiting for the TTL to expire.

---

⬅️ **[Previous: 6. Cache Eviction Policies](06_Cache_Eviction_Policies.md)** | 🏠 **[Back to TOC](README.md)**
