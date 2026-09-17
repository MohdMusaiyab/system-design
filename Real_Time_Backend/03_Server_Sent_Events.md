# Chapter 3 — Server-Sent Events (SSE)

Server-Sent Events (SSE) is a mechanism that allows a server to continuously send events to a client over a **long-lived HTTP connection**.

The key idea is simple:

> **The client opens one HTTP connection, keeps it open, and the server can send multiple events through that connection over time.**

This makes SSE useful for applications where communication primarily flows in one direction:

```text
Server ───────────────────► Client
```

Examples include:

* Live notifications
* Live dashboards
* Progress updates
* Activity feeds
* Live sports scores
* Monitoring systems
* AI response streaming
* Background-job status
* Real-time counters

SSE sits between long polling and WebSockets in terms of communication capabilities.

---

# 3.1 The Problem SSE Solves

We saw two approaches in the previous chapter.

### Short Polling

```text
Client ── Request ──► Server
Client ◄─ Response ── Server

        wait

Client ── Request ──► Server
Client ◄─ Response ── Server
```

The client repeatedly asks:

> "Anything new?"

---

### Long Polling

```text
Client ── Request ─────────► Server
                            │
                            │ wait...
                            │
                            │ event
                            ▼
Client ◄──── Response ────── Server

Client ── Request ─────────► Server
```

This is better because the server can wait for an event.

But the client still has to repeatedly create requests.

SSE takes another step:

```text
Client ── Request ───────────────► Server
                                  │
                                  │ connection stays open
                                  │
Client ◄──── Event 1 ─────────────┤
Client ◄──── Event 2 ─────────────┤
Client ◄──── Event 3 ─────────────┤
Client ◄──── Event 4 ─────────────┤
                                  │
                                  │
                              connection
                                 closes
```

One request can support **many server-to-client events**.

---

# 3.2 What Exactly Is SSE?

SSE stands for:

> **Server-Sent Events**

It is a web technology built on top of HTTP that allows a server to send a stream of events to a client.

The communication model is:

```text
Client ───────────────► Server
       initial request

Server ───────────────► Client
       event stream
```

The important characteristic is:

> **SSE is primarily one-way: server → client.**

The client can still make normal HTTP requests separately.

For example:

```text
                 HTTP API
Client ─────────────────────────► Server
Client ◄───────────────────────── Server


                 SSE
Client ─────────────────────────► Server
       connection establishment

Client ◄───────────────────────── Server
Client ◄───────────────────────── Server
Client ◄───────────────────────── Server
```

---

# 3.3 SSE Is Still HTTP

This is one of the most important things to remember.

SSE does **not** introduce a completely separate transport protocol like WebSockets.

It uses an HTTP connection.

The client essentially says:

```http
GET /events HTTP/1.1
Host: example.com
Accept: text/event-stream
```

The server responds with:

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

Then the response body remains open.

The server can continue writing events into it.

---

# 3.4 The Mental Model

Think of an SSE connection as a pipe.

```text
                 SSE Connection

Server
  │
  │ event
  ▼
══════════════════════════════════════► Client
  │
  │ event
  ▼
══════════════════════════════════════► Client
  │
  │ event
  ▼
══════════════════════════════════════► Client
```

The pipe remains open.

The server doesn't create a new HTTP request for every event.

---

# 3.5 Basic SSE Architecture

A simple system looks like:

```text
┌──────────────┐
│   Browser    │
└──────┬───────┘
       │
       │ HTTP connection
       │
       ▼
┌──────────────┐
│    Server    │
└──────┬───────┘
       │
       ▼
   Event Source
```

The event source could be:

* Database changes
* Redis
* Message queue
* Background worker
* Application logic
* External event stream

For example:

```text
User action
    │
    ▼
Application
    │
    ▼
Event
    │
    ▼
SSE connection
    │
    ▼
Browser
```

---

# 3.6 A Simple Example

Imagine a background job:

```text
Generate large report
```

The client wants progress updates.

Instead of polling:

```text
GET /jobs/123
GET /jobs/123
GET /jobs/123
GET /jobs/123
```

we can establish:

```text
GET /jobs/123/events
```

