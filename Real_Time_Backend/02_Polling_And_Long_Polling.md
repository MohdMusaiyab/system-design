# Chapter 2 — Polling & Long Polling

Before WebSockets and Server-Sent Events, applications had to work within the traditional HTTP request-response model.

The fundamental problem was simple:

> **How can a client learn that something changed on the server when the client doesn't know exactly when the change will happen?**

Polling is one of the simplest answers.

This chapter covers:

* Short Polling
* Polling intervals
* Request frequency and server load
* Freshness vs resource usage
* Long Polling
* HTTP connection lifecycle
* Timeouts
* Scaling implications
* Failure scenarios
* Thundering Herd / reconnect storms
* Why these techniques eventually led to persistent real-time connections

---

# 2.1 The Problem We Are Trying to Solve

Consider a chat application.

User A sends a message:

```text
"Hey, are you free?"
```

The backend stores it:

```text
Database
   │
   └── Message exists
```

But User B's browser doesn't automatically know that the message exists.

Why?

Because a normal HTTP interaction looks like:

```text
Client ─────── Request ───────► Server
Client ◄────── Response ─────── Server
```

The server sends a response **because the client made a request**.

It doesn't normally have an open channel through which it can spontaneously send another HTTP response later.

So we need a mechanism for:

```text
Server has new data
        │
        ▼
How does client find out?
```

One simple solution is:

> **Keep asking.**

That's polling.

---

# 2.2 What Is Polling?

**Polling** means that a client repeatedly sends requests to a server to check whether something has changed.

For example:

```text
Every 5 seconds:

Client ─── "Anything new?" ───► Server
Client ◄────── "No" ─────────── Server

       5 seconds later

Client ─── "Anything new?" ───► Server
Client ◄────── "No" ─────────── Server

       5 seconds later

Client ─── "Anything new?" ───► Server
Client ◄──── "Yes, new data" ── Server
```

This is called **Short Polling**.

The client determines how frequently it checks.

---

# 2.3 Why Is It Called "Short" Polling?

The request is expected to complete quickly.

The lifecycle is:

```text
Request
   │
   ▼
Server checks state
   │
   ▼
Server responds
   │
   ▼
Connection/request ends
```

Then, after some interval:

```text
Request
   │
   ▼
Response
   │
   ▼
Wait
   │
   ▼
Request again
```

The connection is not intentionally kept open waiting for an event.

---

# 2.4 A Simple Example

Suppose the browser wants to know whether a background job has completed.

It might request:

```http
GET /jobs/123/status
```

The server responds:

```json
{
  "status": "processing"
}
```

The browser waits two seconds and asks again:

```http
GET /jobs/123/status
```

Response:

```json
{
  "status": "processing"
}
```

Again:

```http
GET /jobs/123/status
```

Response:

```json
{
  "status": "completed"
}
```

Now the browser updates the UI.

---

# 2.5 The Polling Loop

Conceptually:

```text
              ┌─────────────────────┐
              │                     │
              ▼                     │
         Send Request               │
              │                     │
              ▼                     │
         Server Checks              │
              │                     │
              ▼                     │
         Send Response              │
              │                     │
              ▼                     │
          Wait N seconds             │
              │                     │
              └─────────────────────┘
```

The important thing is:

> **The client controls the frequency of checking.**

---

# 2.6 What Does the Server Actually Do?

Suppose the client asks:

```http
GET /notifications
```

The server may perform something like:

```text
Request
   │
   ▼
Authenticate user
   │
   ▼
Check notifications
   │
   ▼
Return result
```

If nothing changed:

```json
{
  "notifications": []
}
```

If something changed:

```json
{
  "notifications": [
    {
      "id": 42,
      "message": "Someone liked your post"
    }
  ]
}
```

From the server's perspective, each poll is usually just another normal HTTP request.

That simplicity is one of polling's biggest advantages.

---

# 2.7 Polling Is Not Necessarily "Bad"

It's easy to think:

> "Polling is old, therefore polling is bad."

That's incorrect.

Polling is often a perfectly reasonable solution.

For example:

### Background job status

```text
Processing...
Processing...
Completed
```

If the job takes several minutes, checking every few seconds may be completely acceptable.

### Dashboard

If data only needs to update every 30 seconds:

```text
GET /dashboard
```

every 30 seconds may be simpler than maintaining thousands of persistent connections.

