# Chapter 4 — WebSocket Fundamentals

WebSockets provide a **persistent, bidirectional communication channel** between a client and a server.

Unlike SSE:

```text
SSE

Server ─────────────────► Client
```

WebSockets allow:

```text
WebSocket

Client ◄────────────────► Server
```

Both sides can send messages whenever they need to.

This makes WebSockets useful for systems where the client and server need to communicate continuously.

Examples:

- Chat applications
- Multiplayer games
- Collaborative editing
- Real-time trading interfaces
- Live customer support
- Real-time dashboards with client commands
- Presence systems
- Real-time notifications where bidirectional communication is useful

---

# 4.1 Why Do We Need WebSockets?

Consider a chat application.

With polling:

```text
Client ── "Any new messages?" ──► Server
Client ◄──────── "No" ─────────── Server

Client ── "Any new messages?" ──► Server
Client ◄──────── "No" ─────────── Server

Client ── "Any new messages?" ──► Server
Client ◄─────── "Yes!" ────────── Server
```

This is inefficient for continuous communication.

With SSE:

```text
Server ─────────► Client
```

The server can push messages, but the client still needs another mechanism to send messages.

With WebSockets:

```text
Client ─────────────► Server
Client ◄───────────── Server
Client ─────────────► Server
Client ◄───────────── Server
```

Both sides can communicate over the same persistent connection.

---

# 4.2 The Core Idea

A normal HTTP interaction is generally:

```text
Request
   │
   ▼
Server
   │
   ▼
Response
```

WebSockets change the communication model.

After establishing the connection:

```text
Client ◄────────────────────────► Server
          persistent connection

Client ── message ──────────────► Server
Client ◄──────────── message ───── Server
Client ── message ──────────────► Server
Client ◄──────────── message ───── Server
```

There is no need for a new HTTP request for every WebSocket message.

---

# 4.3 Full-Duplex Communication

The technical term you'll hear frequently is:

> **Full-duplex communication**

It means both sides can send data independently.

For example:

```text
Client ────────► Server
       message

Client ◄──────── Server
       message
```

And these can happen independently:

```text
Client ───────────────► Server
           │
           │
           │
Server ───────────────► Client
```

The server does not have to wait for a client request before sending a message.

Likewise, the client doesn't have to wait for the server to send something.

---

# 4.4 WebSockets Still Start With HTTP

This often causes confusion.

A WebSocket connection begins with an HTTP request.

The client essentially asks:

> "Can we upgrade this HTTP connection to WebSocket?"

Conceptually:

```text
Client
   │
   │ HTTP Upgrade Request
   ▼
Server
   │
   │ 101 Switching Protocols
   ▼
WebSocket connection
```

After the upgrade succeeds, communication switches to the WebSocket protocol.

---

# 4.5 The Upgrade Handshake

A simplified request looks like:

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: <random-value>
Sec-WebSocket-Version: 13
```

The server responds with something like:

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: <value>
```

The important part is:

```http
101 Switching Protocols
```

It means:

> The server accepted the protocol switch.

After this point, the connection is no longer being used as an ordinary HTTP request/response exchange.

---

# 4.6 Why Use HTTP for the Handshake?

HTTP is already understood by:

- browsers
- proxies
- load balancers
- firewalls
- servers
- infrastructure

Using HTTP for connection establishment makes it easier for WebSockets to fit into the existing web ecosystem.

But after the upgrade:

```text
HTTP
  │
  │ Upgrade
  ▼
WebSocket protocol
```

---

# 4.7 The WebSocket Connection Lifecycle

A useful mental model is:

```text
             Client
                │
                │ HTTP Upgrade
                ▼
             Server
                │
                │ 101
                ▼
        WebSocket Connected
                │
        ┌───────┴────────┐
        │                │
        ▼                ▼
     Messages         Messages
        │                │
        └───────┬────────┘
                │
                ▼
             Closing
                │
                ▼
            Closed
```

So a WebSocket has a lifecycle:

1. Connecting
2. Handshake
3. Open
4. Message exchange
5. Closing
6. Closed

---

# 4.8 WebSocket Messages

Once connected, the application can send messages.

For example:

```text
Client → Server

{
  "type": "message",
  "text": "Hello"
}
```