Then the server can send:

```text
progress: 10
```

then:

```text
progress: 30
```

then:

```text
progress: 60
```

then:

```text
progress: 100
```

The browser receives the updates as they happen.

---

# 3.7 The SSE Event Format

SSE uses a simple text-based event format.

A basic event can look like:

```text
data: Hello World

```

Notice the blank line after the event.

That blank line is important because it marks the end of the event.

Multiple events:

```text
data: First event

data: Second event

data: Third event

```

Conceptually:

```text
┌──────────────────┐
│ data: Event 1    │
│                  │
├──────────────────┤
│ data: Event 2    │
│                  │
├──────────────────┤
│ data: Event 3    │
│                  │
└──────────────────┘
```

---

# 3.8 Why `text/event-stream`?

The server tells the client:

```http
Content-Type: text/event-stream
```

This means:

> "The response body is an SSE event stream."

The browser can then interpret incoming data as SSE events.

This is different from a normal JSON response:

```http
Content-Type: application/json
```

where the client expects a complete JSON response.

With SSE, the response is intentionally kept open.

---

# 3.9 `EventSource`

Browsers provide a built-in API called:

```javascript
EventSource
```

A client can do:

```javascript
const events = new EventSource("/events");

events.onmessage = (event) => {
  console.log(event.data);
};
```

The browser:

1. opens the HTTP connection
2. keeps it open
3. receives events
4. invokes the appropriate handlers

Conceptually:

```text
Browser
   │
   │ new EventSource()
   ▼
GET /events
   │
   ▼
Server
   │
   │ persistent stream
   ▼
EventSource
   │
   ▼
JavaScript handler
```

---

# 3.10 Why Is EventSource Useful?

Without SSE, the client would have to manually manage:

```text
Request
Wait
Request
Wait
Request
Wait
```

With `EventSource`:

```text
EventSource
     │
     ▼
Persistent connection
     │
     ├── event
     ├── event
     ├── event
     └── event
```

The browser handles much of the connection management for you.

---

# 3.11 Named Events

SSE supports event types.

For example:

```text
event: notification
data: {"message":"Someone liked your post"}

```

And:

```text
event: message
data: {"from":"Alice","text":"Hello"}

```

The client can listen for specific events:

```javascript
events.addEventListener("notification", (event) => {
  console.log("Notification:", event.data);
});

events.addEventListener("message", (event) => {
  console.log("Message:", event.data);
});
```

This allows the same SSE stream to carry different event types.

---

# 3.12 Event Data

The `data:` field contains the payload.

It can contain simple text:

```text
data: Hello
```

or JSON:

```text
data: {"userId":123,"status":"online"}
```

SSE itself doesn't require the payload to be JSON.

JSON is simply a common application-level choice.

This distinction is useful:

> **SSE defines how events are streamed; your application decides what the event data means.**

---

# 3.13 Multi-Line Data

SSE can also contain multiple `data:` lines.

For example:

```text
data: line one
data: line two
data: line three

```

These are treated as part of the same event.

You don't need to memorize the exact parsing rules right now.

The important thing is that SSE is a **text-based event stream**, not simply "a sequence of JSON responses."

---

# 3.14 Event IDs

SSE supports an optional:

```text
id:
```

field.

Example:

```text
id: 101
data: New message

```

Then:

```text
id: 102
data: Another message

```

The ID allows the client and server to reason about where the client was in the event stream.

This becomes especially useful during reconnection.

---

# 3.15 Why Do We Need Event IDs?

Imagine:

```text
Event 101
Event 102
Event 103
Event 104
```

The client receives:

```text
101
102
```

Then the network breaks.

```text
Client
   │
   │ received 101, 102
   X
connection lost
```

What about:

```text
103
104
```

?

If the system supports recovery, the client can tell the server:

> "I last received event 102."

The server may then resume from:

```text
103
```

This is the basic idea behind resumable event streams.

But:

> **SSE event IDs alone do not magically provide message durability.**

The server needs access to the events or some recoverable state.

---

# 3.16 `Last-Event-ID`

When reconnecting, the client can communicate the last event it received.

Conceptually:

```text
Last received:
102
```

Then reconnect:

```http
Last-Event-ID: 102
```

The server can potentially send:

```text
103
104
105
```

This provides a mechanism for catching up.

Again, this requires the server to retain or otherwise reconstruct those events.

---

# 3.17 SSE Automatic Reconnection

One of the nice features of browser `EventSource` is automatic reconnection.

Conceptually:

```text
Connected
    │
    ▼
Receiving events
    │
    │ network failure
    ▼
Disconnected
    │
    │ browser retries
    ▼
Connecting
    │
    ▼
Connected
```

The browser handles the basic retry behavior.

The server can also provide a retry suggestion using:

```text
retry: 5000
```

meaning approximately:

> Try reconnecting after 5000 milliseconds.

---

# 3.18 Reconnection Is Not the Same as Recovery

This distinction is extremely important.

### Reconnection

Means:

> Establish the connection again.

### Recovery

Means:

> Determine which events the client missed and deliver them if required.

For example:

```text
Connection:
       101
       102
       X
       disconnected
       │
       ▼
reconnect
       │
       ▼
       103
       104
```

Reconnection alone only gives you a new connection.

Recovery requires additional application-level support.

---

# 3.19 Heartbeats

What happens if no real events are generated for a long time?

A proxy, load balancer, or network device may consider the connection idle.

One common solution is sending a small heartbeat.

SSE supports comment lines:

```text
: heartbeat

```

These can be used to keep the connection active.

Conceptually:

```text
Server ───── event ─────► Client

          silence

Server ─── heartbeat ────► Client

          silence

Server ───── event ─────► Client
```

The exact heartbeat strategy depends on the infrastructure.

---

# 3.20 Why Heartbeats Matter

Suppose:

```text
Client ─────────────── Server
       SSE connection
```

No events happen for 10 minutes.

A network intermediary might have an idle timeout of:

```text
5 minutes
```

The connection could be terminated even though both application endpoints are healthy.

Heartbeats can prevent the connection from appearing idle.

This gives us a general production lesson:

> **A long-lived connection must account for idle timeouts throughout the network path.**

---

# 3.21 SSE and HTTP Streaming

SSE is essentially a specialized form of HTTP streaming.

The server starts an HTTP response:

```text
HTTP Response
```

and instead of immediately sending the entire body:

```text
[complete response]
```

it gradually sends chunks:

```text
chunk
chunk
chunk
chunk
...
```

The client interprets these chunks according to the SSE format.

Conceptually:

```text
HTTP Response
────────────────────────────────────►

Headers
   │
   ├── Event 1
   ├── Event 2
   ├── Event 3
   ├── Event 4
   └── ...
```

---

# 3.22 SSE vs Normal HTTP Response

Normal HTTP:

```text
Request
   │
   ▼
Server processes
   │
   ▼
Complete response
   │
   ▼
Connection/request finishes
```

SSE:

```text
Request
   │
   ▼
Server starts response
   │
   ├── Event
   ├── Event
   ├── Event
   ├── Event
   │
   │
   ▼
Connection eventually closes
```

The response body is streamed over time.

---

# 3.23 SSE Is One-Way

This is perhaps the biggest limitation.

SSE is designed for:

```text
Server ─────────► Client
```

If the client wants to send something to the server, it normally uses another HTTP request.

For example:

```text
                   SSE
Server ─────────────────────────► Client
        notifications


                   HTTP POST
Client ─────────────────────────► Server
        send message
```

This is completely valid.

The application can use:

* SSE for receiving events
* normal HTTP APIs for sending commands

---

# 3.24 Why Not Just Use WebSockets Everywhere?

Because WebSockets provide capabilities you may not need.

Suppose the requirement is:

> "The server should stream AI-generated text to the browser."

The client doesn't necessarily need to continuously send messages over the same connection.

The communication might simply be:

```text
Server ───── tokens ─────► Client
Server ───── tokens ─────► Client
Server ───── tokens ─────► Client
```

SSE can be a very natural fit.

Using WebSockets would introduce additional protocol and connection-management considerations without necessarily solving a problem you have.

---

# 3.25 SSE vs Long Polling

Let's compare them.

### Long Polling

