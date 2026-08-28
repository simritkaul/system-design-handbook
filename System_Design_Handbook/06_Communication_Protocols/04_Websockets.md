# Chapter 4 — WebSockets

## Goal

Understand why the traditional HTTP request-response model is insufficient for real-time applications, and how WebSockets enable **persistent, bidirectional communication** between clients and servers.

---

# 1. The Problem

Imagine you're building a chat application.

Alice sends a message:

```text
Alice: Hello!
```

The server receives it.

Now Bob should see the message immediately.

The problem is:

> How does the server tell Bob that a new message has arrived?

With traditional HTTP, communication looks like this:

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

The client initiates communication.

The server responds.

Then the interaction is over.

But Bob may be doing nothing.

His application is simply open and waiting.

The server cannot naturally say:

```text
"Bob, you have a new message!"
```

because Bob hasn't sent a new HTTP request.

Now consider other applications:

- WhatsApp messages
- Slack notifications
- Live stock prices
- Uber driver location updates
- Multiplayer games
- Collaborative document editing
- Live sports scores

In all these systems, something can change on the server **at any moment**.

Waiting for the client to manually ask every time is not ideal.

We need a way for communication to remain open.

---

# 2. Why Existing Solutions Fail

Suppose Bob's application repeatedly asks:

```text
Any new messages?
```

The server replies:

```text
No.
```

A moment later:

```text
Any new messages?
```

Again:

```text
No.
```

This might continue:

```text
Client                  Server

Any updates? ─────────► No

Any updates? ─────────► No

Any updates? ─────────► No

Any updates? ─────────► Yes!
```

This approach is called **polling**.

It works.

But it creates problems.

If we have one million connected users, repeatedly asking:

> "Anything new?"

can generate enormous numbers of unnecessary requests.

The faster we poll:

```text
Every 100 ms
```

the more real-time the application feels.

But the more requests we create.

If we poll slowly:

```text
Every 10 seconds
```

we reduce server load.

But messages can be delayed.

So polling creates a tradeoff:

```text
Frequent Polling
      ↓
Low latency
      +
High overhead


Infrequent Polling
      ↓
Low overhead
      +
Higher latency
```

We need a better approach.

---

# 3. The Big Idea

> **Instead of opening a new request for every interaction, create one long-lived connection that both the client and server can use to send messages whenever necessary.**

This is the central idea behind WebSockets.

---

# 4. Detailed Explanation

With a WebSocket connection:

```text
Client
   ║
   ║ Persistent Connection
   ║
Server
```

The connection stays open.

Both sides can communicate through it.

The client can send:

```text
Hello Server
```

The server can later independently send:

```text
New message for you!
```

Neither side needs to create a brand-new HTTP request for every update.

This creates **full-duplex communication**.

---

## What Does Full-Duplex Mean?

Imagine a walkie-talkie.

Only one person can speak at a time.

That is roughly analogous to **half-duplex communication**.

Now imagine a telephone call.

Both people can speak and listen at the same time.

That is **full-duplex communication**.

WebSockets allow:

```text
Client ─────────► Server

Client ◄───────── Server
```

Both directions can operate independently.

---

# 5. Establishing a WebSocket Connection

WebSockets begin with an interesting idea.

The client initially uses HTTP to ask:

> "Can we switch this connection to the WebSocket protocol?"

Conceptually:

```text
Client
   │
   │ HTTP Request
   │ "Let's upgrade to WebSocket"
   ▼
Server
   │
   │ "Okay"
   ▼
WebSocket Connection
```

After the upgrade:

```text
HTTP Request/Response

        ↓

Persistent WebSocket Connection
```

Now communication is no longer limited to the normal HTTP request-response pattern.

Both sides can send messages over the connection.

The initial HTTP handshake makes WebSockets easier to integrate into the web ecosystem.

---

# 6. Before vs After Architecture

## Before — Repeated HTTP Polling

```text
Client                         Server

Any updates? ────────────────►

             ◄──────────────── No

Any updates? ────────────────►

             ◄──────────────── No

Any updates? ────────────────►

             ◄──────────────── New message!
```

The system repeatedly creates request-response interactions.

---

## After — WebSocket

```text
Client ═══════════════════════ Server
              Connection

Client ──────────────────────► Server

Client ◄────────────────────── Server

Client ──────────────────────► Server

Client ◄────────────────────── Server
```

The connection remains available.

Messages flow whenever they are needed.

---

# 7. WebSocket Changes the Communication Model

This is the most important conceptual shift.

With traditional HTTP:

```text
Client decides when communication happens.
```

With WebSockets:

```text
Client and server can independently communicate.
```

This makes WebSockets useful for **event-driven, real-time interactions**.

For example:

```text
Something happens
       │
       ▼
Server immediately pushes event
       │
       ▼
Connected clients receive it
```

---

# 8. A Chat Application Example

Let's build the idea step by step.

Alice and Bob both open the chat application.

