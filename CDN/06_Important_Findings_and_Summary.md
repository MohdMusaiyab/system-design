# 6. Important Findings & Metric Summary

This final section condenses the mechanics of the entire Content Delivery Network study into high-impact, rapid-fire technical definitions and empirical metrics. It serves as a rapid reference guide for architectural decision-making.

---

## ⚡ Core Architectural Metrics & Benchmarks

When evaluating network throughput or analyzing architectural bottlenecks, these industry-standard empirical values dictate design limitations:

| Metric / Action | Standard Architectural Value | Why it matters |
| :--- | :--- | :--- |
| **Edge Node Latency** | `< 10ms` | The time it takes for a user to hit their local Edge PoP. Bypasses the 200ms transcontinental "Speed of Light" network penalty. |
| **Brotli Compression Savings** | `20% to 30%` | Brotli mathematically compresses JSON/HTML packages 20-30% smaller than legacy Gzip, yielding massive throughput savings. |
| **Bandwidth Egress Tax** | `~$0.09 per GB` | The baseline cost penalty AWS/GCP charges when data physically leaves the Origin datacenter. A CDN neutralizes this by acting as a highly efficient egress shield. |
| **Target Cache Hit Ratio (CHR)**| `> 95%` | The golden efficiency metric. If an infrastructure maintains a 95%+ Cache Hit Ratio, the Origin server mechanically computes merely 5% of all global internet requests natively. |
| **Global Cache Purge Delay** | `~150 milliseconds` | The bleeding-edge threshold (driven by Fastly) for structurally invalidating a memory object across every single server globally. Mandatory for real-time live data state integrity. |

---

## 📖 Key Mechanical Summary

### A. Routing & Infrastructure
*   **Anycast BGP:** An underlying physical routing matrix where 5,000 global servers strictly broadcast the exact same IP address (e.g., `1.1.1.1`). Hardwired ISP routers algorithmically execute shortest-path logic to locally sinkhole traffic into the geographically closest Edge Node, bypassing volatile DNS failover mechanics natively.
*   **Point of Presence (PoP):** Massively interwoven data centers anchored rigidly at strategic internet exchange points (e.g., Frankfurt, Tokyo) where the CDN peers heavily with major regional ISPs.
*   **Origin Shielding (Tiered Caching):** Architecting a specialized local middle-tier proxy exactly between the Global Edges and the Origin. It intercepts volatile global Cache Misses natively, collapsing them into exactly one outbound Origin socket (Request Coalescing) to prevent catastrophic database exhaustion.

### B. Header Directives & Resiliency
*   **Time-to-Live (TTL):** The rigid, explicitly declared temporal boundary before an Edge Node mathematically purges an object from its hardware RAM. Enforced natively via HTTP `Cache-Control` logic.
*   **`s-maxage` vs `max-age`:** `s-maxage` exclusively commands the centralized Shared Cache (the CDN Edge Node) dictating its specific temporal retention, whereas `max-age` strictly governs the individual client's local browser memory.
*   **`stale-while-revalidate`:** The ultimate resiliency failsafe. If a cache strictly expires, and the Origin database subsequently crashes, the Edge node forcefully returns the expired asset natively to the client while silently opening an asynchronous shadow connection to probe the dead database, fully masking backend downtime.

### C. Edge Output Optimization
*   **Dynamic Content Acceleration (DCA):** Routing fundamentally uncacheable API payloads (like checkout transactions) entirely through a CDN. The CDN physically forces the packet off the congested BGP public internet and shunts it severely down private, dedicated global submarine fiber-optic cables to aggressively stabilize jitter.
*   **Cache-Busting (Fingerprinting):** Discarding the reliance on API `PURGE` commands for static files. Instead, developers dynamically inject volatile cryptographic hashes intrinsically into the production filename (`styles.1a2b.css`) to enforce spontaneous, organic Cache Misses during deployments.
*   **Byte-Range Mechanics:** The engineering protocol behind video timeline seeking. The client injects `Range: bytes=500-1000` into the HTTP payload, triggering the cache to physically slice isolated memory blocks from a 50GB file utilizing an immediate `HTTP 206 Partial Content` resolution.

### D. Deep Perimeter Security
*   **Edge TLS Termination:** The physical Offloading of intensive asymmetric cryptographic handshakes strictly away from the Origin backend and onto Edge hardware, utilizing `0-RTT` connection mechanisms to negate secure TCP handshake delays.
*   **Token Authorization & Signed Hash URLs:** Neutralizing video piracy scrapers. Altering the asset URL to structurally bind a localized Unix expiration timestamp and a cryptographic HMAC secret (`&expires=XYZ&sig=123`). The Edge proxy computes the hash calculation natively, rejecting unauthenticated clients instantly via an `HTTP 403 Forbidden` drop without invoking the Origin.

---

⬅️ **[Previous: 5. Advanced Edge Architectures](05_Advanced_Edge_Architectures.md)** | 🏠 **[Back to TOC](README.md)** | 🏁 **END OF CDN MODULE**