The server might respond:

```text
Server → Client

{
  "type": "message",
  "text": "Hello back"
}
```

WebSocket itself doesn't force your application to use JSON.

You can send:

- Text
- Binary data

JSON is simply a common application-level format.

---

# 4.9 WebSocket Is Not Your Application Protocol

This distinction is important.

WebSocket defines how communication happens.

Your application defines what the messages mean.

For example:

```text
WebSocket
    │
    ▼
Application messages
    │
    ├── chat.message
    ├── user.typing
    ├── room.join
    ├── room.leave
    └── notification
```

WebSocket doesn't inherently understand:

```text
room.join
```

Your application does.

---

# 4.10 Example: Chat

Suppose Alice opens a chat.

```text
Alice Browser
      │
      │ WebSocket
      ▼
   Server
```

Alice sends:

```json
{
  "type": "message",
  "text": "Hello"
}
```

The server processes it and sends it to Bob:

```text
Alice
  │
  │ message
  ▼
Server
  │
  │ message
  ▼
Bob
```

The important part is that the server can send the message to Bob **without Bob having to poll for it**.

---

# 4.11 Example: Typing Indicators

Suppose Alice starts typing.

Her browser sends:

```json
{
  "type": "typing",
  "userId": "alice"
}
```

The server can immediately send:

```text
Server ─────────► Bob
                  Alice is typing
```

When she stops:

```json
{
  "type": "typing",
  "userId": "alice",
  "typing": false
}
```

This is a good WebSocket use case because updates happen frequently in both directions.

---

# 4.12 WebSocket vs SSE

This is the comparison you should remember.

| Feature               | SSE                    | WebSocket                   |
| --------------------- | ---------------------- | --------------------------- |
| Persistent connection | Yes                    | Yes                         |
| Based around HTTP     | Yes                    | Starts with HTTP upgrade    |
| Server → Client       | Yes                    | Yes                         |
| Client → Server       | Separate HTTP requests | Same connection             |
| Bidirectional         | No                     | Yes                         |
| Browser API           | `EventSource`          | `WebSocket`                 |
| Binary data           | Not its primary model  | Supported                   |
| Automatic reconnect   | Basic browser support  | Usually application/library |
| Complexity            | Lower                  | Higher                      |

The fundamental distinction:

```text
SSE:

Server ─────────► Client


WebSocket:

Server ◄────────► Client
```

---

# 4.13 When Should You Choose WebSockets?

WebSockets are useful when communication is:

### Frequent

Messages may be exchanged continuously.

### Bidirectional

Both sides need to communicate.

### Low-latency

You don't want to repeatedly create HTTP requests.

### Connection-oriented

The application benefits from maintaining a live connection.

Examples:

```text
Chat
Multiplayer games
Collaborative applications
Real-time control interfaces
Live trading interfaces
Presence
```

---

# 4.14 When You Don't Need WebSockets

Don't automatically choose WebSockets simply because something is called "real-time."

For example:

> "Show me when my background job finishes."

You might only need:

```text
Server ─────► Client
```

SSE can be enough.

Or perhaps polling is completely acceptable.

The general principle is:

> **Choose the simplest communication mechanism that satisfies the requirements.**

---

# 4.15 WebSockets and HTTP APIs Can Coexist

A real application doesn't have to choose one technology for everything.

For example:

```text
                 Application
                      │
          ┌───────────┴───────────┐
          │                       │
       REST API               WebSocket
          │                       │
          ▼                       ▼
   Normal operations       Real-time events
```

You might use HTTP for:

```text
POST /messages
GET /profile
GET /rooms
```

and WebSockets for:

```text
message received
typing
presence
live updates
```

The two mechanisms can coexist.

---

# 4.16 WebSockets Behind a Load Balancer

A simple architecture might look like:

```text
                Client
                   │
                   ▼
             Load Balancer
              /          \
             ▼            ▼
       WebSocket A    WebSocket B
```

The important difference from normal short-lived HTTP requests is that a WebSocket connection stays attached to a server.

For example:

```text
Client
  │
  │ connection
  ▼
Server A
```

That connection remains on Server A.

The load balancer doesn't move the already-established connection to Server B for every message.

---