```text
Request
   ↓
Wait
   ↓
Response
   ↓
Request again
   ↓
Wait
   ↓
Response
```

### SSE

```text
Request
   ↓
Persistent response
   ├── Event
   ├── Event
   ├── Event
   └── Event
```

The biggest difference is:

> **SSE allows multiple events to flow through the same HTTP response.**

---

# 3.26 Comparison

| Property                       | Long Polling                         | SSE                            |
| ------------------------------ | ------------------------------------ | ------------------------------ |
| Based on HTTP                  | Yes                                  | Yes                            |
| Client request                 | Repeated                             | Usually one long-lived request |
| Server → Client                | Yes                                  | Yes                            |
| Multiple events per connection | No, generally one response at a time | Yes                            |
| Persistent stream              | No                                   | Yes                            |
| Automatic browser reconnect    | Application-managed                  | Built into `EventSource`       |
| Event IDs                      | Application-specific                 | Built-in mechanism             |
| Bidirectional                  | No                                   | No                             |
| Text event format              | No                                   | Yes                            |
| Complexity                     | Moderate                             | Low/Moderate                   |

---

# 3.27 SSE vs WebSockets

This is one of the most important comparisons.

| Property             | SSE                                       | WebSocket                                         |
| -------------------- | ----------------------------------------- | ------------------------------------------------- |
| Transport            | HTTP                                      | Starts with HTTP Upgrade, then WebSocket protocol |
| Communication        | Server → Client                           | Client ↔ Server                                   |
| Persistent           | Yes                                       | Yes                                               |
| Built-in browser API | `EventSource`                             | `WebSocket`                                       |
| Data model           | Event stream                              | Frames/messages                                   |
| Automatic reconnect  | `EventSource` provides basic reconnection | Usually application/library-managed               |
| Binary data          | Not its primary model                     | Yes                                               |
| Bidirectional        | No                                        | Yes                                               |
| Complexity           | Lower                                     | Higher                                            |
| Good for             | Server updates                            | Interactive two-way systems                       |

The decision should come from requirements.

---

# 3.28 SSE for Notifications

A classic architecture:

```text
                       Notification Event
                              │
                              ▼
                         Application
                              │
                              ▼
                         SSE Handler
                              │
                              ▼
                            Client
```

For example:

```text
event: notification
data: {"type":"LIKE","postId":123}

```

The browser receives it and updates the notification icon.

---

# 3.29 SSE for Live Dashboards

Suppose a monitoring dashboard needs updates:

```text
CPU: 61%
Memory: 72%
Requests: 1240/sec
```

The server can continuously send:

```text
event: metrics
data: {"cpu":61,"memory":72,"rps":1240}

```

then:

```text
event: metrics
data: {"cpu":64,"memory":70,"rps":1310}

```

The browser updates the dashboard.

No repeated polling is required.

---

# 3.30 SSE for Progress Updates

Consider:

```text
Upload
   │
   ▼
Processing
```

The backend can send:

```text
data: {"progress":10}

data: {"progress":30}

data: {"progress":50}

data: {"progress":90}

data: {"progress":100}

```

The UI can display:

```text
████████████████████ 100%
```

This is a common and simple SSE use case.

---

# 3.31 SSE for AI Streaming

Modern AI applications often generate output incrementally.

Instead of waiting:

```text
Request
    │
    │
    │ generate entire answer
    │
    ▼
Complete response
```

the server can stream pieces:

```text
Request
   │
   ▼
Server
   │
   ├── "System"
   ├── " design"
   ├── " is"
   ├── " the"
   ├── " process"
   └── ...
```

The client renders the response incrementally.

This creates a much better perceived response time.

The exact transport may vary—SSE is one common option.

---

# 3.32 SSE and Proxies

There is an important production issue.

Your application may be:

```text
Client
   │
   ▼
CDN / Proxy
   │
   ▼
Load Balancer
   │
   ▼
Application
```

Even though your application keeps the response open, an intermediary may:

* buffer the response
* impose an idle timeout
* impose a maximum connection duration
* close the connection
* interfere with streaming behavior

Therefore:

> **SSE is not just an application-code problem. The entire HTTP infrastructure needs to support streaming correctly.**

