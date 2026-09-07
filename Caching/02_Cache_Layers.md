# Cache Topologies & Layers

Let's walk through the caching layers topologically, from the hardware right up to the user.

---

## 3. The Memory Hierarchy & Caching Layers

The entire computing stack is basically a series of caching layers, each one trading speed for size and cost. The further we move from the CPU, the slower and cheaper the storage gets. The entire point of a modern OS is to manage this hierarchy so that the CPU rarely has to wait for the slowest layer.

### 3.1 Hardware Layers (CPU Registers → L1/L2/L3 → RAM → Disk)
This is the foundation. Every other software cache we build is just mimicking this physical design.

- **CPU Registers:** The absolute fastest. They store the exact operands the CPU is working on right now. 
  - *Access time:* ~0.3 ns
  - *Size:* A few hundred bytes
- **L1/L2/L3 Cache (SRAM):** Built directly onto the CPU die (or very close). They hold copies of frequently accessed RAM data. The hardware automatically moves data from RAM into L3, then L2, then L1 based on spatial and temporal locality—without any programmer intervention.
  - **L1:** ~1 ns, ~32KB per core
  - **L2:** ~4 ns, ~256KB per core
  - **L3:** ~12 ns, ~8-32MB shared across cores
- **RAM (DRAM):** Main memory. 
  - *Access time:* ~100 ns
  - *Size:* 16GB–1TB
  - *(Compared to L1, RAM is 100x slower).*
- **Persistent Storage (SSD / HDD):** The source of truth for files and databases.
  - **NVMe SSD:** ~10,000 ns (10 µs)
  - **SATA SSD:** ~100,000 ns (100 µs)
  - **HDD:** ~10,000,000 ns (10 ms) *(A full 100,000x slower than RAM).*

> **🧠 The Critical Insight:**
> The OS and the CPU hardware are already doing aggressive caching for us (L1/L2/L3 and the Page Cache). When an application reads a file from disk, the OS doesn't necessarily go to the physical disk every time; it often serves it from the OS Page Cache. We should never try to manually manage CPU caches—that's the hardware's job. As backend engineers, our job starts at the application-level cache.

---

### 3.2 OS-Level Caching (Page Cache / Buffer Cache)
Before the application even gets a chance to cache anything, the operating system is already caching disk I/O behind the scenes.

- **What it is:** A portion of the system's free RAM that the OS uses to store recently accessed disk blocks (pages). When we call `read()` on a file, the OS first checks the Page Cache. If the block is there, it copies it directly to the application's memory without touching the disk.
- **Why it matters:** This is why repeatedly reading the same configuration file or a static asset is fast, even if we don't implement any caching in our code. It's also why database engines like PostgreSQL use `fsync` and `O_DIRECT` flags—they sometimes bypass the Page Cache to avoid double-caching and to ensure they have control over when data actually hits the physical disk.

> **Mental Model:** Think of the Page Cache as a free, automatic "neighborhood cache" for all file operations. It operates on physical disk blocks, not on application's logical keys (like `user:123`). It caches the raw bytes of the database file. If we query `user:123` from a database, the DB engine might read page 42 from the `.ibd` file; the OS caches page 42. If we query `user:456` and it happens to live on that same page, it's a cache hit at the OS level, even though the application-level cache missed.

---

### 3.3 Application-Level Caches (In-memory vs. Distributed)
This is where we, as engineers, have full control. We are storing logical results—the deserialized JSON of a user profile, the rendered HTML of a dashboard, the result of a complex aggregation query.

- **In-memory (Local):** The cache lives inside the application's heap (e.g., a Java `ConcurrentHashMap`, a Python dictionary, or a library like Caffeine/Guava). Access is a simple pointer dereference—sub-microsecond.
- **Distributed (Remote):** The cache lives on a separate set of servers (e.g., Redis, Memcached). The application makes a network call (TCP/Unix socket) to fetch the data. Access is ~0.5–2ms (network round-trip within a datacenter).

>*The key distinction: Application-level caches operate on domain objects, not physical blocks. We decide what key to use and what value to store. The trade-off here is control vs. complexity.*

---

### 3.4 Infrastructure Caches (CDN, DNS, Reverse Proxy / API Gateway)
These operate outside the application code entirely. They are networking-level caches that sit between the user and the backend.

- **CDN (Content Delivery Network):** Caches static assets (images, CSS, JS, video) at edge nodes geographically closer to the user. Operates on the URL as the key. A hit means the request never reaches the origin servers. Massive latency benefits (hundreds of ms).
- **DNS Caching:** Every OS, browser, and recursive resolver caches DNS records (domain → IP). This prevents the overhead of a full DNS lookup chain for every single request.
- **Reverse Proxy / API Gateway (e.g., Varnish, Nginx, CloudFront):** These cache full HTTP responses (`GET` requests) at the gateway level, before the request even hits the application container. Great for public, read-only APIs or product catalog pages.

