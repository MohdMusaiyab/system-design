# 🌍 System Design: Content Delivery Networks (CDN)

Welcome to the absolute ultimate curriculum on **Content Delivery Networks (CDN)** for production-grade system design. Rather than having dozens of disparate files, this curriculum is structurally consolidated into **5 Master Chapters**.

Each chapter extensively covers core mechanics, deep-dive architectural paradigms, performance optimization, and real-world implementation strategies used by Senior Developers.

## 📖 The Curriculum

### 1. [Core Fundamentals & Economics](01_Core_Fundamentals_and_Economics.md)
*   **CDN Fundamentals:** What is it, and how Edge Locations & Points of Presence (PoPs) physically work.
*   **Cost Economics:** Dissecting bandwidth, egress taxes, and major providers (Cloudflare, AWS CloudFront, Fastly).
*   **Request Lifecycle:** The strict flow of Cache Hits vs Cache Misses.
*   **Architecture Mapping:** Push CDNs vs Pull CDNs.

### 2. [Content Optimization & Delivery](02_Content_Optimization_and_Delivery.md)
*   **Asset Caching:** Serving Static HTML/CSS/JS instantly globally.
*   **Dynamic Content Acceleration (DCA):** How CDNs speed up non-cacheable API requests.
*   **Edge Compression:** Utilizing Brotli and Gzip directly on the Edge nodes.
*   **Modern Media Delivery:** Deep dive into Video Streaming architectures, HLS/DASH, and Byte-Range Requests.

### 3. [Advanced Caching Mechanics](03_Advanced_Caching_Mechanics.md)
*   **Origin Control:** Understanding Origin Servers, Reverse Proxies, and Origin Shielding (Tiered Caching).
*   **Cache Directives:** The absolute math behind Time-to-Live (TTL) and strict `Cache-Control` headers.
*   **Cache Manipulation:** Safe Invalidation, Purging Strategies, and manipulating Cache Keys.
*   **Resiliency Protocols:** Masking backend downtime natively utilizing `Stale-While-Revalidate`.

### 4. [Edge Routing & Security](04_Edge_Routing_and_Security.md)
*   **Physical Routing:** How client internet traffic naturally routes to a CDN (DNS Geo-Routing vs Anycast BGP).
*   **Security offloading:** TLS & SSL Termination natively at the Edge.
*   **Perimeter Protection:** Integrating Edge Web Application Firewalls (WAF) and DDoS Mitigation techniques.
*   **Authorization:** Defending video streams from piracy using Token Authentication and Signed URLs.

### 5. [Advanced Edge Architectures](05_Advanced_Edge_Architectures.md)
*   **Serverless Edge Computing:** Writing executing code directly inside physical Edge Routers using Cloudflare Workers & Lambda@Edge.
*   **Failover Design:** Setting up Multi-CDN architectures for total global redundancy.
*   **Disaster Prevention:** Solving the notorious Cache Stampede under extreme viral traffic spikes.
*   **The Master Class:** Mega System Design Case Studies featuring extreme scaling architectures (Netflix, Twitch, Amazon).

### 6. [Important Findings & Metric Summary](06_Important_Findings_and_Summary.md)
*   Flash-summary of all ultimate Edge mechanisms (Anycast, DCA, Token Auth, Origin Shielding).
*   Industry standard numerical limits and cost metrics (Egress Tax pricing, exact milliseconds for Edge Latencies).

---

_Curriculum mapped for Senior Software Engineering standards._