### Low-frequency updates

If an event happens once every few minutes, WebSockets may introduce unnecessary infrastructure complexity.

The correct question is:

> **Does the application's freshness requirement justify a more complex communication mechanism?**

---

# 2.8 The First Major Problem: Wasted Requests

Suppose a user checks for notifications every 5 seconds.

Over one minute:

```text
60 / 5 = 12 requests
```

If the user receives only one notification during that minute, most requests may return:

```text
"No new notifications."
```

So we get:

```text
Request 1 → Nothing
Request 2 → Nothing
Request 3 → Nothing
Request 4 → Nothing
Request 5 → Nothing
Request 6 → New notification
Request 7 → Nothing
...
```

The client is spending resources asking a question whose answer is often:

> "Nothing changed."

---

# 2.9 Why Wasted Requests Matter at Scale

For one user:

```text
12 requests/minute
```

is almost irrelevant.

Now imagine:

```text
100,000 users
```

polling every 5 seconds.

Each user:

```text
12 requests/minute
```

Total:

```text
100,000 × 12
= 1,200,000 requests/minute
```

That's:

```text
20,000 requests/second
```

And remember:

> **Most of those requests may simply be checking whether anything happened.**

This is where polling can become expensive.

---

# 2.10 Polling Can Amplify Downstream Load

The HTTP server isn't necessarily the only thing receiving work.

Consider:

```text
Client
   │
   │ Poll
   ▼
Load Balancer
   │
   ▼
Application Server
   │
   ▼
Redis / Database
```

Every poll may cause:

* authentication checks
* application processing
* cache lookups
* database queries
* serialization
* network traffic

So one client request can cause work across several components.

At scale:

```text
Many clients
     │
     ▼
Many polls
     │
     ▼
Many application requests
     │
     ▼
Many database/cache operations
```

Polling can therefore become a **load multiplier**.

---

# 2.11 Polling Interval

The polling interval determines how often the client asks for updates.

Examples:

```text
1 second
5 seconds
10 seconds
30 seconds
60 seconds
```

Choosing the interval is a trade-off.

### Short interval

```text
1 second
```

Advantages:

* fresher data
* lower maximum waiting time

Disadvantages:

* many requests
* higher server load
* more network traffic
* more database/cache work

### Long interval

```text
30 seconds
```

Advantages:

* fewer requests
* lower infrastructure load

Disadvantages:

* stale information
* potentially higher delay before detecting updates

---

# 2.12 Freshness vs Cost

Think of polling as:

```text
             Freshness
                ▲
                │
                │
                │
                │
                └──────────────► Resource Cost
```

Generally:

```text
More frequent polling
        ↓
Better freshness
        ↓
Higher resource consumption
```

while:

```text
Less frequent polling
        ↓
Lower resource consumption
        ↓
More stale data
```

This is a classic system-design trade-off.

---

# 2.13 Maximum Detection Delay

Suppose we poll every 10 seconds.

An event can happen immediately after a poll:

```text
Poll
 │
 │
 │ Event happens here
 │
 │
 │
Next poll
```

The client may not discover the event until almost 10 seconds later.

So with an interval of:

```text
10 seconds
```

the detection delay can be approximately:

```text
0–10 seconds
```

under an idealized model.

On average, if events are uniformly distributed relative to the polling schedule, the detection delay is roughly half the interval.

So:

```text
10-second interval
≈ 5-second average detection delay
```

This is only an approximation; real systems also have processing and network latency.

---

# 2.14 Short Polling and Request Bursts

There's another problem.

Imagine 100,000 clients all load a page at:

```text
12:00:00
```

and all start polling every:

```text
10 seconds
```

They may synchronize:

```text
12:00:00 → 100,000 requests
12:00:10 → 100,000 requests
12:00:20 → 100,000 requests
12:00:30 → 100,000 requests
```

This creates periodic traffic spikes.

Instead of:

```text
smooth traffic
────────────────────
```

we can get:

```text
Requests
  ▲
  │    /\      /\      /\
  │   /  \    /  \    /  \
  │__/    \__/    \__/    \__
  └──────────────────────────► Time
```

A common mitigation is **jitter**—slightly randomizing polling times.

Instead of everyone polling exactly every 10 seconds:

```text
Client A → 9.8 sec
Client B → 10.3 sec
Client C → 9.5 sec
Client D → 10.7 sec
```