```text
Alice App
    │
    │
    ▼
WebSocket Server
    ▲
    │
    │
Bob App
```

Both establish persistent connections.

Now Alice sends:

```text
Hello Bob!
```

The flow becomes:

```text
Alice
  │
  │ Message
  ▼
WebSocket Server
  │
  │ Find Bob's active connection
  ▼
Bob
```

Bob does not need to ask:

```text
Any messages?
```

The server already has an open communication channel.

It simply sends the message.

---

# 9. The Connection Is the New Resource

With HTTP, the server mostly thinks:

```text
Request arrives

↓

Process request

↓

Send response

↓

Done
```

With WebSockets:

```text
Connection established

↓

Connection remains active

↓

Messages arrive over time

↓

Messages sent over time

↓

Connection eventually closes
```

This changes how we design the server.

The server now needs to manage:

- Active connections
- Connection lifecycle
- Disconnections
- Reconnections
- Slow clients
- Millions of concurrent connections

The communication problem becomes easier for the client.

But the infrastructure problem becomes more complicated.

---

# 10. Connection Lifecycle

A WebSocket generally has several stages.

```text
1. Connect
      │
      ▼
2. Establish WebSocket
      │
      ▼
3. Exchange messages
      │
      ▼
4. Connection remains open
      │
      ▼
5. Disconnect
```

But in real life:

```text
User loses internet

Server crashes

Load balancer terminates connection

Mobile app goes to background
```

The connection can disappear unexpectedly.

So real systems usually need:

```text
Disconnect

↓

Detect failure

↓

Reconnect

↓

Restore application state if necessary
```

Persistent connections introduce **connection management** as a new engineering problem.

---

# 11. WebSockets and Horizontal Scaling

Suppose we have:

```text
                Load Balancer
                /           \
               ▼             ▼
          WS Server 1    WS Server 2
```

Alice connects to Server 1.

Bob connects to Server 2.

```text
Alice
  │
  ▼
Server 1


Bob
  │
  ▼
Server 2
```

Now Alice sends a message to Bob.

Server 1 receives it.

But Bob's WebSocket connection lives on Server 2.

How does Server 1 deliver the message?

This introduces a distributed communication problem.

A possible architecture looks like:

```text
                  ┌───────────────┐
                  │ Connection    │
                  │ Routing Layer │
                  └───────┬───────┘
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
          WS Server    WS Server    WS Server
              1            2            3
```

The system needs some way to know:

```text
User A → Connected to Server 1

User B → Connected to Server 3
```

Then events can be routed to the correct server.

This is one reason real-time systems naturally lead into messaging and event-driven architectures, which we will study in Module 7.

---

# 12. What Happens When the Client Is Slow?

Suppose the server can produce:

```text
10,000 messages per second
```

but the client can only process:

```text
100 messages per second
```

Now messages start accumulating.

```text
Server
   │
   ▼
[Message]
[Message]
[Message]
[Message]
[Message]
       ↓
Slow Client
```

Eventually:

- Memory can grow.
- Buffers can fill.
- Latency increases.
- The connection may need to be terminated.

So WebSockets introduce another important problem:

> What happens when producers are faster than consumers?

This idea will become extremely important when we study queues, streaming systems, and backpressure.

---

# 13. WebSocket vs HTTP

Let's compare the communication models.

| HTTP                                                    | WebSocket                                            |
| ------------------------------------------------------- | ---------------------------------------------------- |
| Request-response                                        | Bidirectional messaging                              |
| Client initiates each exchange                          | Either side can send                                 |
| Usually short-lived interaction                         | Long-lived connection                                |
| Stateless at the protocol/application interaction level | Connection maintains an active communication channel |
| Excellent for APIs and resource access                  | Excellent for real-time updates                      |
| Simple infrastructure                                   | Connection management required                       |

The important distinction is not:

> HTTP bad, WebSocket good.

Instead:

```text
Different communication problems
        ↓
Different communication models
```

---

# 14. WebSockets Are Not Automatically Better

Imagine a simple product API.

```text
GET /products/123
```

The client asks once.

The server responds once.

Then nothing else needs to happen.

A WebSocket would add unnecessary complexity.

You would need to:

- Establish a persistent connection.
- Manage connection lifecycle.
- Handle reconnection.
- Scale active connections.

For this problem, HTTP is simpler.

WebSockets are useful when the value comes from the connection **remaining open**.

---

# 15. Real-World Usage

WebSockets are commonly useful for systems such as:

### Chat

```text
User sends message

↓

Server

↓

Other user receives immediately
```

---

### Collaborative Editing

For example:

```text
User A changes document

↓

Server distributes change

↓

Other users see update
```

---

### Live Dashboards

```text
Metric changes

↓

Server pushes update

↓

Dashboard refreshes automatically
```

---

### Multiplayer Games

Players continuously exchange events.

```text
Player A moves

↓

Server

↓

Other players receive update
```

---

### Financial Data

```text
Price changes

↓

Server pushes latest value

↓

Connected clients update
```

