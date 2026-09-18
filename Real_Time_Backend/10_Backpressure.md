# Chapter 10 — Backpressure

In traditional request-response architectures (like a standard REST API), the client maintains ultimate control over the flow of data.

```text
Client ──► "Give me 10 records" ──► Server
Client ◄── "Here are 10" ────────── Server
```

If the client's processor is slow, or their internet lags, they simply wait before asking for the next batch. The server does not care. It only works when explicitly asked.

However, in a real-time push architecture (like WebSockets or SSE), the **server controls the flow of data**.

What happens if the server generates data faster than the client can possibly download it?

This introduces one of the most hidden, dangerous, and consistently fatal problems in distributed real-time systems: **Backpressure**.

---

# 10.1 Fast Producer vs Slow Consumer

Imagine you are building a real-time analytics dashboard streaming live internet traffic.

The Server (the Producer) is hooked up to a high-speed backplane and is generating **10,000 events per second**.

```text
Server ──► 10,000 msgs/sec
```

One specific Client (the Consumer) is connected to your platform while riding a train, relying on a deeply flawed, slow 3G mobile network. Their smartphone can only download and process **100 events per second**.

```text
Server (10,000 events / sec)
       │
       │  (9,900 messages piling up every single second)
       ▼
Client (100 events / sec)
```

Where exactly do those extra 9,900 messages go?

They don't magically disappear into the atmosphere. They begin to pile up in Server memory.

---

# 10.2 Buffers and Infinite Memory Growth

When the server backend code attempts to write a message down a TCP socket to a slow client, the OS kernel intervenes:

> "The client's network window is completely full right now. Please hold this message in your memory buffer until there is physical space on the wire."

So, the WebSocket server obediently places the message into a local memory **Buffer**.

```text
Server RAM
┌───────────────────────────────────────────────┐
│ Memory Queue for Client A:                    │
│ [Msg 1] [Msg 2] [Msg 3] ... [Msg 9900]        │
└───────────────────────────────────────────────┘
```

If the client remains slow, that buffer grows by thousands of messages a second.

If the buffer is allowed to grow infinitely, the WebSocket server process will eventually consume 100% of the machine's RAM and violently crash.

If you have 50,000 concurrent clients connected to a single node, and just 5% of them enter a tunnel and experience bad internet, their growing memory buffers will seamlessly bring down a massive 64GB server in a matter of minutes.

This means a purely slow client can weaponize their bad internet connection to accidentally DDOS and crash your entire cloud infrastructure.

---

# 10.3 Defining Backpressure

**Backpressure** is the physical or logical resistance that a system forcefully applies when it cannot process incoming data fast enough.

In a mathematically healthy system, when the Consumer slows down, they apply physical "backpressure" up the chain to the Producer, forcefully demanding that the Producer slows down as well.

```text
Server (Producer)
       │
       │ ◄── "I'm full, slow down!" (Backpressure)
       ▼
Client (Consumer)
```

But standard raw WebSocket architectures don't automatically have great backpressure mechanics baked natively into your application code.

If a server reads endlessly from a lightning-fast Redis Pub/Sub stream and blindly tries to write every single broadcast into a slow WebSocket connection loop, it is systematically shoveling coal into a blocked furnace.

We have to handle this explicitly in our system design.

---

# 10.4 Mitigation 1: Bounded Buffers

The absolute simplest mechanism to protect a server's integrity is to apply a hard mathematical limit on every single connection buffer.

Instead of an infinitely growing array, we enforce a **Bounded Buffer**.

```text
Server RAM
┌─────────────────────────────────┐
│ Buffer for Client A (MAX: 10)   │
│ [1] [2] [3] [4] [5] [6] [7]     │
└─────────────────────────────────┘
```

If the buffer holds 10 messages, and the client is simply too slow to download them, the buffer instantly fills up: `[10/10]`.

When Message #11 arrives from Redis, the server cannot add it to the buffer. It has to make a hard architectural decision.

---

# 10.5 Mitigation 2: Dropping Messages (Lossy Strategy)

If the bounded buffer is completely full, the most computationally cheap strategy is simply to drop any new messages that arrive.

```text
Buffer: [1] [2] [3] [4] [ ... 10]

Msg 11 Arrives ──► DROP IT.
Msg 12 Arrives ──► DROP IT.
```

The client will miss critical data, but the Server's memory is perfectly protected, keeping the other 99,999 users safe.

This is highly common in live-streaming, analytics, or multiplayer gaming. If a user's internet lags violently, they don't necessarily need to see the exact coordinate history of an enemy's bullet from 10 seconds ago. You purposefully drop the old frames and serialize the newest state when their connection recovers.

---

# 10.6 Mitigation 3: Terminating Connections (Strict Strategy)