This spreads traffic out.

---

# 2.15 Conditional Polling

Polling doesn't always need to download the entire dataset every time.

HTTP provides mechanisms that can help clients ask:

> "Has this resource changed?"

For example:

```http
If-None-Match: "abc123"
```

The server can respond:

```http
304 Not Modified
```

if nothing changed.

This can reduce response-body transfer.

The important distinction is:

```text
Conditional request
≠
No request
```

The server still receives and processes the request.

So conditional requests can reduce **data transfer**, but they don't completely eliminate the cost of polling.

---

# 2.16 Polling With a Timestamp or Cursor

Another common approach is asking for only changes after a particular point.

For example:

```http
GET /messages?after=message_100
```

The server returns:

```json
{
  "messages": [
    {
      "id": 101,
      "text": "Hello"
    },
    {
      "id": 102,
      "text": "How are you?"
    }
  ]
}
```

This is much better than repeatedly downloading all messages.

The general idea is:

```text
Client remembers:
"I have seen up to X."

Client asks:
"Give me everything after X."
```

This pattern becomes useful later for **reconnection and message recovery** in real-time systems as well.

---

# 2.17 Why Not Just Poll Very Frequently?

You might think:

> "If 1 second is better than 10 seconds, why not poll every 100 milliseconds?"

Because the cost grows rapidly.

Suppose:

```text
100,000 clients
```

poll every:

```text
100 ms
```

That's:

```text
10 requests/sec/client
```

Therefore:

```text
100,000 × 10
=
1,000,000 requests/second
```

Most applications don't need that kind of request rate just to discover occasional changes.

This is one of the fundamental motivations for more efficient real-time communication mechanisms.

---

# 2.18 Long Polling

Short polling asks:

> "Anything new?"

and immediately gets:

```text
"No."
```

Long polling changes the strategy:

> **"Tell me when something new happens, but don't respond immediately if nothing has happened."**

The server keeps the request open for some period.

```text
Client ───────────── Request ─────────────► Server
                                             │
                                             │
                                             │ waiting...
                                             │
                                             │ waiting...
                                             │
                                       event occurs
                                             │
                                             ▼
Client ◄──────────── Response ─────────────── Server
```

---

# 2.19 Long Polling Step by Step

Suppose the client sends:

```http
GET /messages
```

The server checks:

```text
New message?
```

No.

Instead of immediately returning:

```json
{
  "messages": []
}
```

the server waits.

```text
Request
   │
   ▼
Check for data
   │
   ▼
Nothing yet
   │
   ▼
Wait
   │
   ▼
Wait
   │
   ▼
New message arrives
   │
   ▼
Return response
```

Now the client receives the response.

---

# 2.20 Long Polling Timeline

```text
Time ───────────────────────────────────────────────►

Client       Server

  │             │
  │── Request ─►│
  │             │
  │             │ waiting...
  │             │
  │             │ waiting...
  │             │
  │             │ New event
  │             │
  │◄── Response ─
  │             │
  │
  │── Request ─►│
  │             │
  │             │ waiting...
```

The important point:

> **The server doesn't continuously send data through the same response. It eventually completes the request. The client then makes another request.**

This distinction matters when we compare long polling with SSE and WebSockets.

---

# 2.21 Long Polling Is Still Request-Response

Long polling may look like a persistent connection, but conceptually it is still:

```text
Request
   ↓
Wait
   ↓
Response
```

Then:

```text
Request
   ↓
Wait
   ↓
Response
```

It is not a true bidirectional persistent communication channel.

Compare:

### Long Polling

```text
Request → wait → response
Request → wait → response
Request → wait → response
```

### WebSocket

```text
Connect
   ↓
Persistent connection
   ↕
Messages
   ↕
Messages
   ↕
Messages
   ↓
Close
```

This difference becomes important later.

---

# 2.22 Why Does Long Polling Help?

The key benefit is that the client doesn't repeatedly ask and immediately receive:

```text
"No change."
```

Instead:

```text
Client asks
     ↓
Server waits
     ↓
Something happens
     ↓
Server responds
```

So the response itself becomes an indication that something happened.

This reduces unnecessary empty responses.

---

# 2.23 Long Polling With Timeout

The server cannot necessarily wait forever.

Suppose the client sends:

```text
GET /events
```

The server waits for an event.

But what if no event occurs for:

```text
10 minutes?
```