# 4.17 Why Multiple WebSocket Servers Create a Problem

Imagine:

```text
Client A ───► Server A

Client B ───► Server B
```

Now Client A sends a message:

```text
Client A
   │
   │ "Hello"
   ▼
Server A
```

But Client B is connected to:

```text
Server B
```

How does Server A tell Server B?

A simple architecture can use a shared Pub/Sub layer:

```text
Client A
   │
   ▼
Server A
   │
   ▼
Redis Pub/Sub
   │
   ▼
Server B
   │
   ▼
Client B
```

This is called a **backplane**.

We'll study this architecture in detail later.

---

# 4.18 Sticky Sessions

Another concept you'll encounter is:

> **Sticky sessions / session affinity**

A load balancer can try to keep a client's connections associated with the same backend server.

Conceptually:

```text
Client A ──► Server A
Client A ──► Server A
Client A ──► Server A
```

instead of:

```text
Client A ──► Server A
Client A ──► Server B
Client A ──► Server C
```

For WebSockets, once the connection is established, the connection itself naturally remains on that server.

Sticky sessions can still matter for related connection/session behavior, but they don't solve the broader problem of distributing events between servers.

That's why a Pub/Sub backplane can still be necessary.

---

# 4.19 WebSocket Connection State

A WebSocket server needs to track connected clients.

Conceptually:

```text
Server
 │
 ├── Connection A
 ├── Connection B
 ├── Connection C
 └── Connection D
```

Each connection may contain:

- socket information
- authentication information
- room membership
- connection state
- buffers
- subscriptions

This consumes resources.

Therefore, WebSocket scaling is not simply:

> "How many HTTP requests can my server process?"

You also need to ask:

> "How many persistent connections can my server maintain?"

---

# 4.20 Connection Memory

Suppose a server maintains:

```text
100,000 connections
```

Even if each connection uses a relatively small amount of memory, the total can become significant.

For example, conceptually:

```text
Connections
     │
     ▼
Per-connection memory
     │
     ▼
Total memory
```

And memory isn't the only limit.

You also need to consider:

- File descriptors
- CPU
- Network bandwidth
- Kernel socket buffers
- Application buffers
- Message frequency

Therefore, persistent connections need capacity planning.

---

# 4.21 What Happens When a Server Dies?

Suppose:

```text
Client
  │
  ▼
Server A
```

Server A crashes.

The connection disappears:

```text
Client
  │
  X
Server A
```

The client must reconnect.

A production WebSocket application therefore needs a reconnection strategy.

A common flow is:

```text
Connected
    │
    │ failure
    ▼
Disconnected
    │
    ▼
Reconnect
    │
    ▼
Authenticate
    │
    ▼
Restore subscriptions/state
    │
    ▼
Connected
```

The exact strategy depends on the application.

---

# 4.22 Reconnection Storms

Imagine:

```text
WebSocket server crashes
          │
          ▼
50,000 clients disconnect
          │
          ▼
50,000 clients reconnect
```

If they all reconnect simultaneously:

```text
           Load
             │
             │       ███████████
             │       ███████████
             │       ███████████
             │_______███████████________
                     Time
```

The reconnect traffic itself can overload the system.

This is called a **thundering herd / reconnect storm** problem.

Later we'll discuss strategies such as:

- exponential backoff
- jitter
- connection limits
- graceful draining

---

# 4.23 Heartbeats

A WebSocket connection may appear alive even when the underlying network path has problems.

WebSockets therefore support control messages such as:

```text
Ping
Pong
```

Conceptually:

```text
Server ───── Ping ─────► Client
Server ◄──── Pong ────── Client
```

This helps detect broken or unresponsive connections.

The detailed frame-level mechanics will be covered in the WebSocket protocol deep dive.

For now, remember:

> **Heartbeats help determine whether a long-lived connection is still healthy.**

---

# 4.24 Graceful Shutdown

Suppose a WebSocket server is being deployed.

You don't want to abruptly terminate thousands of connections.

A better approach is:

```text
Stop accepting new connections
          │
          ▼
Drain existing connections
          │
          ▼
Clients reconnect elsewhere
          │
          ▼
Shutdown
```

This becomes particularly important in rolling deployments.

---

# 4.25 WebSocket Does Not Guarantee Reliable Delivery