Dropping messages silently is incredibly dangerous if your application strictly guarantees delivery sequentially (like executing financial market trades or maintaining chat message history).

If the server drops a stock purchase confirmation, the client's UI will freeze in an inaccurate state indefinitely.

Instead of silently corrupting data, a robust server will simply kill the connection entirely.

```text
Buffer: [1] [2] [3] [4] [ ... 10]

Msg 11 Arrives ──► "Client is too slow. Terminate socket."

Server ──X   Client
```

The server forcefully kicks the struggling client offline. 

This causes the client to naturally trigger its automatic Exponential Backoff reconnect logic. When the client reconnects via a fresh, clean socket, it relies on the REST API to download the exact state history it missed at its own comfortable pace.

This rigorously protects the server from memory bloat without silently corrupting client UIs.

---

# 10.7 Mitigation 4: Latest-Value Semantics

If you are streaming continuous live telemetry (e.g., Bitcoin price updates), you don't necessarily need to queue every single micro-tick.

If the client is heavily lagging, they don't organically care that BTC traveled:
`$60,000 → $60,001 → $60,002 → $60,005`

They typically only care about the absolute most recent snapshot: `$60,005`.

Instead of utilizing a traditional list buffer, a server can utilize a **Latest-Value Buffer** (often a hash map or single variable).

```text
Price Buffer for Client A:
{
  "BTC": 60005
}
```

If 100 new BTC prices arrive rapidly from Redis while the client is stalling, the server just continuously overwrites the single variable in memory.

```text
60001 (Overwritten silently)
60002 (Overwritten silently)
60005 (Saved accurately)
```

When the client TCP window is finally clear and ready to accept bytes, the server flushes the one latest value across the wire.

Memory remains perfectly flat (O(1)), regardless of how terribly slow the client's network gets!

---

# 10.8 Downstream vs Upstream Backpressure

So far, we have only looked at protecting the WebSocket Server from a slow Client. This is called **Downstream Backpressure**.

But what about protecting the WebSocket Server from a blazing fast Redis Backplane? This is called **Upstream Backpressure**.

```text
Redis Pub/Sub ──► (1,000,000 msgs/sec) ──► WebSocket Server
```

If the WebSocket server can only parse, serialize, and route 10,000 messages per second, the Server itself will get rapidly crushed by the Redis pipeline.

This is fundamentally why we discussed **Message Queues vs Pub/Sub** deeply in Chapter 8.

Redis Pub/Sub does not strictly support upstream backpressure. It fiercely and blindly fires data at all local WebSocket servers. If the WebSocket server is slow, the host OS kernel buffer eventually hits capacity, and Redis simply starts dropping the broadcasts permanently.

If your system intends to generate 1,000,000 robust events a second, you might eventually have to replace simple Redis Pub/Sub with **Apache Kafka**, where the WebSocket server can strictly control how quickly it pulls messages from the persistent log to stay healthy.

---

# 10.9 Writing Go and Node.js Code for Backpressure

When you write the actual code for your WebSocket server, you must explicitly implement these protections. They do not happen automatically.

### In Go
You enforce this naturally using **Buffered Channels**:

```go
// Create a bounded channel holding exactly 256 messages.
client.send = make(chan []byte, 256)

// When a new message arrives from Redis Pub/Sub:
select {
case client.send <- message:
    // Successfully added message to the buffer
default:
    // Buffer is completely FULL (Backpressure successfully detected!)
    // Action: Aggressively disconnect client to save RAM
    client.Disconnect() 
}
```

### In Node.js
The native `ws` library relies tightly on underlying Node System Streams, where you have to monitor the `bufferedAmount` property continuously:

```javascript
// Before calling send...
if (socket.bufferedAmount > 1024 * 1024) { 
    // 1 Megabyte hard limit reached.
    // Client connection is choked. 
    socket.terminate();
} else {
    socket.send(data);
}
```

If you forget to check `bufferedAmount` in Node, `socket.send()` will happily eat your physical RAM until the V8 engine crashes permanently.

---

# 10.10 The End Goal

A production-ready distributed real-time system fundamentally respects Backpressure at every single layer of the stack.

```text
Data Source
     │  (Kafka limits delivery speed)
     ▼
WebSocket Server
     │  (Bounded Buffer explicitly checks bounds)
     ▼
TCP Connection
     │  (OS Kernel Windowing applies TCP pressure)
     ▼
Client Browser
```

If any isolated component securely in this chain slows down, the layers above it must either strategically pause, intentionally drop data, or aggressively sever connections to gracefully protect the overall integrity of the platform.

---

⬅️ **[Previous: 9. Reliability of Real-Time Connections](09_Real_Time_Reliability.md)** | 🏠 **[Back to TOC](README.md)** | **[Next: 11. Presence and Connection State ➡️](11_Presence_System.md)**