Keeping the request open indefinitely creates problems.

Instead, the server usually establishes a timeout.

For example:

```text
Maximum wait = 30 seconds
```

Then:

```text
Request
   │
   ▼
Wait
   │
   ├──── event occurs ────► Response with event
   │
   └──── timeout ─────────► Empty response
```

The client then starts another long-poll request.

---

# 2.24 Why Use a Timeout?

A timeout gives the server a way to periodically clean up old requests.

Without a timeout:

```text
Client
   │
   │ request
   ▼
Server
   │
   │ waiting...
   │
   │ waiting...
   │
   │ waiting...
   │
   │ forever?
```

A timeout gives us:

```text
Request
   │
   ▼
Wait
   │
   ▼
Timeout
   │
   ▼
Response
   │
   ▼
Connection ends
```

This makes resource management more predictable.

---

# 2.25 Important: HTTP Timeout vs Application Timeout

There can be multiple timeout layers.

For example:

```text
Client timeout
       │
Load balancer timeout
       │
Reverse proxy timeout
       │
Application server timeout
       │
Database timeout
```

They don't necessarily have the same value.

Suppose your application wants to hold a long-poll request for:

```text
60 seconds
```

but the load balancer closes idle connections after:

```text
30 seconds
```

Then the application may never get the full 60 seconds.

This is an important production consideration.

> **The entire request path must agree with the intended long-polling timeout.**

---

# 2.26 Long Polling Lifecycle

A simplified lifecycle:

```text
1. Client creates request
          ↓
2. Server receives request
          ↓
3. Server checks for event
          ↓
4. No event
          ↓
5. Server waits
          ↓
6. Event occurs OR timeout
          ↓
7. Server sends response
          ↓
8. Request ends
          ↓
9. Client immediately creates another request
```

Notice step 9.

The client has to repeat the process.

---

# 2.27 Long Polling Example

Imagine a notification system.

The browser opens:

```http
GET /notifications/stream
```

Server:

```text
No notification yet.
Keep request open.
```

Then someone likes the user's post.

Backend:

```text
New notification
      ↓
Long-poll request wakes up
      ↓
Return notification
```

Client receives:

```json
{
  "type": "LIKE",
  "message": "Someone liked your post"
}
```

Then the client immediately creates another request:

```http
GET /notifications/stream
```

and waits again.

---

# 2.28 How Does the Server "Wait"?

This is an important under-the-hood question.

The server doesn't necessarily have to continuously execute:

```text
while (!event) {
    checkDatabase();
}
```

That would be terrible.

Instead, a well-designed application can use an event-driven mechanism.

Conceptually:

```text
Long Poll Request
       │
       ▼
Register interest
       │
       ▼
Suspend request
       │
       │
       │ event occurs
       ▼
Resume request
       │
       ▼
Send response
```

The exact implementation depends on the framework and architecture.

The important concept is:

> **Waiting for an event should not mean burning CPU in a busy loop.**

---

# 2.29 Long Polling and the Database

A naïve implementation might do:

```text
Long Poll Request
      │
      ▼
Check DB
      │
      ▼
Nothing
      │
      ▼
Wait 500 ms
      │
      ▼
Check DB again
      │
      ▼
Nothing
      │
      ▼
...
```

This is effectively polling **inside the server**.

It can create unnecessary database load.

A better architecture might use an event source:

```text
                    New Event
                       │
                       ▼
                   Event Bus
                       │
                       ▼
                Long Poll Handler
                       │
                       ▼
                    Client
```

This is one reason real-time systems often introduce messaging or event-driven mechanisms as they become more sophisticated.

---

# 2.30 Short Polling vs Long Polling

| Property              | Short Polling       | Long Polling                |
| --------------------- | ------------------- | --------------------------- |
| Request               | Repeated            | Repeated                    |
| Response              | Usually immediate   | Delayed until event/timeout |
| Connection            | Short-lived         | Held for a while            |
| Empty responses       | Common              | Reduced                     |
| Freshness             | Depends on interval | Usually better              |
| Server complexity     | Low                 | Higher                      |
| Client complexity     | Low                 | Moderate                    |
| Persistent connection | No                  | No                          |
| Bidirectional         | No                  | No                          |
| Scaling complexity    | Relatively simple   | Higher                      |

The key difference:

```text
Short Polling:
Ask → Answer → Wait → Ask again

Long Polling:
Ask → Wait → Answer → Ask again
```