---

# 16. Where WebSockets Help

WebSockets are useful when you need:

- Low-latency updates
- Frequent communication
- Server-to-client push
- Bidirectional communication
- Long-lived sessions
- Real-time interactions

---

# 17. Where WebSockets Don't Help

They are usually unnecessary for:

- Simple CRUD APIs
- Infrequent requests
- Static content
- One-time request-response interactions

For example:

```text
GET /profile
```

does not need a persistent connection.

WebSockets also introduce infrastructure complexity at scale.

---

# 18. Mental Model

## Telephone vs Postal Mail

Traditional HTTP is like postal mail.

```text
Send letter

↓

Wait for reply

↓

Conversation ends
```

Each exchange is independent.

WebSockets are like a telephone call.

```text
Call established

↓

Both people remain connected

↓

Either person can speak at any time
```

The telephone is more powerful for a live conversation.

But you would not make a phone call just to ask:

> "What is the price of this book?"

For a simple one-time interaction, a letter may be enough.

The communication model should match the problem.

---

# 19. Tradeoffs

## Advantages

- Server can push updates immediately.
- Full-duplex communication.
- Avoids repeated polling requests.
- Low latency for ongoing communication.
- Useful for interactive applications.
- Efficient when frequent messages are exchanged.

## Disadvantages

- Persistent connections consume resources.
- Scaling millions of active connections is more complex.
- Requires connection lifecycle management.
- Clients must handle reconnection.
- Load balancing and message routing become more complicated.
- Slow consumers can create buffering and memory problems.
- Not naturally useful for simple request-response APIs.

---

# 20. Common Interview Questions

## Why use WebSockets instead of HTTP polling?

Polling repeatedly sends requests even when nothing has changed.

WebSockets establish a persistent connection so the server can send updates when events actually occur.

This can reduce unnecessary requests and provide lower-latency updates.

---

## Are WebSockets faster than HTTP?

Not inherently.

For a one-time request:

```text
GET /product/123
```

HTTP may be simpler and perfectly efficient.

WebSockets become useful when communication is frequent or ongoing because they avoid repeatedly establishing independent request-response interactions.

---

## How do WebSockets scale horizontally?

This is more complex than scaling stateless HTTP servers because active connections are tied to specific servers.

The system needs a way to:

- Track where users are connected.
- Route messages to the correct server.
- Distribute events across WebSocket servers.

This often leads to shared messaging or event-distribution infrastructure.

---

## What happens when a WebSocket server crashes?

Clients connected to that server lose their connections.

Clients should detect the failure and reconnect.

The system may also need to restore application-level state after reconnection.

---

## Can a WebSocket connection remain open forever?

Conceptually it can remain open for a long time, but real infrastructure often imposes:

- Idle timeouts
- Load balancer limits
- Network interruptions
- Server restarts

Applications typically use heartbeats and reconnection logic to keep connections healthy.

---

## When would you not use WebSockets?

When communication is:

- Infrequent
- Simple
- Naturally request-response based

Using WebSockets for ordinary CRUD APIs usually adds complexity without meaningful benefit.

---

# 21. Before vs After Architecture

## Before — Polling

```text
                  Client
                    │
          Every few seconds
                    │
                    ▼
                HTTP Server
                    │
                    ▼
              Any updates?

               │       │
              No      Yes
               │       │
               ▼       ▼
            Wait    Send Data
```

The client keeps asking.

---

## After — WebSocket

```text
              Client
                 ║
                 ║ Persistent
                 ║ Connection
                 ║
              Server


Event occurs
      │
      ▼
Server immediately sends update
      │
      ▼
Client
```

The client no longer needs to repeatedly ask.

---

# 22. Connections

WebSockets solve a major limitation of traditional request-response communication.

They allow:

```text
Server
   │
   ▼
Push information whenever something happens
```

But WebSockets are **not the only way** to solve the server-push problem.

What if communication only needs to flow in one direction?

For example:

```text
Server ───────► Client
```

A live news feed.

A notification stream.

A dashboard receiving updates.

The client may not need to continuously send messages back through the same persistent channel.

Using a full-duplex WebSocket may be unnecessary.

This leads to another question:

> **Can we keep an HTTP request open and wait until the server actually has something to say?**

That is the idea behind:

# Chapter 5 — Long Polling

---

# 23. Key Takeaways

- Traditional HTTP follows a client-initiated request-response model.
- Polling can simulate real-time communication but creates unnecessary requests or increased latency.
- WebSockets establish a persistent connection.
- Both the client and server can independently send messages.
- This is called full-duplex communication.
- WebSockets are well suited to chat, collaboration, live dashboards, multiplayer applications, and other real-time systems.
- Persistent connections introduce new engineering problems: connection tracking, reconnection, slow consumers, resource usage, and distributed routing.
- WebSockets are not a replacement for HTTP; they solve a different communication problem.
- The next step is a lighter-weight approach for server-driven updates: **Long Polling**.
