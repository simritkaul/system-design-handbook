# Chapter 6 — Server-Sent Events (SSE)

## Goal

Understand how a server can continuously push updates to a client over a single HTTP connection, why this is useful for one-way real-time communication, and when SSE is a better fit than long polling or WebSockets.

---

# 1. The Problem

In the previous chapter, we used long polling to improve ordinary polling.

Instead of repeatedly asking:

```text
Any updates?
```

the client asks once and waits.

```text
Client                         Server

Any updates? ────────────────►

                                Wait...

                                Event occurs

             ◄──────────────── Update
```

This is useful.

But notice what happens after the server sends the update.

```text
Response completes

↓

Client sends another request

↓

Server waits again
```

Now imagine a live dashboard.

The server generates:

```text
CPU Usage: 45%

CPU Usage: 52%

CPU Usage: 48%

CPU Usage: 61%

CPU Usage: 43%
```

The client wants to continuously receive these updates.

With long polling:

```text
Request
   ↓
Update 1
   ↓
New Request
   ↓
Update 2
   ↓
New Request
   ↓
Update 3
```

There is still repeated request lifecycle overhead.

But there is another important observation.

The communication only flows in one direction.

```text
Server ─────────► Client
```

The client doesn't need to continuously send messages back.

So WebSockets may be more than we need.

We need something in between:

> A persistent connection like WebSockets, but primarily for server-to-client communication and built directly on HTTP.

That is the problem Server-Sent Events solve.

---

# 2. Why Existing Solutions Fail

Let's look at the options we already have.

## Polling

```text
Client ───► Server

Client ───► Server

Client ───► Server
```

Problem:

> Too many unnecessary requests.

---

## Long Polling

```text
Client ───► Server

             Wait...

Server ───► Client

Client ───► Server again
```

Better.

But every update eventually leads to another request.

---

## WebSockets

```text
Client ◄══════════════► Server
```

Excellent for continuous bidirectional communication.

But what if all we need is:

```text
Server ───────────────► Client
```

For example:

- News updates
- Notifications
- Live dashboards
- Monitoring data
- Job progress
- Sports scores

We don't necessarily need a full-duplex communication channel.

So the next evolution is:

```text
Client opens connection once

↓

Server keeps it open

↓

Server continuously sends events
```

---

# 3. The Big Idea

> **Server-Sent Events allow a server to continuously stream updates to a client over one long-lived HTTP connection.**

The client establishes the connection.

The server keeps it open.

Then whenever something happens:

```text
Event

↓

Server

↓

Client receives update
```

The same connection can carry many events.

---

# 4. Detailed Explanation

Imagine opening a live cricket score page.

You open the page once.

You do not want this:

```text
GET /score

GET /score

GET /score

GET /score
```

Instead:

```text
Browser
   │
   │ Connect once
   ▼
Score Server
   │
   │ Score: 120/3
   ▼
Browser
   │
   │ Score: 121/3
   ▼
Browser
   │
   │ Score: 125/3
   ▼
Browser
```

The connection remains open.

The server sends updates as events occur.

---

## The Basic Flow

```text
Client
   │
   │ HTTP Request
   │ "I want to receive events"
   ▼
Server
   │
   │ Keeps connection open
   │
   ├──────────► Event 1
   │
   ├──────────► Event 2
   │
   ├──────────► Event 3
   │
   └──────────► Event 4
```

The client does not create a new request after every event.

One connection carries a stream of events.

---

# 5. SSE Is Still Based on HTTP

This is an important difference from WebSockets.

Conceptually:

```text
HTTP Connection
       │
       ▼
Server keeps response open
       │
       ▼
Streams events over time
```

The server begins responding, but instead of sending the entire response and closing the interaction, it keeps the response stream open.

You can think of it as:

```text
Normal HTTP

Request
   │
   ▼
Complete Response
   │
   ▼
Done
```

versus:

```text
SSE

Request
   │
   ▼
Response begins
   │
   ▼
Event
   │
   ▼
Event
   │
   ▼
Event
   │
   ▼
Connection remains open
```

