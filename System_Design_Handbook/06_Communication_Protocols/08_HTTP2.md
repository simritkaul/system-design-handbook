# Chapter 8 — HTTP/2

## Goal

Understand why HTTP/1.1 eventually became a bottleneck for modern web applications, how HTTP/2 changed the way data travels over a connection, and why **multiplexing** became one of the most important improvements in modern web communication.

---

# 1. The Problem

Let's return to a normal web page.

Imagine opening an e-commerce product page.

Your browser needs:

```text
HTML
CSS
JavaScript
Product Image
Product Image
Product Image
User Profile
Recommendations
Reviews
Analytics Scripts
Fonts
```

A single page may require dozens or even hundreds of resources.

Conceptually:

```text
Browser
   │
   ├── Request HTML
   ├── Request CSS
   ├── Request JS
   ├── Request Image 1
   ├── Request Image 2
   ├── Request Image 3
   └── ...
```

HTTP/1.1 already improved things significantly through **persistent connections**.

Instead of:

```text
Request
↓
New TCP Connection
↓
Response
↓
Close
```

we could do:

```text
TCP Connection
      │
      ├── Request
      ├── Response
      ├── Request
      ├── Response
      └── ...
```

This is HTTP Keep-Alive, which we covered earlier.

But eventually, keeping the connection alive was not enough.

The next question became:

> If one connection already exists, why can't multiple requests actively use it at the same time?

That is the problem HTTP/2 addresses.

---

# 2. Why HTTP/1.1 Became Insufficient

Imagine one TCP connection.

The browser sends:

```text
Request A
Request B
Request C
```

Suppose Request A is slow.

For example:

```text
Request A → Large Image
```

while Request B is:

```text
Request B → Small CSS File
```

and Request C is:

```text
Request C → Small JavaScript File
```

Ideally:

```text
CSS and JavaScript should arrive quickly.
```

But HTTP/1.1 communication over a connection had limitations around efficiently handling multiple independent requests and responses simultaneously.

Conceptually, the problem looks like this:

```text
One Connection

Request A ─────────────── Slow response ───────────────►

Request B ── waiting

Request C ── waiting
```

A slow piece of work can delay other work sharing the communication path.

This is related to the idea of **Head-of-Line Blocking**.

---

# 3. What Is Head-of-Line Blocking?

Imagine a supermarket with one checkout line.

```text
Customer A ── Huge Cart
Customer B ── One Item
Customer C ── Two Items
```

The queue looks like:

```text
[Huge Cart] → [One Item] → [Two Items]
```

Even though Customer B only has one item:

```text
Customer B must wait.
```

They are stuck behind the customer in front.

Similarly, if independent pieces of communication are forced to wait behind earlier work, we have a form of head-of-line blocking.

For a modern web page with many resources, this can hurt performance.

---

# 4. The Big Idea

> **Allow multiple independent streams of communication to share the same connection simultaneously.**

Instead of:

```text
One Connection

Request A
    ↓
Response A
    ↓
Request B
    ↓
Response B
```

HTTP/2 introduces the idea of:

```text
One Connection

├── Stream A
├── Stream B
├── Stream C
└── Stream D
```

These streams can make progress independently.

This is called **multiplexing**.

---

# 5. What Is Multiplexing?

Imagine one railway track.

Without multiplexing, you might conceptually think:

```text
Train A passes

↓

Train B passes

↓

Train C passes
```

Only one piece of work progresses at a time.

With multiplexing, think instead of the railway system carrying many independently managed journeys through a shared transportation infrastructure.

For HTTP/2:

```text
One TCP Connection
        │
        ├── Stream 1 → HTML
        ├── Stream 2 → CSS
        ├── Stream 3 → JavaScript
        ├── Stream 4 → Image
        └── Stream 5 → API Call
```

The connection is shared.

But the requests are treated as separate logical streams.

The data from those streams can be interleaved.

Conceptually:

```text
Connection

A1
B1
C1
A2
C2
B2
A3
```

Instead of waiting for all of A to finish:

```text
A1
A2
A3
A4
A5

then B1...
```

HTTP/2 can move pieces of different streams over the same connection.

---

# 6. Streams and Frames

To make multiplexing possible, HTTP/2 breaks communication into smaller units.

Conceptually:

```text
Large Response
      │
      ▼
┌─────┐
│ A1  │
└─────┘

┌─────┐
│ A2  │
└─────┘

┌─────┐
│ A3  │
└─────┘
```

These smaller pieces are called **frames**.

Now multiple streams can share the connection.

```text
Stream A: A1 A2 A3
Stream B: B1 B2
Stream C: C1 C2 C3
```

