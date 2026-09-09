# 11. Migration Strategies

Caches are not static. We must actively upgrade and migrate them.

### 11.1 Why we migrate
- **Provider change:** Moving from self-hosted Redis to a managed service like AWS ElastiCache.
- **Resharding:** Our 3-node cluster is too small; we need to move to a 6-node cluster to handle more keys.
- **Eviction Policy change:** We realize LFU would be better than LRU for our workload, but we need to restart the cluster with the new configuration.
- **Upgrade:** Moving from an older Redis version to a newer one for features or security patches.

---

### 11.2 Zero-Downtime Migration Approaches

#### 11.2.1 Dual-Writes + Backfill (The "Safest")
1. **Phase 1:** Deploy the new cache cluster (Cluster B) alongside the old one (Cluster A).
2. **Phase 2:** Modify our application to write to *both* Cluster A and Cluster B on every SET/DELETE.
3. **Phase 3:** Run a background batch job that reads every key from Cluster A (or the primary database) and writes it to Cluster B. This "backfills" the history.
4. **Phase 4:** Our reads are still pointing to Cluster A. At this point, Cluster B is effectively fully warmed up.
5. **Phase 5:** Flip a configuration flag. All reads now go to Cluster B. Writes continue to both (or we can strategically stop writing to A).
6. **Phase 6:** Decommission Cluster A after verifying metrics (hit ratio, latency).

#### 11.2.2 Gradual Key Migration (The "Traffic Shaping")
We use Consistent Hashing (discussed in Section 9) and slowly increase the weight of the new nodes. We use a feature flag to redirect only 1% of traffic to the new cluster, then 5%, then 20%, then 100%. This allows us to slowly validate the new cluster's behavior under load before fully committing.

#### 11.2.3 Blue-Green Cache Clusters
We fully provision Cluster B based on Cluster A's state. We change the DNS entry or the service discovery address (`redis-cluster.prod -> new-ip`) atomically. All traffic cuts over instantly. If something fails, we instantly cut back.

---

### 11.3 Rollback Planning

Rollback planning is the most important part of any migration.

- We always keep Cluster A alive for at least 48 hours after the migration.
- We ensure our application has a dynamic configuration flag to instantly revert reads back to Cluster A without needing a code deployment.
- We monitor the error rates meticulously. If the Hit Ratio drops significantly or latency spikes during the cutover, **we roll back immediately**. We never try to "fix" the new cluster under heavy production load; we revert and debug offline.

---

⬅️ **[Previous: 10. Production Failure Modes](10_Production_Failure_Modes.md)** | 🏠 **[Back to TOC](README.md)** | **[Next: 12. Unseen Production Realities ➡️](12_Unseen_Production_Realities.md)**
