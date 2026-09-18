# Chapter 8 — Redis Pub/Sub and the Real-Time Backplane

In the previous chapter, we looked at scaling WebSocket servers.

We identified a major problem:

> **WebSockets are stateful.**

If Client A connects to Server 1, that TCP connection lives exclusively on Server 1.

```text
Client A
   │
   │ (Persistent Connection)
   ▼
Server 1
```

If Client B connects to Server 2:

```text
Client B
   │
   │ (Persistent Connection)
   ▼
Server 2
```

Now, suppose Client A and Client B are in the same chat room.

Client A sends a message:

```json
{
  "room": "general",
  "text": "Hello everyone!"
}
```

Server 1 receives this message.

But Server 1 looks at its local memory and says:

> "I only have Client A connected to me. I don't know who Client B is."

Server 1 drops the message.

Client B never sees it.

---

# 8.1 The Invisible Wall

Between Server 1 and Server 2, there is an invisible communication wall.

```text
Server 1     │     Server 2
             │
Client A     │     Client B
```

They are completely isolated.

To solve this, we need a way for Server 1 to tell Server 2:

> "Hey, Client A just sent a message to the general room!"

We need a communication channel that sits *behind* the WebSocket servers.

This leads us to the concept of a **Backplane**.

---

# 8.2 What is a Backplane?

A backplane is a high-speed, central messaging bus.

It acts as the nervous system connecting all your WebSocket servers together.

```text
                  Load Balancer
                  /           \
                 ▼             ▼
             Server 1       Server 2
               /               \
         Client A             Client B
               \               /
                \             /
                 ▼           ▼
               THE BACKPLANE
```

When you introduce a backplane, the architecture fundamentally changes.

Instead of keeping messages trapped inside local server memory, servers do this:

1. Receive a message from a client.
2. Shout the message into the backplane.
3. Listen to the backplane for messages shouted by other servers.

This immediately shatters the invisible wall.

---

# 8.3 Choosing a Backplane Technology

To act as a backplane, a technology needs to be:

* Extremely fast (low latency)
* Able to support broadcasting (one-to-many)
* Able to handle high throughput

There are many options:

* Apache Kafka
* RabbitMQ
* NATS
* Redis

But the undisputed industry standard for simple real-time WebSockets is **Redis**.

Specifically, a feature inside Redis called **Pub/Sub**.

---

# 8.4 Redis Pub/Sub

Redis is primarily known as a lightning-fast, in-memory database used for caching.

But Redis also has a built-in messaging paradigm called **Publish / Subscribe (Pub/Sub)**.

Pub/Sub is entirely separate from the normal key-value storage of Redis.

It does not store data on disk.

It relies on three core concepts:

1. Publishers
2. Subscribers
3. Channels

Let's look at each one in depth.

---

# 8.5 Publishers

A **Publisher** is any entity that sends a message into the system.

In our real-time architecture, the publishers are our WebSocket servers.

```text
Server 1
   │
   │ "Publish message"
   ▼
Redis
```

When a client sends a chat message to Server 1, Server 1 turns around and publishes it to Redis.

The publisher doesn't care who is listening.

It simply blindly throws the message into Redis.

---

# 8.6 Subscribers

A **Subscriber** is any entity that is actively listening for messages.

In our real-time architecture, ALL of our WebSocket servers act as subscribers.

```text
Redis
   │
   │ "Here is a message"
   ▼
Server 2
```

Whenever a server boots up, it connects to Redis and says:

> "I am subscribing. Send me any messages that pass through here."

So in reality, our WebSocket servers are both **Publishers** and **Subscribers** simultaneously.

---

# 8.7 Channels

If Server 1 just yelled every message into the void, and Server 2 had to listen to millions of irrelevant messages a second, the system would crash.

We need a way to organize the messages.

This is where **Channels** come in.

A channel is simply a named topic string.

For example:

```text
chat:room:general
```

Or:

```text
notifications:user:123
```