Over the network:

```text
A1
B1
C1
A2
B2
C2
A3
C3
```

The receiver knows which frame belongs to which stream.

Then it reconstructs:

```text
Stream A → A1 A2 A3
Stream B → B1 B2
Stream C → C1 C2 C3
```

This is the fundamental mechanism behind HTTP/2 multiplexing.

---

# 7. Before vs After Architecture

## HTTP/1.1 — Conceptually

```text
TCP Connection

Request A
    │
    ▼
Response A
    │
    ▼
Request B
    │
    ▼
Response B
```

To improve concurrency, browsers often used multiple connections:

```text
Browser
   ├──── TCP Connection 1
   ├──── TCP Connection 2
   ├──── TCP Connection 3
   └──── TCP Connection 4
```

This allowed more parallel work.

But creating and managing many connections has costs.

---

## HTTP/2

```text
Browser
   │
   │ One TCP Connection
   │
   ├── Stream 1 ── HTML
   ├── Stream 2 ── CSS
   ├── Stream 3 ── JavaScript
   ├── Stream 4 ── Image
   └── Stream 5 ── API
```

One connection can support many concurrent streams.

---

# 8. Why Multiple TCP Connections Are Expensive

Earlier, we learned that creating connections is not free.

A new connection may involve:

```text
TCP Handshake
      ↓
Possibly TLS Handshake
      ↓
Connection setup
```

Then every connection requires resources.

At scale:

```text
More Connections
      │
      ├── More memory
      ├── More operating system resources
      ├── More handshake overhead
      └── More network management
```

HTTP/1.1 applications sometimes opened multiple connections to achieve greater concurrency.

HTTP/2 instead asks:

> Can we achieve more concurrency without repeatedly creating additional connections?

Multiplexing is the answer.

---

# 9. Request Prioritization

Not every resource is equally important.

Suppose a browser loads:

```text
HTML
CSS
JavaScript
Large Background Image
Analytics Script
```

Which is more important?

Usually:

```text
HTML
↓
CSS
↓
Critical JavaScript
```

should receive attention before:

```text
Large Background Image
```

HTTP/2 introduced mechanisms for expressing stream priority and dependency information.

Conceptually:

```text
High Priority

HTML
CSS
Critical JS

──────────────

Lower Priority

Images
Analytics
Non-critical resources
```

This allows the communication layer to make better use of available bandwidth.

The important architectural idea is:

> Not all data has equal urgency.

This same idea appears later in:

- Message queues
- Task scheduling
- Rate limiting
- Backpressure
- Distributed systems

---

# 10. Header Compression

HTTP requests contain more than just the main data.

Consider:

```text
GET /products/123
```

The request may also contain headers.

For example:

```text
Authorization
Cookies
User-Agent
Accept
Language
```

For a browser making many requests:

```text
Request 1 → Same headers
Request 2 → Same headers
Request 3 → Same headers
Request 4 → Same headers
```

Sending the same information repeatedly creates overhead.

HTTP/2 introduced header compression.

Conceptually:

```text
First Request

Full Header Information
```

Then later requests can take advantage of the shared understanding of repeated header information.

The goal is simple:

> Don't repeatedly send large amounts of redundant metadata.

Again, this follows a familiar engineering principle:

```text
Repeated work
      ↓
Identify repetition
      ↓
Avoid unnecessary repetition
```

The same principle gave us caching.

---

# 11. HTTP/2 and Server Push

HTTP/2 also introduced the idea of **server push**.

Imagine the browser requests:

```text
GET /index.html
```

The server knows:

> This HTML page will probably require `style.css`.

So conceptually, the server could proactively send:

```text
HTML requested
      │
      ├── HTML response
      │
      └── style.css proactively
```

The idea was:

> Send something the client will probably need before it explicitly asks.

Conceptually:

```text
Client ───► Give me page

Server ───► Here's the page
       └──► You'll probably need this CSS too
```

This sounds useful.

But it depends on a dangerous assumption:

> The server correctly knows what the client will need.

If the browser already has the resource cached:

```text
Server Push
     ↓
Unnecessary data
```

If the server guesses incorrectly:

```text
Bandwidth wasted.
```

This is an important engineering lesson.

Even a seemingly clever optimization can become harmful if the prediction is wrong.

In practice, server push saw limited adoption and has been deprecated or removed from support in major browser implementations.

The larger lesson matters more than the feature itself:

> Optimization based on prediction is only useful when the prediction is reliable.

---

# 12. Does HTTP/2 Completely Eliminate Head-of-Line Blocking?