This makes SSE conceptually simpler when the communication model is naturally one-way.

---

# 6. One-Way Communication

The most important characteristic of SSE is:

```text
Server ─────────► Client
```

The server pushes events.

The client receives them.

But SSE is not intended to provide a continuous bidirectional message channel like WebSockets.

Of course, the client can still make ordinary HTTP requests separately.

For example:

```text
Client
   │
   ├── HTTP POST ───► Server
   │
   └── SSE Stream ◄── Server
```

Imagine a notification application.

The client might send:

```text
POST /notifications/read
```

using ordinary HTTP.

Meanwhile, the server can continuously push:

```text
New notification
```

through the SSE connection.

So the application can still communicate both ways overall.

The distinction is that the **SSE stream itself is server-to-client**.

---

# 7. An Example: Live Notifications

Suppose a user is logged into a social media application.

They open the application.

The browser creates an SSE connection.

```text
User Browser
      │
      │ Connect
      ▼
Notification Service
```

The connection remains open.

Then:

```text
Someone liked your post
```

The server sends:

```text
Notification Event
```

Later:

```text
You have a new follower
```

Another event.

```text
User Browser
      ▲
      │ New follower
      │
      ▲
      │ New like
      │
Notification Server
```

The client simply listens.

---

# 8. Before vs After Architecture

## Before — Long Polling

```text
Client                         Server

Request ──────────────────────►

                                Wait...

                                Event

             ◄──────────────── Update


New Request ──────────────────►

                                Wait...

                                Event

             ◄──────────────── Update
```

Every response eventually requires another request.

---

## After — SSE

```text
Client                         Server

Connect ──────────────────────►

             ◄──────────────── Event 1

             ◄──────────────── Event 2

             ◄──────────────── Event 3

             ◄──────────────── Event 4
```

One connection.

Many events.

---

# 9. SSE vs WebSockets

At first glance, SSE and WebSockets can look similar.

Both can maintain long-lived connections.

Both can support real-time updates.

But their communication models are different.

| SSE                                               | WebSocket                                       |
| ------------------------------------------------- | ----------------------------------------------- |
| Primarily server → client                         | Client ↔ server                                 |
| One-way event stream                              | Full-duplex communication                       |
| Built on HTTP streaming                           | Upgrades to WebSocket protocol                  |
| Good for server-driven updates                    | Good for interactive real-time communication    |
| Client can use normal HTTP separately for sending | Both sides exchange messages on same connection |

The most important question is:

> **Who needs to speak?**

If only the server continuously needs to send updates:

```text
Server ───► Client
```

SSE may be sufficient.

If both sides continuously exchange messages:

```text
Client ◄────► Server
```

WebSockets may be more natural.

---

# 10. SSE vs Long Polling

Both approaches solve the server-push problem.

But they do it differently.

## Long Polling

```text
Request
   │
   ▼
Wait
   │
   ▼
One response
   │
   ▼
New request
```

---

## SSE

```text
One connection
      │
      ├── Event
      ├── Event
      ├── Event
      └── Event
```

Long polling repeatedly recreates requests.

SSE keeps one response stream open.

So for continuous or repeated server-driven updates, SSE can be more efficient and conceptually cleaner.

---

# 11. Connection Failures and Reconnection

Persistent connections eventually fail.

For example:

```text
Wi-Fi disconnects

Mobile network changes

Server restarts

Load balancer closes connection
```

The SSE connection disappears.

The client needs to reconnect.

Conceptually:

```text
Connected

    │

    ▼

Connection lost

    │

    ▼

Reconnect

    │

    ▼

Continue receiving events
```

This raises an important question.

Suppose the client disconnects after receiving:

```text
Event 100
```

While disconnected, the server produces:

```text
Event 101
Event 102
Event 103
```

When the client reconnects:

> How does it know what it missed?

This is where event identity becomes useful.

Conceptually:

```text
Last received event: 100
```

On reconnection:

```text
Give me events after 100.
```

The system may then replay missed events if the architecture supports storing them.

This begins connecting real-time communication with deeper concepts such as:

- Event delivery
- Event storage
- Ordering
- Replay
- Messaging

We will explore these ideas in Module 7.

---

# 12. SSE and Horizontal Scaling

Suppose we have several SSE servers.

```text
                  Load Balancer
                 /      |      \
                ▼       ▼       ▼
             SSE 1    SSE 2    SSE 3
```

A user connects to SSE Server 1.

Now an event occurs somewhere else in the system.

```text
Order Service
      │
      │ Event
      ▼
     ???
      │
      ▼
SSE Server 1
      │
      ▼
   Connected User
```

The system needs to distribute the event to the server that holds the user's active connection.

At scale, this often requires some form of event distribution.

Conceptually:

```text
                 Event Source
                      │
                      ▼
              Event Distribution
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
       SSE 1        SSE 2        SSE 3
         │
         ▼
      Client
```

Again, the communication protocol solves only part of the problem.

Once many servers are involved, we need architectural mechanisms for distributing events.

That is one of the central motivations behind messaging and event-driven systems.

---

# 13. Where SSE Helps

SSE is particularly useful when:

### The server produces a continuous stream of updates

```text
Server ─────► Client
```

Examples:

- Live dashboards
- Notifications
- Monitoring systems
- News feeds
- Sports scores
- Market data
- Job progress

---

### The client mostly listens

The client does not need to continuously send messages through the same channel.

---

### You want HTTP-based communication

SSE fits naturally into the HTTP ecosystem.

This can simplify integration compared with introducing a fully separate bidirectional protocol.

---

# 14. Where SSE Doesn't Help

SSE is not ideal when:

### Both sides communicate continuously

For example:

```text
Multiplayer game

Client → Server: movement

Server → Client: world updates

Client → Server: actions

Server → Client: other players
```

A full-duplex communication model is more natural.

---

### Messages are extremely interactive in both directions

Chat can sometimes use SSE for incoming messages plus HTTP for outgoing messages, but if both sides exchange high-frequency messages, WebSockets often provide a more natural communication channel.

---

### Infrastructure cannot efficiently support long-lived HTTP streams

Proxies, load balancers, and servers must be configured to support long-running connections correctly.

---

# 15. Real-World Usage

SSE fits naturally into systems where the client behaves primarily as a listener.

For example:

## Monitoring Dashboard

```text
Application Metrics
        │
        ▼
Monitoring System
        │
        ▼
SSE Server
        │
        ▼
Dashboard
```

Every time a metric changes:

```text
CPU: 40%

↓

CPU: 55%

↓

CPU: 48%
```

the dashboard can update without polling.

---

## Job Progress

Imagine uploading a large video.

```text
Upload
   │
   ▼
Video Processing
   │
   ├── 10%
   ├── 30%
   ├── 60%
   ├── 90%
   └── Complete
```

The server can stream progress events to the user.

---

## Notifications

```text
Event occurs
      │
      ▼
Notification System
      │
      ▼
SSE Connection
      │
      ▼
User receives notification
```

---

# 16. Mental Model

## Radio Station

SSE is like listening to a radio station.

```text
Radio Station ─────────► Listener
```

The station continuously broadcasts.

The listener receives the stream.

The listener does not speak back through the radio channel.

If the listener wants to communicate with the station, they can use another mechanism:

```text
Listener ───► Phone Call / Message
```

Similarly:

```text
Server ─────────► Client
```

through SSE.

The client can still use normal HTTP requests for other actions.

WebSockets, by comparison, are more like a telephone call:

```text
Person A ◄────────► Person B
```

Both sides can continuously speak.

---

# 17. Tradeoffs

## Advantages

- Efficient for repeated server-to-client updates.
- One long-lived connection can carry many events.
- Built on HTTP.
- Simpler than full-duplex communication when only server push is needed.
- Avoids repeated request creation used by long polling.
- Fits naturally with event-based applications.

