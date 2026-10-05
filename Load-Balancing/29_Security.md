# 29. Load Balancing Security

Because the Load Balancer sits at the absolute edge of your network architecture, it is the primary target for malicious attacks. Structuring the security of your Load Balancer correctly is the difference between a minor annoyance and a catastrophic data breach.

## 29.1 Encryption: TLS & mTLS

*   **TLS (Transport Layer Security):** The modern replacement for SSL. When a user connects to `https://`, the Load Balancer performs the intensive mathematical handshake to decrypt the traffic (TLS Termination). The traffic is then sent *unencrypted* over your private datacenter network to backend servers to save CPU power.
*   **mTLS (Mutual TLS):** Used in High-Security environments (like Zero Trust Service Meshes or Banking networks). Not only does the Client verify the Load Balancer is legally who it says it is, but the **Load Balancer explicitly cryptographically verifies the identity of the Client/Microservice** before allowing the connection. Every internal backend server encrypts traffic to every other internal server. 

## 29.2 Understanding Header Spoofing

We learned in Chapter 28 that the Load Balancer injects the `X-Forwarded-For` header so the backend knows the User's real IP address.

**The Spoofing Attack:**
Imagine you have an Admin dashboard that only allows requests from IP `10.0.0.5`. 
If a hacker physically writes their own HTTP header `X-Forwarded-For: 10.0.0.5` and sends it to a poorly configured Load Balancer, the Load Balancer might blindly pass the hacker's fake header to the backend! The backend server reads the fake IP, thinks the hacker is an Admin, and grants them full access.

**The Fix:**
You absolutely must configure your Load Balancer (or WAF) to aggressively **strip and overwrite** any `X-Forwarded-For` headers provided by the client, strictly trusting only the IP address physically read from the actual TCP socket connection.

## 29.3 Public vs Private (Internal) Load Balancers

Senior architectures strictly isolate traffic into physical networking tiers (VPCs and Subnets).

1.  **Public Load Balancer (External-facing):** Sits in a "Public Subnet". It holds a real, public IP address (e.g., `8.8.8.8`). Anyone on the planet can ping it. Its only job is to intercept public traffic, run WAF security rules, terminate TLS, and forward traffic backward.
2.  **Private Load Balancer (Internal):** Sits in a "Private Subnet" deep within your datacenter. It has a local IP address (e.g., `10.0.1.55`). It is physically impossible to access from the public internet. 

**Architectural Flow:**
`Internet` ➡️ `Public ALB` ➡️ `Frontend React Servers` ➡️ `Internal ALB` ➡️ `Backend API Servers` ➡️ `Database`.

By using Internal LBs, your crucial Backend API servers are mathematically shielded from being directly hit by a stray DDoS attack or public hacker.

---

⬅️ **[Previous: 28. Observability & Logging](28_Observability.md)** | 🏠 **[Back to TOC](README.md)** | **[Next: 30. Load Balancing Databases ➡️](30_Load_Balancing_Databases.md)**