When a Publisher sends a message, it doesn't just send it to "Redis". It sends it to a specific channel.

```text
Server 1 ──► PUBLISH "chat:room:general" ──► Redis
```

When a Subscriber listens, it doesn't listen to everything. It subscribes to a specific channel.

```text
Server 2 ──► SUBSCRIBE "chat:room:general"
```

Only subscribers listening to that exact channel will receive the broadcast.

---

# 8.8 Message Propagation (The Full Flow)

Let's trace a message through the entire distributed architecture.

Assume Client A and Client B are in `Room 5`.

Client A is on Server 1.
Client B is on Server 2.

### Step 1: The Client Sends

Client A types "Hello!" and hits enter.

The browser sends it over the WebSocket connection.

```text
Client A ──► "Hello!" ──► Server 1
```

---

### Step 2: The Server Publishes

Server 1 receives the message.

It executes a Redis command:

```text
PUBLISH room:5 "Hello!"
```

```text
Server 1 ──► [room:5] "Hello!" ──► Redis
```

---

### Step 3: The Cross-Server Broadcast

Redis immediately looks at its internal mapping.

Who is subscribed to `room:5`?

Server 1, Server 2, and Server 3 are all subscribed.

Redis instantly pushes the exact same message to all three servers simultaneously.

```text
Redis ──► Server 1 (New message on room:5)
Redis ──► Server 2 (New message on room:5)
Redis ──► Server 3 (New message on room:5)
```

---

### Step 4: Local Delivery

Now, every server has a copy of the message.

Each server looks at its local active connections.

**Server 1** sees: "Oh, Client A is here. But they already sent it. I can ignore this."

**Server 3** sees: "I don't have anyone in Room 5 connected to me right now. Dropping message."

**Server 2** sees: "Client B is here, and they are in Room 5!"

Server 2 pushes the message down the WebSocket pipe.

```text
Server 2 ──► "Hello!" ──► Client B
```

The gap is bridged. 

The invisible wall between Server 1 and Server 2 has been shattered by Redis.

---

# 8.9 The Danger of Pub/Sub

This architecture sounds perfect.

It allows us to scale horizontally to 100 WebSocket servers seamlessly.

But Redis Pub/Sub has a very specific, very dangerous constraint that you MUST understand for system design:

> **At-Most-Once Delivery**

---

# 8.10 At-Most-Once Delivery

Redis Pub/Sub is inherently a **Fire-and-Forget** mechanism.

Redis does not save the messages.

It does not hold them in memory.

It does not write them to disk.

When a message arrives on a channel, Redis says:

> "Are there any subscribers listening right at this exact millisecond?"

If YES: It pushes the message to them.

If NO: It deletes the message instantly.

---

# 8.11 What Happens During a Server Crash?

Imagine this timeline.

```text
Time = 12:00:00
```
Server 2 experiences a brief network glitch and crashes.

It is no longer subscribed to Redis.

```text
Time = 12:00:01
```
Client A (on Server 1) sends a message.

Server 1 publishes the message to Redis.

Redis looks for subscribers. It only sees Server 1 and Server 3.

It delivers the message to them.

```text
Time = 12:00:03
```
Server 2 finishes rebooting.

It reconnects to Redis and subscribes to the channels again.

**Will Server 2 receive the message from 12:00:01?**

No.

The message is gone forever.

Client B missed the message completely.

This is the trade-off of pure Pub/Sub. You gain blistering speed and broadcast capabilities, but you lose reliability during disconnections.

---

# 8.12 Why Not Use Message Queues?

Because Redis Pub/Sub drops messages, junior engineers often suggest:

> *"Let's just use a Message Queue instead! Like RabbitMQ or AWS SQS. They save the messages safely on disk!"*

This sounds logical.

But Message Queues and Pub/Sub solve two entirely different system design problems.

---

# 8.13 The Queuing Model

A Message Queue (like AWS SQS) is designed to distribute heavy work evenly among workers.