This is where we need to be precise.

HTTP/2 solves an important **application-layer** problem.

Multiple HTTP streams can progress independently.

If one response is slow:

```text
Stream A → Slow
```

another stream can still make progress:

```text
Stream B → Continue
Stream C → Continue
```

However, HTTP/2 typically runs over TCP.

TCP itself guarantees ordered delivery of bytes.

Suppose one TCP packet is lost.

```text
Packet 1 ✓
Packet 2 ✓
Packet 3 ✕ Lost
Packet 4 ✓
Packet 5 ✓
```

TCP cannot simply skip Packet 3 and deliver later data to the application as though nothing happened.

It must recover the missing data while preserving ordered delivery.

This means packet loss can still affect multiple HTTP/2 streams sharing that TCP connection.

Conceptually:

```text
One TCP Connection
        │
        ├── Stream A
        ├── Stream B
        └── Stream C

Packet loss
        │
        ▼
TCP recovery can affect all streams
```

So HTTP/2 reduces application-level head-of-line blocking.

But it does not eliminate the underlying transport-level blocking caused by TCP's ordered delivery model.

This becomes the motivation for HTTP/3.

---

# 13. HTTP/2 and gRPC

Now we can better understand the previous chapter.

gRPC needs to support:

```text
Unary calls

Streaming

Multiple concurrent requests
```

HTTP/2 provides useful capabilities for this.

```text
One Connection

├── gRPC Call 1
├── gRPC Call 2
├── gRPC Call 3
└── Streaming Call
```

Each can use its own logical stream.

So:

```text
gRPC
   │
   ▼
HTTP/2
   │
   ▼
Multiplexed Streams
```

This is one reason gRPC fits naturally with HTTP/2.

---

# 14. Real-World Usage

HTTP/2 is particularly valuable for modern web applications.

Imagine a large website.

```text
User
  │
  ▼
Browser
  │
  ▼
Load Balancer / CDN
  │
  ▼
Web Application
```

The browser may need:

```text
HTML
CSS
JavaScript bundles
Fonts
Images
API responses
```

HTTP/2 allows these resources to be handled through multiplexed streams rather than relying on multiple independent TCP connections purely for concurrency.

The architecture conceptually becomes:

```text
Browser
   │
   │ HTTP/2 Connection
   ▼
Server

├── Stream → HTML
├── Stream → CSS
├── Stream → JS
├── Stream → Images
└── Stream → APIs
```

This improves the efficiency of modern resource-heavy applications.

---

# 15. Where HTTP/2 Helps

HTTP/2 is particularly useful when:

### Many resources are requested

```text
One page
   │
   ├── CSS
   ├── JS
   ├── Images
   ├── Fonts
   └── APIs
```

Multiplexing allows these to share a connection efficiently.

---

### Multiple concurrent API calls exist

Modern applications may do:

```text
GET /user
GET /products
GET /notifications
GET /recommendations
```

at approximately the same time.

HTTP/2 allows these calls to use separate streams over one connection.

---

### Connection setup overhead matters

Fewer connections can reduce:

- Handshakes
- Resource consumption
- Connection management overhead

---

### Streaming is useful

Protocols such as gRPC benefit from HTTP/2's streaming and multiplexing capabilities.

---

# 16. Where HTTP/2 Doesn't Magically Help

HTTP/2 does not solve:

### Slow servers

If the database takes:

```text
5 seconds
```

to respond, HTTP/2 cannot make the database faster.

---

### Slow application logic

```text
Bad Algorithm
      ↓
Still bad
```

Multiplexing does not solve inefficient computation.

---

### TCP packet loss entirely

Because HTTP/2 typically uses TCP, transport-level head-of-line blocking can still occur.

---

### Poor system architecture

If your architecture is:

```text
User
  │
  ▼
Single overloaded server
```

switching to HTTP/2 does not suddenly make the system scalable.

Protocols optimize communication.

They do not replace architectural design.

---

# 17. Mental Model

## One Highway, Many Lanes

Imagine HTTP/1.1 communication as a transportation system where independent deliveries can end up waiting behind each other.

A common workaround is:

```text
Build more roads.
```

That means more connections.

HTTP/2 instead tries to make one major highway support many independent flows.

```text
                One Highway

    ┌────────┬────────┬────────┬────────┐
    │ Lane A │ Lane B │ Lane C │ Lane D │
    └────────┴────────┴────────┴────────┘
```

Different vehicles represent different streams.

They share the same infrastructure while still progressing independently at the HTTP layer.

But they all depend on the same underlying road.

If the road itself is blocked:

```text
TCP packet loss
```