---

# 2.31 Resource Usage: Short vs Long Polling

Short polling primarily creates:

```text
Request overhead
+
Response overhead
+
Repeated downstream work
```

Long polling reduces the number of immediate empty responses, but it creates:

```text
Long-lived in-flight requests
+
Connection/resource management
+
Timeout management
```

So long polling doesn't eliminate cost.

It **changes the type of cost**.

This is a very important system-design lesson.

> **Optimization rarely makes a cost disappear. It usually moves the cost somewhere else.**

---

# 2.32 Long Polling at Scale

Imagine:

```text
100,000 connected clients
```

with long polling.

Many clients may have active requests:

```text
Load Balancer
     │
     ├── Server A → 30,000 waiting requests
     ├── Server B → 35,000 waiting requests
     └── Server C → 35,000 waiting requests
```

These aren't necessarily consuming 100% CPU.

But they are still resources that the infrastructure must manage.

You need to consider:

* file descriptors
* socket resources
* memory
* server connection limits
* proxy/load-balancer limits
* timeouts
* connection lifecycle
* reconnect behavior

---

# 2.33 Open Connections Are Not Free

This is a crucial concept for all persistent or semi-persistent communication.

If a server maintains:

```text
100,000 open connections
```

it must maintain some state associated with those connections.

That may include:

* socket information
* connection metadata
* authentication context
* buffers
* request state
* framework objects
* memory allocations

The exact cost depends heavily on the runtime and implementation.

Therefore:

> **Concurrent connections are a capacity dimension of their own.**

This becomes even more important with WebSockets.

---

# 2.34 Load Balancing Long Polling

Suppose we have:

```text
              Load Balancer
              /           \
             ▼             ▼
         Server A       Server B
```

A client sends:

```text
Long Poll Request
```

and gets assigned to Server A.

Server A waits for an event.

Now the event happens somewhere else:

```text
Database / Event System
       │
       ▼
    Server B
```

But the client's long-poll request is sitting on:

```text
Server A
```

So Server B needs a way to notify Server A.

This introduces a distributed coordination problem.

---

# 2.35 Why Shared Event Infrastructure Helps

A common architecture becomes:

```text
                 Application
                     │
                     ▼
                  Pub/Sub
                 /       \
                ▼         ▼
            Server A   Server B
                │           │
                ▼           ▼
             Clients     Clients
```

Now if an event occurs:

```text
Event
  │
  ▼
Pub/Sub
  │
  ├────► Server A
  │
  └────► Server B
```

The server holding the relevant client's connection can deliver the event.

This concept becomes extremely important when we study **WebSocket scaling**.

---

# 2.36 Sticky Sessions

Another technique you may encounter is **sticky sessions**, also called session affinity.

The idea is:

> A particular client tends to be routed to the same backend server.

For example:

```text
Client A
   │
   ▼
Load Balancer
   │
   └────────► Server A
```

Future requests from Client A are routed to Server A.

This can make some state-management problems easier.

However:

> **Sticky sessions do not magically make a distributed system consistent.**

If Server B needs to communicate with Client A, Server B still needs a way to reach Server A or otherwise access shared state.

We will study sticky sessions in much greater detail in the WebSocket scaling chapter.

---

# 2.37 What Happens When a Server Restarts?

Imagine:

```text
Client
   │
   ▼
Server A
   │
   │ long-poll request waiting
   │
   ▼
Server crashes
```

The request disappears.

The client eventually detects the failure and creates another request.

```text
Server A
   ↓
Crash
   ↓
Client detects failure
   ↓
Reconnect
   ↓
Load Balancer
   ↓
Server B
```

This is where reconnection behavior becomes important.

---

# 2.38 Reconnection

A real-time client should generally expect that communication can fail.

A simplified client loop:

```text
Connect
   │
   ▼
Wait for response
   │
   ├── Event → process → reconnect
   │
   └── Timeout → reconnect
```

But there is a danger.

Suppose the server goes down and:

```text
100,000 clients
```

all reconnect immediately.

---

# 2.39 The Thundering Herd Problem

A **thundering herd** occurs when many clients react to the same event simultaneously and create a huge burst of work.

For example:

```text
Server restart
      │
      ▼
100,000 clients disconnected
      │
      ▼
100,000 reconnect attempts
      │
      ▼
Load Balancer
      │
      ▼
Backend servers overloaded
```