---

# 3.33 Buffering

Suppose your server generates:

```text
Event 1
Event 2
Event 3
```

but a proxy buffers the response.

Instead of:

```text
Event 1 → client
Event 2 → client
Event 3 → client
```

the client might receive:

```text
Event 1
Event 2
Event 3
```

all at once.

That defeats the purpose of streaming.

Therefore, production SSE systems may need appropriate proxy/CDN configuration to ensure data is flushed rather than unnecessarily buffered.

---

# 3.34 Flushing

When an application writes an SSE event, it needs to make sure the data is actually sent through the relevant layers rather than sitting in an application/framework buffer.

Conceptually:

```text
Application writes event
        │
        ▼
Application buffer
        │
        ▼
HTTP server
        │
        ▼
Proxy
        │
        ▼
Network
        │
        ▼
Client
```

Streaming only works as expected when the event progresses through the pipeline promptly.

The exact flushing behavior depends on the server framework and infrastructure.

---

# 3.35 SSE and Load Balancing

Suppose:

```text
                Load Balancer
                /           \
               ▼             ▼
           Server A       Server B
```

Client A connects:

```text
Client A ─────► Server A
```

The SSE connection remains open on Server A.

Now an event is generated.

Where does it go?

```text
Event
  │
  ▼
???
```

The application needs a way for Server A to learn about the event.

With multiple servers, this can lead to:

```text
                    Event Source
                         │
                         ▼
                      Pub/Sub
                     /       \
                    ▼         ▼
                Server A   Server B
                    │         │
                    ▼         ▼
                 Clients   Clients
```

This is the same distributed problem we'll encounter with WebSockets.

---

# 3.36 Why Redis Pub/Sub Can Appear

Imagine:

```text
Client A ─── SSE ───► Server A

Client B ─── SSE ───► Server B
```

An event is created:

```text
New notification for Client A
```

The application publishes:

```text
notifications:user:123
```

to a shared Pub/Sub system.

Server A receives it:

```text
Pub/Sub
   │
   ▼
Server A
   │
   ▼
SSE connection
   │
   ▼
Client A
```

Server B doesn't necessarily need to send it to Client A because Client A's connection belongs to Server A.

This architecture will become much more important later.

---

# 3.37 SSE and Stateful Connections

Even though SSE uses HTTP, the connection itself is long-lived.

Therefore:

```text
Client
   │
   │ SSE
   ▼
Server A
```

Server A maintains that connection.

This means SSE can have similar scaling considerations to WebSockets:

* concurrent connections
* connection limits
* memory
* load balancing
* graceful shutdown
* reconnect storms
* heartbeats
* backpressure

So:

> **Using HTTP does not mean SSE has no persistent-connection scaling problems.**

---

# 3.38 Connection Limits

Suppose one server supports:

```text
50,000 SSE connections
```

and each connection consumes some amount of memory.

If we need:

```text
500,000 clients
```

we may need multiple servers.

```text
500,000 clients
       │
       ▼
Load Balancer
   │    │    │
   ▼    ▼    ▼
 SSE1  SSE2  SSE3
```

The exact connection capacity depends on:

* runtime
* OS limits
* memory
* file descriptors
* network
* application behavior
* buffering
* event frequency

Never assume a universal "connections per server" number.

---

# 3.39 Backpressure in SSE

Suppose the server produces:

```text
10,000 events/sec
```

but a client can only consume:

```text
100 events/sec
```

Then data can accumulate.

```text
Producer
   │
   │ 10,000/sec
   ▼
SSE Server
   │
   │ buffer grows
   ▼
Slow Client
   │
   │ 100/sec
```

If the buffer grows without bound:

```text
Memory
  ↑
  │
  │        /
  │      /
  │    /
  │  /
  └──────────────► Time
```

Eventually the server can run out of memory.

Possible strategies include:

* bound buffers
* drop low-value events
* keep only the latest state
* disconnect slow clients
* apply upstream pressure
* use durable messaging for events that must not be lost

We will study backpressure separately later.

---

# 3.40 SSE and Message Semantics

Suppose the server sends:

```text
data: {"temperature":25}
data: {"temperature":26}
data: {"temperature":27}
```

Do we need every event?

Maybe not.

For a temperature dashboard, the latest value may be enough.

If:

```text
25
26
27
28
29
```

arrives while the client is slow, we may only care about:

```text
29
```

But for a financial transaction or chat message:

```text
Transaction A
Transaction B
Transaction C
```

dropping events may be unacceptable.

Therefore:

> **The importance of an event determines the delivery and buffering strategy.**

---

# 3.41 SSE Is Not a Message Queue

This distinction is very important.

SSE is a **delivery mechanism**.

It does not automatically provide:

* durable storage
* retries across arbitrary failures
* consumer groups
* durable message queues
* exactly-once processing

You can build these capabilities around SSE, but SSE itself does not provide them.

Think:

```text
Message System
      │
      ▼
    SSE
      │
      ▼
   Browser
```

The message system and transport are separate concerns.

---

# 3.42 SSE vs Redis Pub/Sub

These are also not alternatives in the same category.

### SSE

Moves events:

```text
Server → Browser
```

### Redis Pub/Sub

Moves events:

```text
Application Server ↔ Application Server
```

For example:

```text
Application
    │
    ▼
Redis Pub/Sub
    │
    ▼
SSE Server
    │
    ▼
Browser
```

They can work together.

---

# 3.43 SSE and Authentication

An SSE endpoint may need authentication.

For example:

```text
GET /events
Authorization: Bearer ...
```

or authentication through the application's session mechanism.

The server should determine:

```text
Who is this client?
What events is this client allowed to receive?
```

For example:

```text
User A
   │
   ▼
SSE connection
   │
   ▼
Only User A's notifications
```

This is especially important because an SSE connection can remain open for a long time.

---

# 3.44 SSE and Authorization Changes

Because the connection is long-lived, another subtle issue appears.

Suppose a user's permissions change while the SSE connection is already open.

The server may need to decide whether:

* the connection remains valid
* the user should be disconnected
* future events should reflect the new permissions

This is one example of why long-lived connections require more lifecycle thinking than ordinary HTTP requests.

---

# 3.45 Graceful Shutdown

Suppose Server A is being deployed.

It currently has:

```text
30,000 SSE connections
```

If the process is killed immediately:

```text
Server A
   │
   ├── Client 1 X
   ├── Client 2 X
   ├── Client 3 X
   └── ...
```

thousands of clients reconnect simultaneously.

A graceful shutdown can instead:

```text
Stop accepting new connections
        │
        ▼
Notify/drain existing connections
        │
        ▼
Allow clients to reconnect gradually
        │
        ▼
Shutdown server
```

Connection draining becomes an important production concern.

---

# 3.46 The Thundering Herd Problem Applies Here Too

Imagine:

```text
SSE Server crashes
       │
       ▼
100,000 clients disconnect
       │
       ▼
100,000 reconnect attempts
```

This can overload:

* load balancer
* application servers
* authentication services
* Redis
* databases

So reconnection strategy matters even with SSE.

`EventSource` provides basic automatic reconnection, but large-scale systems may still need infrastructure and application-level strategies to control reconnect storms.

---

# 3.47 SSE Connection Lifecycle

A useful mental model:

```text
              Client creates EventSource
                         │
                         ▼
                    HTTP Request
                         │
                         ▼
                  Server accepts
                         │
                         ▼
                 Connection Open
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
          Send Event              Heartbeat
              │                     │
              └──────────┬──────────┘
                         │
                         ▼
                  Continue streaming
                         │
                 network failure
                         │
                         ▼
                    Connection Lost
                         │
                         ▼
                      Reconnect
                         │
                         ▼
                 Resume if possible
```

---

# 3.48 Complete SSE Flow

Let's put everything together.

```text
┌──────────────┐
│    Client    │
└──────┬───────┘
       │
       │ GET /events
       │ Accept: text/event-stream
       ▼
┌──────────────┐
│ Load Balancer│
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ SSE Server   │
└──────┬───────┘
       │
       │ subscribe
       ▼
┌──────────────┐
│ Event Source │
│ / PubSub     │
└──────┬───────┘
       │
       │ event
       ▼
┌──────────────┐
│ SSE Server   │
└──────┬───────┘
       │
       │ data: {...}
       ▼
┌──────────────┐
│    Client    │
└──────────────┘
```

