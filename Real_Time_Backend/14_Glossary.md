# 14. Glossary and Terminology

Below is the definitive glossary of real-time backend terminology used throughout this guide. Think of this as your quick-reference sheet for system design interviews and production discussions regarding persistent connections.

| Term | Category | Definition / Our Mental Model |
| :--- | :--- | :--- |
| **Short Polling** | Communication Pattern | The client repeatedly sends HTTP requests to check for new data. Simple to build, but incredibly inefficient in terms of network overhead and server concurrency. |
| **Long Polling** | Communication Pattern | The client sends an HTTP request, and the server intentionally holds it open indefinitely until an event occurs. Reduces empty responses, but requires complex timeout and reconnect handling. |
| **Server-Sent Events (SSE)** | Communication Pattern | A unidirectional (server-to-client) persistent streaming channel operating entirely on standard HTTP. Ideal for live feeds, stock tickers, and AI token generation. |
| **WebSocket** | Communication Pattern | A fully persistent, full-duplex (bidirectional) TCP protocol. Initiated over HTTP, it upgrades to allow low-latency, two-way binary or text message framing. |
| **HTTP Upgrade** | Protocol Mechanic | The handshake process wherein a client asks an HTTP server to upgrade their active TCP connection from standard HTTP/1.1 to the WebSocket protocol (yielding a `101 Switching Protocols` response). |
| **Frame** | Protocol Mechanic | The fundamental atomic unit of data on the WebSocket layer. A large application message can be fragmented into many physical transmission frames. Includes control structures for Ping, Pong, and Close. |
| **Ping / Pong** | Reliability | The heartbeat control-frame mechanism built directly into the WebSocket protocol to routinely monitor connection vitality and aggressively detect silently dropped TCP links (half-open connections). |
| **Stateful Connection** | System Design | A connection physically bolted and pinned into the local RAM of a *specific* backend server. WebSockets are inherently stateful, challenging standard "stateless" HTTP auto-scaling patterns. |
| **Connection Ownership** | Distributed Systems | The reality that a single node "owns" a user's TCP socket. To send a message to User B, the system must forcefully route the payload to whichever specific physical Server Node owns User B's socket. |
| **Pub/Sub Backplane** | System Architecture | An internal message-routing nervous system (e.g., Redis Pub/Sub, NATS, Kafka) that seamlessly bridges independent WebSocket nodes to allow horizontal inter-server broadcasting and fan-out. |
| **Sticky Sessions** | Load Balancing | A load balancing technique that forces all traffic from a specific user session to route to the exact same backend server. While sometimes necessary, it fundamentally *does not* solve cross-server chat broadcasting. |
| **Half-Open Connection** | Networking Trap | A zombie connection where a client loses network unexpectedly (e.g. driving through a tunnel) without sending a clean TCP termination. The server keeps the socket silently in memory leaking resources unless caught by a Ping/Pong heartbeat. |
| **Thundering Herd** | Production Failure | A catastrophic cascading failure caused when a large backend server crashes, dumping 50,000 WebSocket users onto the street simultaneously, all of whom instantly auto-reconnect and overwhelm the remaining infrastructure. |
| **Exponential Backoff and Jitter** | Reliability Pattern | The absolute mathematical necessity of forcing disconnected clients to wait increasing amounts of time (plus randomized jitter) before retrying a connection to mitigate reconnect storms. |
| **Backpressure** | Production Failure | The destructive physics of a fast server (producer) emitting messages significantly quicker than a slow client (consumer) can read them over a poor network, resulting in unbound RAM queues and Out-of-Memory crashes. |
| **Presence System** | Application Feature | Intricate architectures dedicated exclusively to accurately tracking distributed "Online", "Offline", and "Typing" statuses across infinite nodes while tolerating network flakiness. |
| **Fan-Out** | Messaging | The workload of taking one incoming broadcast payload and efficiently multiplying it outward to thousands of connected individual client network sockets in real time. |
| **Message Durability** | Reliability | The guarantee that if a client temporarily disconnects during a broadcast, the message is queued onto physical disk (or durable memory) until they reconnect. WebSockets inherently *do not* provide this. |

---

⬅️ **[Previous: 13. Real-Time Tradeoffs Series](13_Real_Time_Tradeoffs.md)** | 🏠 **[Back to TOC](README.md)**
