# Real-Time Backend

Real-time systems are backend systems where information needs to reach users **without the user repeatedly asking for it**.

Traditional applications often work like this:

```text
Client
   │
   │ Request
   ▼
Server
   │
   │ Response
   ▼
Client
```

The client asks, and the server responds.

Real-time applications introduce a different requirement:

> **The server should be able to send information to the client when something happens.**

For example:

- A new WhatsApp message arrives.
- A food-order status changes.
- A stock price changes.
- Someone joins a multiplayer game.
- A notification appears.
- A hospital bed becomes available.
- A user comes online.
- A live match score changes.

This chapter explores how we build these systems, starting from simple polling and gradually moving toward WebSockets and distributed real-time architectures.

---

# Learning Roadmap

```text
                    Real-Time Backend
                           │
             ┌─────────────┴─────────────┐
             │                           │
       Communication                Architecture
             │                           │
     ┌───────┼────────┐          ┌───────┼────────┐
     │       │        │          │       │        │
 Polling   SSE   WebSockets    Scaling  Pub/Sub  Reliability
     │       │        │          │       │        │
     └───────┴────────┘          └───────┴────────┘
                           │
                    Production Systems
```

---

# Chapters

## Chapter 1 — Real-Time Backend Fundamentals

**File:** `01_Real_Time_Fundamentals.md`

We start with the problem itself.

### Topics

- What does "real-time" actually mean?
- Traditional request-response systems
- Why normal HTTP requests are sometimes insufficient
- Short polling
- Long polling
- Server-Sent Events (SSE)
- WebSockets
- WebRTC — overview only
- Evolution from polling to persistent connections
- Latency vs freshness
- Choosing the appropriate communication mechanism

### Goal

Understand **why different real-time technologies exist** before learning how they work internally.

---

## Chapter 2 — Polling and Long Polling

**File:** `02_Polling_And_Long_Polling.md`

Before WebSockets, understand the simpler approaches.

### Topics

- Short polling
- Request intervals
- Wasted requests
- Latency introduced by polling
- Long polling
- Holding an HTTP request open
- How long polling reduces unnecessary requests
- Connection lifecycle
- Advantages and limitations
- Scaling implications
- Thundering herd during reconnects

### Goal

Understand the limitations that eventually led to persistent real-time connections.

---

## Chapter 3 — Server-Sent Events (SSE)

**File:** `03_Server_Sent_Events.md`

SSE provides a simple way for servers to continuously push events to clients over HTTP.

### Topics

- What SSE is
- `EventSource`
- Persistent HTTP connection
- Server → Client communication
- Event format
- Reconnection
- Event IDs
- Heartbeats
- SSE vs polling
- SSE vs WebSockets
- When SSE is a better fit
- Scaling SSE connections

### Goal

Understand when **one-way real-time communication** is enough.

---

## Chapter 4 — WebSockets Fundamentals

**File:** `04_WebSockets_Fundamentals.md`

Now we move to the most important real-time communication mechanism in this roadmap.

### Topics

- What WebSockets solve
- Full-duplex communication
- Persistent connections
- WebSocket lifecycle
- Connection establishment
- HTTP Upgrade
- `101 Switching Protocols`
- Client ↔ Server communication
- Connection state
- Basic WebSocket server/client
- When WebSockets are appropriate

### Goal

Build a strong mental model of **what a WebSocket connection actually is**.

---

## Chapter 5 — WebSocket Protocol Deep Dive

**File:** `05_WebSocket_Protocol.md`

Now we go below the high-level API and understand the protocol itself.

### Topics

- RFC 6455
- HTTP Upgrade handshake
- `Sec-WebSocket-Key`
- `Sec-WebSocket-Accept`
- WebSocket frames
- Frame structure
- Opcodes
- Text frames
- Binary frames
- Close frames
- Ping/Pong
- Fragmentation
- Masking
- Connection lifecycle

### Goal

Understand what is happening **underneath libraries such as `ws` or Gorilla WebSocket**.

---

## Chapter 6 — WebSocket Libraries and Implementations

**File:** `06_WebSocket_Libraries.md`

A protocol is one thing; implementing it in production is another.

### Node.js

- `ws`
- Socket.IO
- `uWebSockets.js`

### Go

- `gorilla/websocket`
- `nhooyr.io/websocket`
- `coder/websocket`
- `gobwas/ws`

### Topics

- Abstraction vs control
- Performance considerations
- Memory usage
- API ergonomics
- Ecosystem
- Reconnection support
- Rooms and namespaces
- Framework lock-in
- When to choose each approach