## Disadvantages

- Communication through the SSE stream is one-way.
- Persistent connections still consume infrastructure resources.
- Reconnection and missed-event handling require careful design.
- Horizontal scaling requires event distribution between servers.
- Not ideal for high-frequency bidirectional communication.
- Long-lived HTTP connections may require careful infrastructure configuration.

---

# 18. Common Interview Questions

## What problem does SSE solve?

SSE allows a server to continuously push events to a client over one long-lived HTTP connection.

It avoids repeated polling or repeatedly creating long-poll requests when updates are ongoing.

---

## How is SSE different from WebSockets?

SSE is primarily:

```text
Server → Client
```

WebSockets support:

```text
Client ↔ Server
```

SSE is useful when the client mainly receives updates.

WebSockets are useful when both sides need continuous communication.

---

## How is SSE different from long polling?

Long polling:

```text
One request
    ↓
One eventual response
    ↓
New request
```

SSE:

```text
One request/stream
    ↓
Many events
    ↓
Connection remains open
```

---

## Can the client send data when using SSE?

Yes, the application can send data using separate mechanisms such as ordinary HTTP requests.

But the SSE stream itself is designed for server-to-client event delivery.

---

## What happens when an SSE connection breaks?

The client reconnects.

The system may use event identifiers or another application-level mechanism to determine whether events were missed and whether they should be replayed.

---

# 19. Before vs After Architecture

## Before — Repeated Long Polls

```text
Client
   │
   ├── Request ──► Server
   │                  │
   │                  ▼
   │                Event
   │
   ◄── Response ──────┘

   │
   ├── New Request ──► Server
   │
   ◄── Next Event ────
```

---

## After — Event Stream

```text
Client
   ▲
   │ Event 1
   │
   │ Event 2
   │
   │ Event 3
   │
   │ Event 4
   │
Server
```

The client connects once and continuously receives updates.

---

# 20. Connections

We now have several ways to support real-time communication.

```text
Polling
   │
   ▼
Client repeatedly asks
```

↓

```text
Long Polling
   │
   ▼
Client asks once and waits
```

↓

```text
SSE
   │
   ▼
Server continuously streams events
```

↓

```text
WebSockets
   │
   ▼
Both sides continuously communicate
```

This raises an important architectural question.

> If WebSockets, SSE, and long polling all enable different forms of real-time communication, how do we decide which one to use?

The answer depends on:

- Who initiates communication?
- Who needs to send updates?
- How frequently do messages occur?
- Is bidirectional communication required?
- Is HTTP infrastructure sufficient?
- What level of connection-management complexity is acceptable?

Before moving to the next protocol, we will eventually need to understand how modern HTTP itself evolved to handle communication more efficiently.

But first, we need to study a very different communication model: **gRPC**.

REST, GraphQL, WebSockets, long polling, and SSE are often discussed in the context of browser and web applications.

But backend services also need to communicate with each other.

Imagine:

```text
Order Service
      │
      ▼
Payment Service
      │
      ▼
Inventory Service
      │
      ▼
Shipping Service
```

These services may make enormous numbers of internal requests.

Now the questions change:

> Do we always need human-readable JSON?

> Is HTTP/1.1 request-response communication efficient enough?

> Can services communicate using strongly defined contracts?

> Can we support streaming between backend services efficiently?

Those questions lead naturally to:

# Chapter 7 — gRPC

---

# 21. Key Takeaways

- SSE enables continuous server-to-client event streaming.
- The client establishes one long-lived HTTP connection.
- The server can send many events over that connection.
- SSE is useful when communication is primarily one-way.
- It avoids the repeated request cycle of long polling.
- It is simpler than WebSockets when full-duplex communication is unnecessary.
- SSE connections still require reconnection, scaling, and event-distribution strategies.
- The protocol is only one part of the architecture; distributing events across many servers leads naturally toward messaging systems.
- The next communication problem is different: efficient, strongly defined communication between backend services.

**Next: Chapter 7 — gRPC.**