The connection remains open between the SSE server and client.

---

# 3.49 SSE's Strengths

SSE is attractive because it provides:

### Simple API

Browsers provide:

```javascript
EventSource
```

### HTTP-based infrastructure

It works through the familiar HTTP ecosystem.

### Server-to-client streaming

Excellent for one-way updates.

### Automatic reconnection

The browser provides basic reconnect behavior.

### Event semantics

Named events and IDs are built into the mechanism.

### Lower conceptual complexity

For server-to-client updates, SSE can be simpler than WebSockets.

---

# 3.50 SSE's Limitations

SSE is not suitable for every real-time application.

### One-way communication

It doesn't provide full bidirectional messaging over the same connection.

### Text-oriented

SSE is fundamentally a text event stream.

### Persistent connections

You still have to manage:

* connection count
* memory
* timeouts
* load balancing
* reconnection

### Proxy infrastructure

Streaming must work correctly through:

* reverse proxies
* load balancers
* CDNs
* gateways

### Mobile/network behavior

Long-lived connections can be affected by network transitions and device power-management behavior.

---

# 3.51 When SSE Is a Good Fit

SSE is particularly attractive when:

```text
Server → Client
```

is the dominant communication pattern.

Examples:

### Notifications

```text
Server ───► Browser
```

### Live dashboards

```text
Server ───► Browser
```

### Progress updates

```text
Server ───► Browser
```

### AI streaming

```text
Server ───► Browser
```

### Live feeds

```text
Server ───► Browser
```

---

# 3.52 When SSE Is Probably Not Enough

Suppose you are building:

### Chat

Client needs to send:

```text
Client ─── message ───► Server
```

and receive:

```text
Server ─── message ───► Client
```

continuously.

SSE can still be combined with HTTP requests, but if the application fundamentally requires frequent two-way communication, WebSockets become a natural technology to evaluate.

---

### Multiplayer game

The communication is frequently:

```text
Client ◄──────► Server
```

SSE is not designed for this pattern.

---

### Real-time collaboration

Multiple users continuously send edits:

```text
Client A ◄──────► Server
Client B ◄──────► Server
Client C ◄──────► Server
```

Bidirectional communication is central.

WebSockets are generally more appropriate to evaluate.

---

# 3.53 Decision Tree

A simple first-pass decision tree:

```text
                 Need real-time updates?
                         │
                        Yes
                         │
                         ▼
             Who primarily sends data?
                         │
              ┌──────────┴──────────┐
              │                     │
            Server                Both
              │                     │
              ▼                     ▼
             SSE               WebSockets
              │
              │
              ▼
     Is polling sufficient?
              │
        ┌─────┴─────┐
       Yes           No
        │             │
     Polling         SSE
```

This isn't a universal rule.

It's simply a useful starting point.

---

# 3.54 A More Practical Decision Framework

Ask these questions:

### Question 1

Do we actually need real-time updates?

If no:

```text
Normal HTTP / Polling
```

may be sufficient.

---

### Question 2

Does the server primarily push information?

If yes:

```text
SSE
```

may be appropriate.

---

### Question 3

Does the client also need frequent communication?

If yes:

```text
WebSockets
```

becomes a stronger candidate.

---

### Question 4

Do we need peer-to-peer audio/video/data?

Then:

```text
WebRTC
```

should be considered.

---

# 3.55 The Big Picture

We have now evolved from:

```text
Short Polling

Client → Server
Client ← Server
```

to:

```text
Long Polling

Client → Server
         │
         │ wait
         ▼
Client ← Server
```

to:

```text
SSE

Client → Server
         │
         ├── Event ──► Client
         ├── Event ──► Client
         ├── Event ──► Client
         └── Event ──► Client
```

and eventually:

```text
WebSocket

Client ◄──────────────► Server
       messages both ways
```

The evolution is about **communication requirements**, not simply replacing old technology with newer technology.

---

# 3.56 Key Takeaways