A very important misconception:

> WebSocket being connection-oriented does not mean your application automatically has durable message delivery.

Suppose:

```text
Server ─── message ───► Client
```

and immediately afterward the connection breaks.

What happens if the client never processed the message?

WebSocket itself does not provide a durable message history like a message broker.

If your application needs recovery, you may need:

```text
WebSocket
    +
Message IDs
    +
Persistent storage / message system
```

For example:

```text
Client reconnects
       │
       ▼
"Last message I received = 104"
       │
       ▼
Server
       │
       ▼
Send missing messages
```

This is an **application-level reliability mechanism**.

---

# 4.26 Ordering

Messages sent over one WebSocket connection are delivered according to the WebSocket/TCP transport ordering characteristics.

But distributed applications introduce additional complexity.

For example:

```text
Client A
   │
   ▼
Server A ──► Pub/Sub ──► Server B
```

Now your application may have:

- multiple producers
- multiple servers
- multiple channels
- reconnects

Therefore, "WebSocket preserves ordering" should not be interpreted as:

> "My entire distributed system automatically has global ordering."

Transport ordering and application-level ordering are different concerns.

---

# 4.27 WebSocket Security

WebSockets should also be secured.

For browser applications, production connections generally use:

```text
wss://
```

instead of:

```text
ws://
```

Conceptually:

```text
ws://
  │
  └── WebSocket without TLS


wss://
  │
  └── WebSocket over TLS
```

You also need to consider:

- Authentication
- Authorization
- Origin validation
- Input validation
- Rate limiting
- Connection limits

A WebSocket connection can stay open for a long time, so authentication and authorization decisions need to be handled carefully.

---

# 4.28 WebSockets and Rooms

Many applications need logical groups.

For example:

```text
Room: cricket-match-123
```

Clients can join:

```text
Client A ──┐
Client B ──┼──► Room 123
Client C ──┘
```

When an event occurs:

```text
Server
   │
   ▼
Room 123
   │
   ├──► Client A
   ├──► Client B
   └──► Client C
```

Rooms are not part of the core WebSocket protocol.

They're an **application/library-level abstraction**.

This distinction becomes important when comparing libraries such as Socket.IO with lower-level WebSocket implementations.

---

# 4.29 Raw WebSocket vs Higher-Level Libraries

At the protocol level, you can work directly with WebSockets.

For example, in Node.js, a common library is:

```text
ws
```

It gives relatively direct access to WebSocket behavior.

Higher-level libraries such as:

```text
Socket.IO
```

provide additional abstractions such as:

- rooms
- namespaces
- reconnection
- event-oriented APIs
- additional connection behavior

The trade-off is that you're no longer working with only the basic WebSocket protocol.

We'll compare these libraries later.

---

# 4.30 WebSocket vs Socket.IO

Do not think:

```text
WebSocket = Socket.IO
```

They are different.

### WebSocket

A protocol/standard communication mechanism.

### Socket.IO

A higher-level real-time framework/library with its own protocol and features.

Conceptually:

```text
Application
     │
     ▼
Socket.IO
     │
     ▼
Transport layer
```

Socket.IO may use WebSocket when available, but it is not simply "WebSocket with a nicer API."

We'll cover the distinction properly in the library chapter.

---

# 4.31 A Basic WebSocket Architecture

A small application:

```text
┌──────────────┐
│    Client    │
└──────┬───────┘
       │
       │ WebSocket
       ▼
┌──────────────┐
│ WebSocket    │
│ Server       │
└──────┬───────┘
       │
       ├──────────────► Database
       │
       └──────────────► Application Logic
```

For example, a chat message:

```text
Client
  │
  │ send message
  ▼
WebSocket Server
  │
  ├── validate
  ├── authenticate
  ├── persist
  └── broadcast
```

---

# 4.32 A Scaled WebSocket Architecture

When we have multiple servers:

```text
                         Clients
                      /     |     \
                     /      |      \
                    ▼       ▼       ▼
              ┌─────────────────────────┐
              │      Load Balancer      │
              └────────────┬────────────┘
                           │
                ┌──────────┼──────────┐
                ▼          ▼          ▼
             WS Server  WS Server  WS Server
                │          │          │
                └──────────┼──────────┘
                           ▼
                      Pub/Sub
                           │
                           ▼
                       Database
```