### Goal

Understand **what the libraries actually provide** instead of treating them as interchangeable.

---

## Chapter 7 — Scaling WebSocket Servers

**File:** `07_Scaling_WebSockets.md`

One WebSocket server is relatively simple.

Multiple servers create a much more interesting problem.

```text
                    Load Balancer
                   /      |      \
                  /       |       \
             WS Server  WS Server  WS Server
                │          │          │
                └──────────┼──────────┘
                           │
                       Pub/Sub
```

### Topics

- Why WebSockets are stateful
- Persistent connections
- Horizontal scaling
- Load balancers
- Multiple WebSocket servers
- Connection ownership
- Sticky sessions / session affinity
- Why sticky sessions don't solve everything
- Redis Pub/Sub as a backplane
- Broadcasting between servers

### Goal

Understand how we move from:

> "One server has WebSocket connections"

to:

> "Thousands or millions of connections are distributed across many servers."

---

## Chapter 8 — Redis Pub/Sub and the Real-Time Backplane

**File:** `08_Redis_PubSub_Backplane.md`

When clients are connected to different WebSocket servers, those servers need a way to communicate.

### Topics

- The backplane concept
- Redis Pub/Sub
- Publishers
- Subscribers
- Channels
- Message propagation
- Cross-server broadcasting
- At-most-once delivery
- What happens when a subscriber is disconnected
- Pub/Sub vs queues
- Why Pub/Sub is not a durable message system
- Redis Streams — conceptual comparison

### Goal

Understand **why a backplane is required** and what Redis Pub/Sub does—and does not—guarantee.

---

## Chapter 9 — Reliability of Real-Time Connections

**File:** `09_Real_Time_Reliability.md`

Real-time connections fail.

Users disconnect. Networks disappear. Servers restart. Deployments happen.

### Topics

- Connection failures
- Reconnection
- Exponential backoff
- Jitter
- Thundering herd
- Heartbeats
- Ping/Pong
- Read deadlines
- Connection timeouts
- Connection draining
- Graceful shutdown
- Message ordering
- Duplicate messages
- At-most-once delivery
- Reconnection recovery

### Goal

Understand how real-time systems behave **when things go wrong**.

---

## Chapter 10 — Backpressure

**File:** `10_Backpressure.md`

A real-time system can generate messages faster than a client can consume them.

```text
Producer
   │
   │ 1000 msg/sec
   ▼
WebSocket Server
   │
   │ 100 msg/sec
   ▼
Slow Client
```

### Topics

- What backpressure means
- Fast producer vs slow consumer
- Buffered messages
- Memory growth
- Bounded buffers
- Dropping messages
- Closing connections
- Applying upstream pressure
- Latest-value vs every-event semantics
- Backpressure in WebSockets

### Goal

Understand how to prevent a slow client from becoming a **resource problem for the entire system**.

---

## Chapter 11 — Presence and Connection State

**File:** `11_Presence_System.md`

Real-time applications often need to answer questions such as:

> Is this user online?

> Which server is this user connected to?

> When was the user last seen?

### Topics

- Presence
- Online/offline state
- Last seen
- Heartbeats and presence
- Ephemeral state
- Redis for presence
- TTL-based presence
- Multiple devices
- Presence across multiple servers
- Race conditions

### Goal

Understand how real-time systems manage **short-lived distributed state**.

---

## Chapter 12 — Real-Time Architecture

**File:** `12_Real_Time_Architecture.md`

Now we combine everything into a production-style architecture.

```text
                         Clients
                    /       |       \
                   /        |        \
                  ▼         ▼         ▼
             WebSocket  WebSocket  WebSocket
               Server      Server      Server
                  \         |         /
                   \        |        /
                    ▼       ▼       ▼
                    Redis Pub/Sub
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
         Presence      Services     Workers
                          │
                          ▼
                       Database
```

### Topics

- Load balancer
- WebSocket servers
- Pub/Sub backplane
- Database
- Presence system
- Connection management
- Authentication
- Message flow
- Failure scenarios
- Horizontal scaling
- Deployment and graceful shutdown

### Goal

Put the individual concepts together into a **real production architecture**.

---

## Chapter 13 — Choosing the Right Real-Time Technology

**File:** `13_Real_Time_Tradeoffs.md`

There is no universal "best" real-time technology.

### Compare