## 1. SSE is built on HTTP

It uses a long-lived HTTP response.

```text
HTTP Request
     │
     ▼
Persistent HTTP Response
     │
     ├── Event
     ├── Event
     └── Event
```

---

## 2. SSE is primarily one-way

```text
Server ─────────► Client
```

The client can still use normal HTTP requests to send data.

---

## 3. SSE supports multiple events over one connection

Unlike long polling:

```text
Request → Response
Request → Response
```

SSE allows:

```text
Request
   │
   ├── Event
   ├── Event
   ├── Event
   └── Event
```

---

## 4. `EventSource` provides a browser API

It manages the SSE connection and provides basic automatic reconnection.

---

## 5. SSE has event semantics

Important fields include:

```text
event:
data:
id:
retry:
```

---

## 6. Reconnection and recovery are different

Reconnection means:

> "Connect again."

Recovery means:

> "Find what I missed and deliver it."

Event IDs and `Last-Event-ID` can help with recovery, but the server needs a way to retrieve missed events.

---

## 7. Persistent HTTP connections still have scaling costs

You must consider:

* concurrent connections
* memory
* file descriptors
* timeouts
* proxy behavior
* load balancing
* graceful shutdown
* reconnect storms
* backpressure

---

## 8. SSE is not a message queue

SSE is a **transport/delivery mechanism**.

A separate system may be responsible for:

* storing events
* retrying events
* durable delivery
* fan-out
* message processing

---

## 9. SSE and Pub/Sub can work together

A common architecture is:

```text
Application
     │
     ▼
 Redis Pub/Sub
     │
     ▼
SSE Servers
     │
     ▼
Clients
```

---

# 3.57 Final Mental Model

Remember SSE like this:

```text
                         SSE
                          │
                          ▼
              One HTTP connection
                          │
                          ▼
                  Server keeps it open
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
           Event        Event        Event
             │            │            │
             └────────────┼────────────┘
                          ▼
                        Client
```

And compare the three mechanisms we've learned:

```text
┌───────────────────────────────────────────────────┐
│                 SHORT POLLING                     │
│                                                   │
│ Client → Request → Server → Response → repeat     │
└───────────────────────────────────────────────────┘

                       ↓

┌───────────────────────────────────────────────────┐
│                 LONG POLLING                      │
│                                                   │
│ Client → Request → Server waits → Response        │
│              → Client requests again              │
└───────────────────────────────────────────────────┘

                       ↓

┌───────────────────────────────────────────────────┐
│                     SSE                           │
│                                                   │
│ Client → Request → Server keeps response open     │
│                    ├── Event                      │
│                    ├── Event                      │
│                    ├── Event                      │
│                    └── Event                      │
└───────────────────────────────────────────────────┘

                       ↓

┌───────────────────────────────────────────────────┐
│                  WEBSOCKET                        │
│                                                   │
│ Client ◄──────── persistent connection ────────►  │
│        messages can travel in both directions     │
└───────────────────────────────────────────────────┘
```

The key lesson:

> **SSE gives us a persistent server-to-client event stream over HTTP. It removes the need for repeated polling when the server needs to continuously push updates, while remaining simpler than a fully bidirectional WebSocket connection.**

---

# 3.58 What Comes Next

The next chapter is where we move to **WebSocket Fundamentals**.

We will answer:

* Why was SSE not enough for some applications?
* What does "full-duplex" actually mean?
* What happens when a WebSocket connection is established?
* How does HTTP Upgrade work?
* What is `101 Switching Protocols`?
* Why does a WebSocket start with HTTP?
* What does a persistent WebSocket connection actually look like?
* How are messages sent in both directions?
* What happens when a WebSocket closes?
* What does a basic WebSocket server actually manage?

After that, we'll go one level deeper into the **WebSocket protocol itself—RFC 6455, frames, opcodes, masking, ping/pong, fragmentation, and the handshake fields.**

---

⬅️ **[Back: Polling & Long Polling](02_Polling_And_Long_Polling.md)**

➡️ **[Next: WebSocket Fundamentals](04_WebSockets_Fundamentals.md)**

⬆️ **[Back to Real-Time Backend](README.md)**