Imagine an Image Processing Queue with 1,000 images waiting to be resized.

You have 5 Worker Servers listening to the queue.

If an image arrives in the queue:

```text
Queue ──► Image
```

**ONLY ONE** worker will receive the image.

```text
Worker 1 ──► gets the image
Worker 2 ──► gets nothing
Worker 3 ──► gets nothing
```

This is called **Point-to-Point** or **Competing Consumers**.

It is fantastic for batch jobs and asynchronous workloads.

---

# 8.14 Why Queues Fail for Real-Time Backplanes

Now, try using a Message Queue for our chat room.

Client A sends a message.

It goes into the RabbitMQ queue.

```text
Queue ──► "Hello!"
```

Server 1, Server 2, and Server 3 are listening to the queue.

Because it is a queue, **only one server** gets the message!

Suppose Server 3 gets it.

```text
Server 3 ──► receives "Hello!"
```

But Client B (the intended recipient) is connected to Server 2!

Server 2 never got the message. Client B never sees the chat.

This is why Message Queues fail for WebSocket backplanes.

For real-time connections, we don't want to distribute the work to one server. We want to **broadcast** the event to ALL servers instantly.

---

# 8.15 Redis Streams: The Conceptual Bridge

So, we have two extremes:

1. **Redis Pub/Sub:** Broadcasts to everyone, but drops messages instantly if a server blips off.
2. **Message Queues:** Extremely durable and safe, but only delivers to one server.

Is there a technology that does both?

Yes.

 Technologies like **Apache Kafka** or **Redis Streams** provide persistent broadcasting.

---

# 8.16 How Redis Streams Work

Redis Streams allow multiple servers to read the exact same message (like Pub/Sub), but the messages are actually saved to disk/memory (like a Queue).

```text
Publisher ──► Redis Stream (Persisted Log)
                        │
                        ├──► Server 1 (Reading from offset 10)
                        └──► Server 2 (Reading from offset 10)
```

Every message in a Stream gets a permanent, unique ID.

If Server 2 crashes at `12:00:00` and reboots at `12:00:05`, it doesn't lose the messages!

Server 2 connects to Redis and says:

> "I crashed. Give me all the messages that happened on `room:5` since my last known ID."

Redis replays the missed messages, and Server 2 catches up perfectly.

---

# 8.17 Why We Usually Stick to Pub/Sub

If Redis Streams is safer, why don't we use it for all WebSockets?

### 1. Massive Memory Cost
If you have 10,000 chat messages flowing per second, persisting them all in Redis RAM is unbelievably expensive.

### 2. High Operational Complexity
Managing stream offsets, consumer groups, memory eviction policies, and garbage collection of old logs requires significant infrastructure engineering.

### The Real-World Compromise
For standard WebSocket backplanes, engineering teams generally accept the "At-Most-Once" limitation of simple Redis Pub/Sub.

To handle dropped messages during server crashes, we offload the responsibility to the **Client**.

When the Client's WebSocket connection drops and reconnects, the Client simply makes a traditional HTTP request:

```http
GET /chat/history?since=TIMESTAMP
```

It fetches the missed messages directly from the primary database, completely bypassing the real-time backplane.

---

# 8.18 The Big Picture

We now understand how to scale WebSocket servers horizontally.

```text
1. Clients connect via a Load Balancer.
2. The Load Balancer maintains stateful TCP pipes.
3. Servers push local state into Redis Pub/Sub.
4. Redis instantly broadcasts it back out to all servers.
```

But solving the scaling problem creates a host of new reliability problems.

What happens when the client loses network connectivity?
What happens when 50,000 clients all reconnect to the Load Balancer at the exact same millisecond?

We will explore this in the next chapter.

---

⬅️ **[Previous: 7. Scaling WebSockets](07_Scaling_WebSockets.md)** | 🏠 **[Back to TOC](README.md)** | **[Next: 9. Reliability of Real-Time Connections ➡️](09_Real_Time_Reliability.md)**