| Technology    | Communication           | Connection            | Typical Use                              |
| ------------- | ----------------------- | --------------------- | ---------------------------------------- |
| Short Polling | Client → Server         | Repeated HTTP         | Simple periodic updates                  |
| Long Polling  | Client → Server         | Held HTTP request     | Older/simple real-time systems           |
| SSE           | Server → Client         | Persistent HTTP       | Notifications, feeds, live updates       |
| WebSockets    | Bidirectional           | Persistent connection | Chat, multiplayer, collaborative systems |
| WebRTC        | Peer-to-peer media/data | Peer connection       | Video/audio, P2P communication           |

### Topics

- Polling vs SSE
- SSE vs WebSockets
- WebSockets vs WebRTC
- One-way vs bidirectional communication
- Reliability requirements
- Infrastructure requirements
- Browser support
- Scaling considerations
- Appropriate use cases

### Goal

Learn to make the decision based on **requirements**, not popularity.

---

# Topics We Will Keep Conceptual

Some topics are important to understand but don't need an entire implementation chapter initially.

### WebRTC

We will understand:

- What WebRTC is
- Peer-to-peer communication
- Signaling
- STUN
- TURN
- Why WebRTC is different from WebSockets

We will **not** deeply implement a video-calling system in this fundamentals track.

---

### Socket.IO Redis Adapter

We will understand:

- Why an adapter is required
- How multiple Socket.IO servers communicate
- Relationship with Redis Pub/Sub

We won't build an entire Socket.IO infrastructure around it.

---

### uWebSockets.js

We will understand:

- Why it exists
- Why it can achieve high throughput
- How it differs from typical Node.js WebSocket libraries
- Its trade-offs

We won't make it the primary implementation framework.

---

### Go Hub Pattern

We will understand the architecture:

```text
                Hub
          /      |      \
       Client  Client  Client
```

and how it manages:

- Connected clients
- Register
- Unregister
- Broadcast

Implementation can be added when we reach the Go implementation section.

---

# What We Will Build Toward

The final mental model should look something like this:

```text
                         ┌───────────────┐
                         │    Clients    │
                         └───────┬───────┘
                                 │
                          Load Balancer
                                 │
                 ┌───────────────┼───────────────┐
                 │               │               │
                 ▼               ▼               ▼
             WS Server       WS Server       WS Server
                 │               │               │
                 └───────────────┼───────────────┘
                                 │
                          Redis Pub/Sub
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
          Presence          Application         Workers
              │               Services              │
              │                  │                  │
              └──────────────────┼──────────────────┘
                                 │
                              Database
```

And we should be able to explain:

> **Why every component exists, what problem it solves, what can fail, and what trade-off it introduces.**

---

# Learning Order

The recommended order is:

```text
01. Real-Time Fundamentals
        ↓
02. Polling & Long Polling
        ↓
03. SSE
        ↓
04. WebSocket Fundamentals
        ↓
05. WebSocket Protocol
        ↓
06. WebSocket Libraries
        ↓
07. Scaling WebSockets
        ↓
08. Redis Pub/Sub Backplane
        ↓
09. Real-Time Reliability
        ↓
10. Backpressure
        ↓
11. Presence
        ↓
12. Real-Time Architecture
        ↓
13. Technology Trade-offs
```

This order intentionally goes from:

**Problem → Simple Solutions → Persistent Connections → Protocol → Libraries → Distributed Scaling → Reliability → Production Architecture**

---

# What This Section Should Give You

By the end of this section, you should be able to look at a requirement such as:

> "Users should receive new messages instantly."

and reason about it systematically:

```text
Do we need real-time?
        │
        ▼
Is communication one-way?
        │
    ┌───┴───┐
   Yes      No
    │        │
    ▼        ▼
   SSE    WebSocket
    │        │
    └───┬────┘
        ▼
Do we need multiple servers?
        │
       Yes
        │
        ▼
Need a backplane
        │
        ▼
Redis Pub/Sub
        │
        ▼
How do failures behave?
        │
        ├── Reconnection
        ├── Heartbeats
        ├── Ordering
        ├── Backpressure
        └── Graceful shutdown
```

The objective is **not to memorize WebSocket APIs**.

The objective is to understand how to design a real-time backend from first principles.

---

## Prerequisites

Before starting this section, you should be comfortable with:

- Client–Server architecture
- HTTP
- HTTP requests and responses
- HTTP status codes
- APIs
- Stateful vs stateless systems
- Databases
- Basic caching
- Basic distributed-system concepts

You do **not** need advanced distributed-systems knowledge to begin.

---

⬅️ **[Back to System Design Fundamentals](../README.md)**

➡️ **[Next: 1. Real-Time Backend Fundamentals](01_Real_Time_Fundamentals.md)**
