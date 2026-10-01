# 24. Rate Limiting & Load Balancing

Load Balancers are designed to distribute traffic, but what happens when the total incoming traffic vastly fundamentally exceeds the entire processing capacity of your backend cluster? Or worse, what if a single malicious actor is intentionally flooding your servers to crash them (Denial of Service)?

A Load Balancer cannot just blindly route everything. It must mathematically protect the backend infrastructure. This introduces **Rate Limiting**—the architectural bouncer of your system.

## 24.1 Why Rate Limiting?

Rate Limiting restricts the absolute number of requests a client can make within a specific time window (e.g., `100 requests per minute`). 
If the limit is exceeded, the Load Balancer instantly rejects the traffic and returns an `HTTP 429 Too Many Requests` error. 

**Senior Developer Scenarios:**
*   **Preventing Resource Starvation:** Stopping one poorly-coded internal microservice from accidentally monopolizing 100% of your database bandwidth.
*   **Cost Control:** Blocking scrapers from looping through your API 1,000,000 times a day and driving up your AWS billing.
*   **Preventing Brute Force:** Blocking a hacker attempting 500 password combinations per second on your login endpoint.

## 24.2 IP-Based vs User-Based Rate Limiting

*   **IP-Based Limiting:** The Load Balancer tracks the incoming IP address. 
    *   *Flaw:* If 500 college students are connecting from the same university Wi-Fi, they all share one single public IP (due to NAT). Banning that IP accidentally bans 500 legitimate users.
*   **User-Based (Token) Limiting:** The Load Balancer reads the `Authorization: Bearer <token>` header or API Key.
    *   *Advantage:* It uniquely identifies the exact user, regardless of what IP they are connecting from. This is heavily preferred for authenticated APIs.

## 24.3 Global vs Distributed Rate Limiting

Rate Limiting implies the Load Balancer holds "State" (e.g., keeping a counter of how many times User A has connected).
*   **Local In-Memory Limiting:** The Load Balancer (Node 1) keeps a counter in its own local RAM. If you have 5 Load Balancers distributing traffic, a user might be allowed `10 req/min`, but they can maliciously hit all 5 limiters independently, allowing `50 req/min`.
*   **Distributed (Global) Rate Limiting:** All 5 Load Balancers connect to a centralized, massively fast external database (almost always **Redis**) to track the mathematical counters. Redis becomes the absolute source of truth globally.

## 24.4 The Token Bucket Algorithm

This is the algorithm natively used by **Amazon AWS API Gateway**.
Imagine a physical bucket for every user. The bucket has a maximum capacity (e.g., `10 tokens`). 
*   **Refill Rate:** An automated background process drops exactly `1 token` into the bucket every second.
*   **Request Execution:** When the user makes an HTTP request, the Load Balancer takes 1 token out of the bucket. If the bucket is empty, the Load Balancer returns a `429 Error`.
*   **Senior Dev Insight (Bursting):** Token Bucket allows for sudden **Bursting**. If a user hasn't made a request in 10 seconds, their bucket is completely full. They can theoretically fire off 10 requests in a single millisecond safely!

## 24.5 The Leaky Bucket Algorithm

Used prominently by **NGINX** (as a FIFO Queue).
Imagine a bucket with a literal hole in the bottom. 
*   Traffic aggressively pours into the top of the bucket at any completely random, chaotic speed (bursts).
*   Traffic "leaks" out of the bottom of the bucket at a **strictly constant mathematical rate** (e.g., exactly `5 requests per second`).
*   If traffic pours in faster than it leaks out, the bucket fills up. If the bucket spills over, incoming traffic is instantly discarded (`429 Error`).
*   **Senior Dev Insight (Smoothing):** Unlike Token Bucket, Leaky Bucket fundamentally completely destroys bursts. It explicitly forces incoming chaotic spikes to physically smooth out into a strictly constant stream of traffic before it hits your backend servers.

## 24.6 Fixed Window Counter

A very simple implementation used strictly in low-complexity environments. 
Imagine a limit of `100 requests per minute`.
1. The Load Balancer keeps a counter for the current chronological minute (e.g., `10:00 to 10:01`).
2. When the clock hits exactly `10:01`, the system blindly resets the counter to `0`. 

**The Fatal Flaw (The Burst Problem):** 
A clever user can send 100 requests at `10:00:59`, and then 100 more requests at `10:01:01`. Even though the limit was `100 per minute`, the backend was just violently slammed with **200 requests within a 2-second physical window**, potentially crashing the servers.

## 24.7 Sliding Window (Log / Counter)

To aggressively fix the "Fixed Window Burst Problem," advanced Load Balancers use a **Sliding Window**.
Instead of looking at a static clock minute (10:00 to 10:01), the Load Balancer explicitly looks backward in time exactly 60 seconds from the *current physical millisecond*.

*   **Sliding Window Log:** The Load Balancer literally saves the exact UNIX timestamp of every single request in a Redis Sorted Set. It physically counts the timestamps in the last 60 seconds. *Flaw: Horribly heavy on memory if a user makes millions of requests.*
*   **Sliding Window Counter:** A brilliant mathematical compromise that takes the previous minute's traffic (e.g. 50 requests) and weights them mathematically against the current minute's traffic using a percentage formula to tightly approximate the sliding window curve without storing massive arrays of timestamps in memory.

---

### 24.8 Senior Developer Matrix: Rate Limiting Algorithms

| Algorithm | How it handles sudden Traffic Spikes (Bursts) | Memory Footprint in Redis | Primary Use Case |
| :--- | :--- | :--- | :--- |
| **Token Bucket** | **Allows Bursting.** Excels at handling sudden spikes if the bucket is full. | Very Low (Just tracking Tokens & Last Refill time). | Public APIs (AWS API Gateway). Users like fast spikes. |
| **Leaky Bucket** | **Destroys Bursting.** Violently smooths out all spikes into a flat, safe trickle. | Very Low (FIFO Queue size). | Asynchronous processing, protecting fragile legacy databases (NGINX). |
| **Fixed Window** | **Fails at Bursting.** Allows 2x absolute limit at the boundary edges of the clock turn. | Extremely Low (Just a basic counter). | Basic, non-critical internal rate limiting. |
| **Sliding Window** | **Perfectly prevents Boundary Bursting.** Scientifically accurate limits at any literal millisecond. | Very High (If using Logs) / Moderate (If using Counters). | Tiered SaaS limits (e.g., "Developer Plan gets exactly 500 Req/Hr"). |

---

⬅️ **[Previous: 23. Load Balancing & Queues](23_Load_Balancing_Queues.md)** | 🏠 **[Back to TOC](README.md)** | **[Next: 25. CDN, WAF & Load Balancing ➡️](25_CDN_WAF_Load_Balancer.md)**
