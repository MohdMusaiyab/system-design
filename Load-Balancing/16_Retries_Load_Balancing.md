# 16. Retries & Load Balancing

Networks are inherently unreliable. Packets will drop, connections will magically snap, and servers will throw transient 500 errors. In a distributed system, a Load Balancer cannot simply pass an ugly network error back to the user without at least _trying_ to securely fix it behind the scenes.

This introduces the **Retry Mechanism**—the true architectural lifeline of the internet.

## 16.1 The Retry Mechanism

A **Retry Mechanism** precisely allows the Load Balancer (or the client) to automatically re-send a formally failed HTTP request to a logically completely different backend server before giving up safely.

If **Server A** drops your database connection mid-flight, the Load Balancer invisibly securely redirects your request forcefully to **Server B**. If Server B successfully smoothly processes it safely, the user natively experiences a slight 100ms delay, but sees a successful `HTTP 200 OK`. The failure is beautifully abstracted away organically.

## 16.2 Retryable vs Non-Retryable Errors

You absolutely cannot blindly retry everything cleanly. A Load Balancer must natively act intelligently correctly based on the specific _type_ of HTTP error.

- **Retryable Errors:**
  - `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout`.
  - Network-layer TCP Connection issues (e.g. `Connection Refused`).
  - _Teacher's Note:_ These securely imply the server itself logically is fundamentally having a bad time, and elegantly gracefully trying a totally securely cleanly securely different server mathematically logically might explicitly cleanly effortlessly logically cleverly actually work flawlessly.
- **Non-Retryable Errors:**
  - `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`.
  - _Teacher's Note:_ These indicate client-side issues or permanent rejections. Retrying them blindly will only worsen traffic without changing the outcome.

## 16.3 Retry Storms

A **Retry Storm** occurs when a sudden wave of failures causes numerous clients or load balancers to simultaneously retry their requests, overwhelming recovering backend services with an avalanche of synchronized traffic.

- _Teacher's Note:_ Think of it like shouting in a crowded room because you didn't hear someone; if everyone shouts at once, communication breaks down entirely rather than improving.

## 16.4 Retry Amplification

**Retry Amplification** happens when retries multiply the total request volume across downstream services, turning a minor blip into an exponential resource drain.

- _Teacher's Note:_ If 1,000 initial requests fail and each retries 3 times, you've instantly generated 3,000 new requests. Multiply that across multiple layers of microservices, and your infrastructure suffocates under self-inflicted load.

## 16.5 Exponential Backoff

To prevent Retry Storms, you must strategically space out your network retries.
**Exponential Backoff** forces the Load Balancer (or upstream client) to wait longer and longer between each retry.

- **Attempt 1:** Wait 1s
- **Attempt 2:** Wait 2s
- **Attempt 3:** Wait 4s
- **Attempt 4:** Wait 8s

_Teacher's Note:_ This elegantly acts as a massive pressure relief valve. If the database is overwhelmed, giving it 8 seconds to breathe could be the exact margin it needs to clear its deadlocks and recover.

## 16.6 Jitter

Exponential Backoff is brilliant, but it has a fatal flaw. If 5,000 requests all fail at the exact same millisecond, they will all wait exactly 1 second, and then blindly retry at the exact same time. The database gets slammed again.

**Jitter** mathematically injects random chaos into the backoff timer.

- Instead of waiting exactly `1.0s`, a request waits `1.2s`.
- Another waits `0.8s`.

_Teacher's Note:_ Jitter smooths out the spike entirely. It transforms a devastating, synchronized traffic spike into a manageable, rolling wave of traffic over time.

## 16.7 Retry Limits (Circuit Breakers)

You cannot blindly retry forever.

A Load Balancer must enforce strict **Retry Limits** (e.g., maximum 3 retries) globally.
Modern Load Balancers enforce a **Circuit Breaker** pattern. If the LB notices that 95% of _all_ retries to Server A are failing, the LB will "trip the breaker" and mechanically stop retrying Server A entirely for the next 60 seconds to allow it to logically heal.

## 16.8 Idempotency (The Golden Rule)

_Teacher's Note: This is exactly the most crucial architectural concept of this specific module._

You must NEVER blindly retry non-idempotent HTTP requests.
**Idempotency** means that executing a request yields the exact same state, no matter if you run it 1 time or 1,000 times.

- `GET /profile` (Idempotent. Safe to retry).
- `POST /checkout` (NOT Idempotent! Never blindly retry).

If the LB retries a `POST` request because the first server timed out, the first server might have legitimately processed the payment, but just failed to quickly send the `HTTP 200 OK` back. Retrying it will accidentally double-charge your customer!

## 16.9 Timeout + Retry Interaction

Retries always amplify the overall user-facing latency.

If the user has a **Hard Application Timeout** of 5 seconds, and your Load Balancer's retry backoff algorithm technically pushes the total wait time to 8 seconds, the Frontend UI will just crash and return an error before the Load Balancer even finishes its retries.
You must intentionally align your Frontend UI Timeouts to comfortably exceed your Backend Retry Budgets.

---

⬅️ **[Previous: 15. Load Balancing & Autoscaling](15_Load_Balancing_Autoscaling.md)** | 🏠 **[Back to TOC](README.md)** | **[Next: 17. Load Balancing & Resilience Patterns ➡️](17_Load_Balancing_Resilience_Patterns.md)**