The Pub/Sub layer allows servers to exchange events.

For example:

```text
Client A
   │
   ▼
WS Server 1
   │
   ▼
Pub/Sub
   │
   ├────► WS Server 2 ───► Client B
   │
   └────► WS Server 3 ───► Client C
```

This is a foundational architecture pattern for large real-time systems.

---

# 4.33 The Most Important Scaling Insight

With ordinary HTTP:

```text
Request
   │
   ▼
Server
   │
   ▼
Response
```

The request is short-lived.

With WebSockets:

```text
Connection
   │
   ├── message
   ├── message
   ├── message
   ├── message
   └── ...
```

The server must maintain the connection for potentially a long time.

Therefore:

> **WebSocket scaling is heavily influenced by concurrent connections, not just requests per second.**

You still care about message throughput, but concurrent connection count becomes a major capacity dimension.

---

# 4.34 What WebSocket Solves

WebSockets solve a specific communication problem:

> **How can a client and server maintain a persistent, low-latency, bidirectional communication channel?**

The answer:

```text
HTTP Upgrade
      │
      ▼
WebSocket connection
      │
      ▼
Bidirectional messages
```

---

# 4.35 What WebSocket Does NOT Solve

WebSocket does not automatically solve:

- Distributed fan-out
- Message durability
- Offline clients
- Message replay
- Global ordering
- Authentication
- Authorization
- Scaling across many servers
- Backpressure
- Reconnect storms

These are **system-design problems around the WebSocket connection**.

This distinction is critical.

---

# 4.36 Final Mental Model

Remember WebSockets as:

```text
                    HTTP
                     │
                     │ Upgrade
                     ▼
              WebSocket Connection
                     │
                     ▼
          ┌──────────────────────┐
          │   Persistent Channel │
          └──────────────────────┘
               ▲            │
               │            │
               │            │
          Client           Server
               │            │
               └────────────┘
                 bidirectional
```

Or simply:

```text
SSE:

Server ─────────────────► Client


WebSocket:

Client ◄────────────────► Server
```

And at system level:

```text
                    Load Balancer
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
            WS 1       WS 2       WS 3
              │          │          │
              └──────────┼──────────┘
                         ▼
                      Pub/Sub
                         │
                         ▼
                      Database
```

The WebSocket itself is only the **communication channel**.

The rest of the architecture exists to make that channel useful, scalable, reliable, and secure.

---

# 4.37 Key Takeaways

1. **WebSocket provides persistent bidirectional communication.**

```text
Client ◄────────► Server
```

2. A WebSocket connection **starts with an HTTP Upgrade handshake**.

3. `101 Switching Protocols` indicates that the upgrade was accepted.

4. After the handshake, communication uses the **WebSocket protocol**, not ordinary HTTP request/response semantics.

5. WebSockets are useful when both client and server need frequent, low-latency communication.

6. WebSockets are not automatically better than SSE or polling.

7. A WebSocket connection consumes server resources for as long as it remains open.

8. Multiple WebSocket servers often need a **shared Pub/Sub backplane** for cross-server event distribution.

9. Reconnection, heartbeats, graceful shutdown, backpressure, reliability, and message recovery are separate system-design concerns.

10. **WebSocket is the transport/channel—not the complete real-time architecture.**

---

# 4.38 What's Next?

Now that we understand what WebSockets are and why they exist, the next chapter goes **one level deeper into the actual protocol**.

We'll study:

- RFC 6455
- HTTP Upgrade handshake
- `Sec-WebSocket-Key`
- `Sec-WebSocket-Accept`
- WebSocket frames
- Frame structure
- Opcodes
- Text vs binary frames
- Masking
- Ping/Pong
- Close frames
- Fragmentation
- Message boundaries

That chapter is where we stop treating WebSocket as a black box and understand **what is actually travelling over the network**.

---

⬅️ **[Back: Server-Sent Events (SSE)](03_Server_Sent_Events.md)**

➡️ **[Next: WebSocket Protocol Deep Dive](05_WebSocket_Protocol.md)**

⬆️ **[Back to Real-Time Backend](README.md)**