This can cause a feedback loop:

```text
Overload
   ↓
Requests fail
   ↓
Clients retry
   ↓
More requests
   ↓
More overload
```

This is extremely dangerous.

---

# 2.40 Exponential Backoff

One common solution is **exponential backoff**.

Instead of retrying immediately:

```text
Retry
Retry
Retry
Retry
```

the client gradually increases the waiting time.

For example:

```text
1st retry → 1 second
2nd retry → 2 seconds
3rd retry → 4 seconds
4th retry → 8 seconds
5th retry → 16 seconds
```

Usually there is also a maximum delay.

For example:

```text
max delay = 30 seconds
```

The exact values depend on the application.

---

# 2.41 Jitter

Even exponential backoff can synchronize clients.

Suppose all 100,000 clients use:

```text
1 sec
2 sec
4 sec
8 sec
```

They may still retry at roughly the same times.

**Jitter** adds randomness.

For example:

```text
Client A → 1.2 sec
Client B → 0.8 sec
Client C → 1.5 sec
Client D → 0.9 sec
```

Now reconnect attempts are spread over time.

Conceptually:

```text
Without jitter:

Requests
  ▲
  │     │       │
  │     │       │
  │     │       │
  └─────┼───────┼────────► Time


With jitter:

Requests
  ▲
  │   ╱╲  ╱╲ ╱╲
  │ ╱   ╲╱  ╲  ╲
  └──────────────────────► Time
```

The goal is to smooth the load.

---

# 2.42 Long Polling and Message Loss

Suppose:

```text
Client
   │
   │ long poll
   ▼
Server
```

The server returns:

```text
Message 101
```

Then the client disconnects before creating the next request.

Meanwhile:

```text
Message 102
Message 103
Message 104
```

arrive.

What happens?

It depends on the application's architecture.

If the server only keeps events in temporary memory, messages may be lost.

If messages are stored durably, the client may reconnect with a cursor:

```http
GET /messages?after=101
```

and receive:

```text
102
103
104
```

This introduces an important principle:

> **Connection management and message durability are separate problems.**

A connection can tell you how data travels.

It does not automatically guarantee that data will never be lost.

---

# 2.43 Delivery Guarantees

Real-time systems eventually need to discuss:

* At-most-once delivery
* At-least-once delivery
* Exactly-once processing

For this chapter, understand the basic distinction.

### At-most-once

A message may be delivered:

```text
0 or 1 times
```

It may be lost.

### At-least-once

A message is retried until delivery is considered successful.

It may arrive more than once.

```text
Message 101
Message 101   ← duplicate
```

Therefore the consumer may need idempotency.

### Exactly-once

The system aims for the logical effect of processing a message exactly once.

This is considerably harder than simply saying:

> "Send it once."

We will revisit delivery guarantees later.

---

# 2.44 Ordering

Suppose a user sends:

```text
1. "Hello"
2. "How are you?"
3. "Are you free?"
```

The receiver ideally sees:

```text
1 → 2 → 3
```

But distributed systems can produce:

```text
1 → 3 → 2
```

Depending on the architecture, maintaining ordering may require:

* sequence numbers
* partitioning
* single ordered streams
* buffering
* acknowledgements

The important lesson for now:

> **A communication mechanism does not automatically guarantee application-level message ordering.**

---

# 2.45 Short Polling Failure Modes

Short polling is simple, but it can suffer from:

### Too much traffic

Clients poll too frequently.

### Stale data

Clients poll too infrequently.

### Database overload

Every poll triggers a database query.

### Traffic synchronization

Many clients poll simultaneously.

### Waste

Many requests return no new information.

### Scaling pressure

Large client populations create large request rates.

---

# 2.46 Long Polling Failure Modes

Long polling solves some problems but introduces others.

### Open request accumulation

Large numbers of clients can create many active requests.

### Timeout configuration

Every layer must support the intended timeout.

### Reconnection storms

Server failures can cause many clients to reconnect simultaneously.

### Load balancing complexity

Events may need to reach the server holding a particular client's request.

### Resource management

Open requests consume server/proxy/socket resources.

### Message recovery

The system needs a strategy for events that occur around disconnects.

---

# 2.47 Why Long Polling Was Important

Long polling was an important evolutionary step because it demonstrated a different idea:

Instead of:

```text
Client:
"Has anything happened?"

Server:
"No."

Client:
"Has anything happened?"

Server:
"No."
```

we can do:

```text
Client:
"Tell me when something happens."

Server:
"Okay, I'll keep this request open."

             ...

Server:
"Something happened."
```

This significantly improves the communication model.

But it still has the fundamental structure:

```text
Request
   ↓
Wait
   ↓
Response
   ↓
New Request
```

That limitation leads us toward persistent streaming mechanisms.

---

# 2.48 Polling → Long Polling → Persistent Connections

The evolution can now be understood clearly:

### Short Polling

```text
Ask repeatedly
```

```text
Request → Response
Request → Response
Request → Response
```

### Long Polling

```text
Ask and wait
```

```text
Request → Wait → Response
Request → Wait → Response
```

### Persistent Streaming

```text
Open once and keep communicating
```

```text
Connect
   │
   ├── Event
   ├── Event
   ├── Event
   ├── Event
   └── Close
```

SSE and WebSockets take the next step.

---

# 2.49 When Should You Use Short Polling?

Short polling can be a good choice when:

* updates are infrequent
* a few seconds of delay is acceptable
* the number of clients is relatively small
* implementation simplicity is important
* the system already uses normal HTTP APIs
* you don't need a persistent connection
* the operation is temporary

Examples:

```text
Check background job status
Check whether report is ready
Refresh dashboard periodically
Fetch exchange/configuration data periodically
```

---

# 2.50 When Should You Consider Long Polling?

Long polling can make sense when:

* updates should arrive relatively quickly
* you want to reduce empty polling responses
* persistent WebSocket infrastructure isn't necessary
* HTTP-based infrastructure is preferable
* communication is still fundamentally request/response

However, for new systems, the choice should be based on actual requirements and infrastructure rather than assuming long polling is always preferable.

---

# 2.51 When Polling Is Actually the Better Engineering Choice

Suppose:

```text
100 users
```

need an update every:

```text
30 seconds
```

A simple polling endpoint may be completely adequate.

Building:

```text
Load Balancer
      │
WebSocket cluster
      │
Redis Pub/Sub
      │
Presence system
      │
Connection management
```

would add substantial complexity.

This leads to an important engineering principle:

> **Don't introduce real-time infrastructure when ordinary HTTP already satisfies the requirement.**

---

# 2.52 Decision Framework

When deciding whether polling is enough, ask:

```text
How quickly must updates appear?
             │
             ▼
How frequently do updates occur?
             │
             ▼
How many clients are involved?
             │
             ▼
How expensive is each poll?
             │
             ▼
Can the system tolerate stale data?
             │
             ▼
Does the client need two-way communication?
```

If:

```text
Low frequency
+
Moderate freshness requirement
+
Simple architecture desired
```

polling may be sufficient.

If:

```text
High frequency
+
Low latency requirement
+
Large number of connected clients
+
Continuous communication
```

persistent real-time communication becomes much more attractive.

---

# 2.53 Production Mental Model

When you see polling in a system, don't just think:

> "The frontend calls an API repeatedly."

Think about the entire path:

```text
Client
   │
   │ Poll
   ▼
Load Balancer
   │
   ▼
Application Server
   │
   ├── Authentication
   │
   ├── Cache
   │
   └── Database
   │
   ▼
Response
   │
   ▼
Client
   │
   │ wait
   │
   └──────────────► Poll again
```

Then ask:

> **What happens when there are 10,000? 100,000? 1,000,000 clients?**

That is where system design begins.

---

# 2.54 Production Mental Model for Long Polling

For long polling:

```text
Client
   │
   │ Long Poll
   ▼
Load Balancer
   │
   ▼
Application Server
   │
   │
   │ waiting for event
   │
   ▼
Event Source / PubSub
   │
   │ event
   ▼
Application Server
   │
   ▼
Client
   │
   │ reconnect
   ▼
Load Balancer
```

Now the important questions become:

* How many open requests can each server handle?
* How long can requests remain open?
* What happens during deployment?
* What happens if the server crashes?
* Where do events live?
* How are events recovered?
* How do we avoid reconnect storms?

These are the questions we will carry into the next chapters.

---

# 2.55 Important Distinction: Connection vs Request

This distinction is worth remembering.

### Short polling

```text
Request
   ↓
Response
   ↓
Done
```

### Long polling