everyone can still be affected.

That leads directly to the next evolution.

---

# 18. Tradeoffs

## Advantages

- Multiple streams can share one connection.
- Reduces application-layer head-of-line blocking.
- Reduces the need for multiple connections purely for concurrency.
- Header compression reduces repeated metadata overhead.
- Supports efficient streaming.
- Useful for modern resource-heavy applications.
- Provides a strong foundation for protocols such as gRPC.

## Disadvantages

- More protocol complexity than HTTP/1.1.
- Debugging is less straightforward because communication is binary framed rather than plain text.
- TCP-level head-of-line blocking still exists.
- Benefits depend on the application's request patterns.
- Upgrading the protocol does not solve slow application logic or poor architecture.

---

# 19. Common Interview Questions

## Why was HTTP/2 created?

HTTP/2 was created to improve the efficiency of HTTP communication as modern applications began requiring many concurrent resources and requests.

The key improvement was allowing multiple logical streams to share a single connection.

---

## What is multiplexing?

Multiplexing allows multiple independent streams of communication to share one physical connection.

Conceptually:

```text
One Connection

├── Stream A
├── Stream B
├── Stream C
└── Stream D
```

Frames from different streams can be interleaved.

---

## Does HTTP/2 use multiple TCP connections?

No, the main idea is that multiple logical streams can share a connection.

The exact connection behavior can depend on clients, servers, and connection establishment patterns, but HTTP/2's core multiplexing model does not require opening separate TCP connections for every concurrent request.

---

## What is head-of-line blocking?

It occurs when one piece of work delays other independent work behind it.

HTTP/2 significantly reduces this problem at the HTTP layer through multiplexing.

However, because it generally runs over TCP, packet loss can still cause transport-level blocking.

---

## How does HTTP/2 help gRPC?

gRPC benefits from:

- Multiplexed streams.
- Efficient connection reuse.
- Streaming support.

Multiple RPC calls can share one underlying connection.

---

## Does HTTP/2 make every website faster?

No.

Performance still depends on:

- Network conditions
- Server speed
- Application design
- Payload size
- Caching
- Database performance

HTTP/2 improves the communication layer, not every layer of the system.

---

# 20. Before vs After Architecture

## HTTP/1.1

```text
Browser
   │
   ├──── Connection 1 ─── Resource A
   │
   ├──── Connection 2 ─── Resource B
   │
   ├──── Connection 3 ─── Resource C
   │
   └──── Connection 4 ─── Resource D
```

Multiple connections were often used to increase concurrency.

---

## HTTP/2

```text
Browser
   │
   │
   │ One Connection
   ▼
Server
   │
   ├── Stream 1 ─── Resource A
   ├── Stream 2 ─── Resource B
   ├── Stream 3 ─── Resource C
   └── Stream 4 ─── Resource D
```

One connection can carry multiple independent streams.

---

# 21. Connections

HTTP/2 solved an important problem:

```text
HTTP/1.1
   │
   ▼
Multiple requests competing inefficiently
```

↓

```text
HTTP/2
   │
   ▼
Multiplexed streams
```

But remember the deeper limitation.

All streams usually share:

```text
One TCP connection
```

Suppose one packet is lost.

```text
Packet loss
     │
     ▼
TCP waits for recovery
     │
     ▼
Multiple HTTP/2 streams can be affected
```

So we solved head-of-line blocking at one layer.

But discovered it still exists at a lower layer.

This is exactly the kind of architectural evolution we have been following throughout this handbook:

```text
Solve a problem
      ↓
Discover a deeper limitation
      ↓
Introduce the next design
```

The next question is:

> **Can we keep HTTP/2's benefits while avoiding TCP's transport-level head-of-line blocking?**

That leads us to the final chapter of this module:

# Chapter 9 — HTTP/3

---

# 22. Key Takeaways

- HTTP/1.1 Keep-Alive reduced connection setup overhead but was not enough for modern applications with many concurrent resources.
- HTTP/2 introduced **multiplexing**.
- Multiple logical streams can share a single connection.
- Data is divided into frames, allowing frames from different streams to be interleaved.
- HTTP/2 reduces application-layer head-of-line blocking.
- Header compression reduces repeated metadata overhead.
- HTTP/2's standard transport foundation, TCP, can still cause transport-level head-of-line blocking during packet loss.
- HTTP/2 is especially useful for modern web applications and protocols such as gRPC.
- HTTP/2 improves communication efficiency, but it does not solve slow databases, inefficient application code, or poor architecture.
- The remaining problem lies beneath HTTP itself: **TCP**. HTTP/3 changes that foundation.
