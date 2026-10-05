# 21. Load Balancing & Caching

When we talk about Caching strictly in the context of Load Balancing, we are talking about the Load Balancer natively caching entire HTTP responses directly at the network edge so the request _never even physically reaches your backend infrastructure_.

Here is exactly how a Load Balancer operates as a high-speed HTTP cache.

## 21.1 Reverse Proxy Caching

A modern Load Balancer (like Nginx, HAProxy, or AWS ALB) is a Reverse Proxy. It can intercept an incoming HTTP request, check its own internal RAM or disk, and serve the response instantly.

- **Workflow:**
  1. User requests `GET /api/trending`.
  2. The Load Balancer checks its internal cache.
  3. **Cache Miss:** The LB safely routes the request to Backend Server A. The server responds with JSON. The LB _saves_ that JSON in its memory, and then logically forwards it to the user.
  4. **Cache Hit:** The next 10,000 users request the exact same URL. The LB serves the JSON directly from its own cache. **Zero traffic reaches the backend.**

## 21.2 Static vs Dynamic Caching

- **Static Asset Caching:** The Load Balancer natively caches all `.jpg, .css, .js` files indefinitely based on file hashes. Your expensive backend application servers should strictly only be used for processing complex business logic, absolutely never for serving static images.
- **Dynamic API Caching (Micro-caching):** Caching a purely dynamic database-driven API response (like `GET /stock-price`) for an extremely aggressive, short duration (e.g., exactly 1 to 5 seconds).

## 21.3 Solving the Thundering Herd with Micro-Caching

Micro-caching at the Load Balancer is the ultimate Senior Engineer weapon against massive Traffic Spikes (The Thundering Herd).

If Elon Musk tweets a link to your API, you might suddenly get 50,000 requests per second to `GET /product/1`.
Instead of your backend Redis cluster trying to serve 50,000 requests and melting down, you tell your Load Balancer to cache the API response for exactly **2 seconds**.

- **The Result:** The Load Balancer physically only forwards exactly **1 request** to the backend every 2 seconds. The other 49,999 requests per second are served directly from the Load Balancer's edge memory without the backend servers even knowing a spike occurred!

## 21.4 Cache-Control Headers

How does the Load Balancer know what is safe to cache and what isn't? It mathematically relies on standard HTTP Headers returned by the Backend Application.

- `Cache-Control: public, max-age=3600`: The Backend tells the LB: _"You can safely cache this response for exactly 3600 seconds (1 hour)."_
- `Cache-Control: private, no-store`: The Backend tells the LB: _"This is sensitive User Profile data. NEVER cache this on the Load Balancer."_

By using HTTP Headers, the backend code strictly controls the Load Balancer's caching behavior without requiring manual LB config file changes.

## 21.5 Cache Purging (Invalidation)

What happens if an Admin updates a Product's price, but the Load Balancer's `max-age` still has 59 minutes left? The LB will serve incorrect, stale prices to customers.

We fix this structurally using **Cache Purging**:

1.  Admin updates the Database.
2.  The Backend Application fires an explicit `PURGE` API request directly to the Load Balancer (or CloudFront CDN).
3.  The LB mathematically deletes that specific URL from its RAM.
4.  The next user request cleanly triggers a Cache Miss, forcing the LB to fetch the fresh price from the Backend.

---

### 21.6 Senior Developer Matrix: Backend Cache vs Load Balancer Cache

| Feature                        | Backend Cache (Redis/Memcached)                                             | Load Balancer Cache (Nginx/CDN)                                                                   |
| :----------------------------- | :-------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------ |
| **Location**                   | Lives strictly inside your internal private network.                        | Lives securely at the public Network Edge.                                                        |
| **What it caches**             | DB Query Results, Session IDs, complex object graphs.                       | Entire HTTP Responses (JSON, HTML) and Static Assets.                                             |
| **Thundering Herd Protection** | Vulnerable. The DB will crash if all Servers miss the cache simultaneously. | **Immune.** Eliminates the herd at the edge. The backend never safely sees the spike.             |
| **Invalidation Method**        | Direct `Redis.del(key)` commands.                                           | HTTP `PURGE` webhooks or TTL Expiry.                                                              |
| **Security Risk**              | Low (Internal only).                                                        | **High** (Accidentally caching a private Authenticated response publicly exposes sensitive data). |

---

---