```text
Request
   ↓
Wait
   ↓
Response
   ↓
Done
```

### WebSocket

```text
Connection
   ↓
Message
   ↕
Message
   ↕
Message
   ↕
Message
   ↓
Close
```

Long polling keeps an **HTTP request** alive.

A WebSocket establishes a **persistent communication connection**.

These are not the same thing.

---

# 2.56 Common Misconceptions

## Misconception 1: "Polling is real-time."

It can provide near-real-time behavior, but the freshness is limited by the polling interval and other processing/network delays.

---

## Misconception 2: "Long polling is a WebSocket."

No.

Long polling is still repeated HTTP request-response.

---

## Misconception 3: "Long polling eliminates server load."

No.

It reduces repeated empty responses but creates long-lived requests and connection-management costs.

---

## Misconception 4: "A persistent connection guarantees message delivery."

No.

Connection lifetime and message durability are separate concerns.

---

## Misconception 5: "Polling every 1 second means 1-second latency."

Not exactly.

The update can be detected within roughly the polling interval under ideal conditions, but actual end-to-end latency also includes:

* event generation
* server processing
* network latency
* client processing
* UI rendering

---

## Misconception 6: "More frequent polling is always better."

Only if the additional freshness is worth the additional resource consumption.

---

# 2.57 Key Takeaways

### Short Polling

> **Client repeatedly asks the server for updates.**

Simple but potentially wasteful.

```text
Ask → Answer → Wait → Repeat
```

---

### Long Polling

> **Client asks once and the server waits before responding.**

It reduces unnecessary empty responses.

```text
Ask → Wait → Answer → Repeat
```

---

### Polling Interval

Controls the trade-off between:

```text
Freshness
     ↕
Resource Cost
```

---

### Scaling

Large numbers of polling clients can generate enormous request rates.

Persistent connections instead introduce a different scaling dimension:

```text
Concurrent Connections
```

---

### Timeouts

Long polling requires carefully configured timeouts across:

```text
Client
Load Balancer
Proxy
Application Server
```

---

### Reconnection

Failures require clients to reconnect.

Naive immediate retries can create:

```text
Thundering Herd
```

Use:

```text
Exponential Backoff
+
Jitter
```

where appropriate.

---

### Delivery

A connection does not automatically guarantee:

* delivery
* ordering
* durability
* exactly-once processing

Those are separate system-design concerns.

---

# 2.58 Final Mental Model

The entire chapter can be reduced to this:

```text
                    Need Updates
                         │
                         ▼
                Can client ask?
                         │
                         ▼
                  Short Polling
                         │
                         │
                  Too many empty
                    responses?
                         │
                         ▼
                  Long Polling
                         │
                         │
                 Need continuous
                    streaming?
                         │
                         ▼
                  SSE / WebSocket
```

And the evolution is:

```text
SHORT POLLING

Client ──Request──► Server
Client ◄─Response── Server
       wait
Client ──Request──► Server
Client ◄─Response── Server


LONG POLLING

Client ──Request────────────► Server
                              │
                              │ wait
                              │
                              │ event
                              ▼
Client ◄────Response───────── Server
       │
       │ new request
       ▼


PERSISTENT CONNECTION

Client ◄────────────────────► Server
       │       open           │
       │◄──────event──────────│
       │──────message────────►│
       │◄──────event──────────│
       │──────message────────►│
       │       close          │
```

The important lesson is not:

> "WebSockets are better than polling."

The important lesson is:

> **Each communication model exists because it solves a different problem with a different cost.**

As system requirements become more demanding, we move from repeatedly asking for information toward mechanisms that allow the server to deliver information more efficiently.

---

## What Comes Next

In the next chapter, we will study **Server-Sent Events (SSE)**.

We will answer:

* How can a normal HTTP connection remain open?
* How does the browser receive multiple events over one connection?
* What exactly is `text/event-stream`?
* How does `EventSource` work?
* How does SSE reconnect?
* What are event IDs?
* How does SSE compare with long polling?
* How does SSE scale across multiple servers?
* When is SSE enough, and when do we actually need WebSockets?

⬅️ **[Back to Real-Time Backend](README.md)**

⬅️ **[Previous: 1. Real-Time Fundamentals](01_Real_Time_Fundamentals.md)**

➡️ **[Next: 3. Server-Sent Events](03_Server_Sent_Events.md)**