**hierarchy placement:** They are the outermost layer. They have the highest latency (because they traverse the internet), but they have the largest capacity (they can cache terabytes of data across millions of users). They also reduce load on the application more effectively than any internal cache because they stop the request at the door.

---

## 4. Fundamental Cache Topologies (Local vs. Distributed)

Now we zoom in on the two ways to implement application-level caches. This is a foundational architectural decision that affects consistency, performance, and operational complexity.

### 4.1 Local / In-Process Caches
- **What it is:** The cache is a data structure (usually a HashMap + an eviction policy) that lives inside the memory heap of the application process. Every instance of the service has its own completely independent cache.
- **Examples:** Caffeine (Java), Guava Cache (Java), `lru_cache` (Python), `groupcache` (Go).
- **Performance:** Blazing fast. Read path: `application → local heap → return`. No serialization, no network, no system calls (just a pointer dereference). Latency: ~100–300 ns.
- **Capacity:** Limited by the JVM/process heap. If there is a 4GB heap, we can only allocate maybe 1GB to caching without causing GC pressure.

> **🚨 The Critical Nuance:**
> Because each instance has its own copy, the caches are highly inconsistent across instances. If Instance A caches `user:123` and then a `PUT /users/123` request lands on Instance B, Instance B can invalidate its own local cache, but Instance A will continue serving the stale version until its TTL expires. There is no built-in mechanism to inform other instances.

### 4.2 Distributed Caches (e.g., Redis, Memcached)
- **What it is:** A separate cluster of servers dedicated solely to storing key-value pairs in memory. All application instances talk to the same cluster over the network.
- **Examples:** Redis, Memcached, Hazelcast, Apache Ignite.
- **Performance:** Slower than local. Read path: `application → serialize key → TCP network call → server lookup → network response → deserialize value`. Latency: ~0.5ms – 2ms.
- **Capacity:** Massive. We can add nodes to the cluster and scale to hundreds of gigabytes or terabytes of RAM shared across all services.

> **🚨 The Critical Nuance:**
> This gives us cross-instance consistency. If Instance A invalidates `user:123` in Redis, Instance B will immediately see that invalidation (or miss) on the next read, because they share the same centralized store. However, we introduce a network dependency—if Redis is slow or down, the application must handle that gracefully.

---

### 4.3 The Trade-off (Speed vs. Consistency vs. Capacity)
This is the heart of the decision. There is no "best" topology; there is only the right one for the workload.

| Dimension | Local / In-Process | Distributed / Centralized |
| :--- | :--- | :--- |
| **Speed (Latency)** | **Extreme.** ~100ns. No network overhead. | **Medium.** ~1ms. Network and serialization dominate. |
| **Consistency** | **Weak.** Each instance has its own view. Stale data is almost guaranteed. | **Strong (relative).** All instances see the shared source of truth. Invalidation is immediate. |
| **Capacity** | **Small.** Limited by process heap per instance. | **Massive.** Independent cluster can scale out massively. |
| **Cost** | **Free (mostly).** Uses existing application memory. | **Expensive.** Requires dedicated servers & operational overhead. |
| **Failure Domain** | **Tied to application.** Instance restart clears cache, doesn't affect others. | **Independent.** If cluster crashes, all apps lose cache (possible Thundering Herd). |
| **Complexity** | **Trivial.** Just a library import. | **High.** Manage connections, retries, failovers, partitions. |

#### 💡 The Decision Mental Model:

- **Use Local when:** The data is expensive to compute but small, changes infrequently, and slight inconsistency across instances is acceptable (e.g., global config map updated daily, static reference data). Or when you *cannot* afford the extra 1ms of network latency (e.g. HFT, real-time video).
- **Use Distributed when:** The cache needs to be large, shared across services (e.g., Auth tokens, product catalog across microservices), or strong invalidation is critical to avoid showing stale data to users after updates.

#### 🏆 The Production Pattern: The Two-Level Cache
Check L1 (Local) first. If it misses, check L2 (Distributed). If that misses, go to the DB.
*This provides the speed of local for absolute hot keys, the consistency of distributed for other keys, and the DB as the final fallback.*

---

⬅️ **[Previous: 1. Introduction to Caching](01_Introduction.md)** | 🏠 **[Back to README](README.md)** | **[Next: 3. Cache Updating Strategies ➡️](03_Updating_Strategies.md)**
