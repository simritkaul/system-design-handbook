# Chapter 5 — Long Polling

## Goal

Understand how we can simulate server-driven updates using ordinary HTTP, why traditional polling is inefficient, and why long polling emerged as an intermediate solution before persistent real-time communication approaches such as WebSockets and SSE.

---

# 1. The Problem

In the previous chapter, we built a chat application.

Bob wants to receive Alice's messages as soon as they arrive.

The simplest approach was polling.

Bob's application repeatedly asks:

```text
Any new messages?
```

The server replies:

```text
No.
```

Then Bob asks again.

```text
Any new messages?
```

Again:

```text
No.
```

Eventually:

```text
Any new messages?
```

And finally:

```text
Yes! Here is Alice's message.
```

The architecture looks like this:

```text
Client                         Server

Any updates? ────────────────►

             ◄──────────────── No


Any updates? ────────────────►

             ◄──────────────── No


Any updates? ────────────────►

             ◄──────────────── Yes!
```

The obvious problem is that most requests may be useless.

The client keeps asking even when nothing has happened.

But there is another problem.

Suppose Bob polls every 10 seconds.

Alice sends a message immediately after Bob's last request.

Bob might not see it for almost 10 seconds.

If we poll every second, latency improves.

But now we generate ten times more requests.

So we face a tradeoff.

```text
Poll Frequently
      │
      ├── Lower latency
      └── More unnecessary requests


Poll Infrequently
      │
      ├── Fewer requests
      └── Higher latency
```

Can we avoid both problems?

---

# 2. Why Existing Solutions Fail

Traditional polling works like this:

```text
Request
   │
   ▼
"Any updates?"
   │
   ▼
Immediate Response
```

The server must answer immediately, even when it has nothing useful to say.

This creates the waste.

```text
Client: Any updates?
Server: No

Client: Any updates?
Server: No

Client: Any updates?
Server: No
```

The server is repeatedly saying:

> "Nothing happened."

What if instead, when the client asks:

> "Any updates?"

the server says:

> "I'll wait until something happens."

The request remains open.

That is the core idea behind long polling.

---

# 3. The Big Idea

> **The client sends an HTTP request, but the server delays its response until new data becomes available or a timeout occurs.**

Instead of this:

```text
Request → Immediate "No" → Wait → New Request
```

we do:

```text
Request → Wait → Event happens → Response
```

The client does not keep repeatedly asking while nothing has changed.

---

# 4. Detailed Explanation

Let's return to our chat application.

Bob sends:

```text
GET /messages
```

But suppose there are no new messages.

With normal polling:

```text
Bob
 │
 │ Any messages?
 ▼
Server
 │
 ▼
No messages
 │
 ▼
Respond immediately
```

With long polling:

```text
Bob
 │
 │ Any messages?
 ▼
Server
 │
 │
 │ Wait...
 │
 │ Wait...
 │
 │ Wait...
 │
```

The HTTP request remains open.

Then Alice sends Bob a message.

```text
Alice sends message
        │
        ▼
Server now has new data
        │
        ▼
Respond to Bob's waiting request
```

Bob receives the message immediately.

```text
Bob
 ▲
 │ New message!
 │
Server
```

Then Bob's application immediately starts another long-poll request.

```text
Receive response
       │
       ▼
Open new request
       │
       ▼
Wait again
```

This cycle continues.

---

# 5. The Long Polling Lifecycle

The complete flow looks like this:

```text
Client
   │
   │ Request: Any updates?
   ▼
Server
   │
   │ No updates yet
   │
   │ Hold request open
   │
   │
   │ Event occurs
   ▼
Send response
   │
   ▼
Client receives update
   │
   ▼
Client immediately sends another request
```

Then:

```text
Wait
 ↓
Event
 ↓
Response
 ↓
New request
 ↓
Wait again
```

This creates something that _feels_ like server push.

But technically, the client is still initiating every request.

The server cannot spontaneously create a connection to the client.

Instead, the server takes advantage of a request that is already waiting.

---

# 6. Long Polling vs Traditional Polling

Let's compare them.

## Traditional Polling

```text
Client                         Server

Request ──────────────────────►

        ◄────────────────────── No

Wait

Request ──────────────────────►

        ◄────────────────────── No

Wait

Request ──────────────────────►

        ◄────────────────────── Yes!
```

The server responds immediately every time.

---

## Long Polling

```text
Client                         Server

Request ──────────────────────►

                                Wait...

                                Wait...

                                Event!

        ◄────────────────────── New Data


Request ──────────────────────►

                                Wait...
```

The request itself becomes the waiting mechanism.

---

# 7. Why Is Long Polling More Efficient?

Suppose we have one million users.

With polling every second:

```text
1,000,000 users

×

1 request/second

=

1,000,000 requests per second
```

Even when there are no events.

With long polling, users generally send a new request only when:

- An event arrives.
- The request times out.
- The connection fails.

If the system is mostly idle, this can significantly reduce useless requests.

However, the server now has another problem.

It may need to maintain a large number of waiting requests.

---

# 8. The Cost of Waiting Requests

Imagine one million clients all doing this:

```text
Client 1  ── Waiting
Client 2  ── Waiting
Client 3  ── Waiting
...
Client 1,000,000 ── Waiting
```

The server cannot simply forget about them.

It must maintain enough state to eventually respond when:

```text
Event occurs
     │
     ▼
Find waiting client
     │
     ▼
Send response
```

This introduces resource concerns.

Depending on the implementation and infrastructure, large numbers of open requests can consume:

- Connections
- Memory
- File descriptors
- Worker or event-loop capacity
- Load balancer resources

So long polling reduces **request frequency**, but it increases the importance of efficiently handling **long-lived requests**.

---

# 9. What Happens if Nothing Happens?

A request cannot necessarily remain open forever.

Suppose Bob waits:

```text
GET /messages
```

but Alice sends nothing for several minutes.

Eventually, one of the following may happen:

- Application timeout
- Server timeout
- Load balancer timeout
- Proxy timeout
- Network interruption

So long polling usually includes a timeout.

```text
Client
   │
   │ Request
   ▼
Server
   │
   ├── Event occurs
   │        │
   │        ▼
   │     Respond
   │
   └── Timeout
            │
            ▼
         Respond
```

If the request times out without new data, the client opens another request.

```text
Timeout
   │
   ▼
Client receives empty response
   │
   ▼
Client immediately starts another long poll
```

---

# 10. A Chat Application Example

Let's build the complete flow.

Bob opens the application.

```text
Bob
 │
 │ GET /messages?after=100
 ▼
Chat Server
```

The server determines:

> Bob has already received messages up to message 100.

There are currently no newer messages.

So:

```text
Bob's request
      │
      ▼
Waiting
```

Then Alice sends:

```text
Message 101
```

The system processes it.

```text
Alice
   │
   ▼
Chat Server
   │
   ▼
Message stored
   │
   ▼
Bob has waiting request
   │
   ▼
Respond with Message 101
```

Bob receives:

```text
Message 101
```

Immediately afterward:

```text
Bob
 │
 │ GET /messages?after=101
 ▼
Chat Server
```

And waits again.

The communication appears almost real-time.

---

# 11. Long Polling Does Not Create a Permanent Connection

This distinction is important.

Long polling:

```text
Request
   │
   ▼
Wait
   │
   ▼
Response
   │
   ▼
Connection/request completes
```

Then the client creates another request.

WebSocket:

```text
Connect
   │
   ▼
Persistent connection
   │
   ├── Message
   ├── Message
   ├── Message
   └── Message
```

The same connection is used for multiple messages.

So while long polling can provide real-time-like behavior, it still repeatedly recreates the request cycle.

---

# 12. Long Polling vs WebSockets

| Long Polling                                     | WebSockets                                                   |
| ------------------------------------------------ | ------------------------------------------------------------ |
| Built on normal HTTP requests                    | Uses a persistent WebSocket connection                       |
| Client starts a new request after every response | One connection carries many messages                         |
| Server can delay a response                      | Server can send messages whenever needed                     |
| Mostly useful for server-to-client updates       | Supports full bidirectional communication                    |
| Easier to fit into existing HTTP infrastructure  | Requires persistent connection management                    |
| More request overhead over time                  | Lower per-message request overhead for ongoing communication |

Neither is automatically better.

---

## When Long Polling Can Be Simpler

Suppose updates are relatively infrequent.

For example:

```text
Notification arrives every few minutes.
```

Maintaining a dedicated WebSocket connection may not provide enough benefit to justify additional complexity.

Long polling can work with familiar HTTP infrastructure.

---

## When WebSockets Are Better

Suppose a collaborative application generates:

```text
Thousands of updates per minute.
```

Repeatedly finishing one long-poll request and opening another becomes inefficient.

A persistent bidirectional channel is more natural.

```text
Many messages
     │
     ▼
WebSocket connection
     │
     ▼
Continuous communication
```

---

# 13. The Scaling Problem

Suppose we have multiple application servers.

```text
                   Load Balancer
                  /      |      \
                 ▼       ▼       ▼
              Server1 Server2 Server3
```

Bob's long-poll request reaches Server 1.

```text
Bob
 │
 ▼
Server 1

Request waiting...
```

Now Alice sends a message.

But her request reaches Server 3.

```text
Alice
  │
  ▼
Server 3
```

Server 3 processes the message.

How does Server 3 notify Server 1?

```text
Server 3
   │
   │ New message for Bob
   ▼
???
   │
   ▼
Server 1
```

Just like WebSockets, long polling eventually encounters a distributed communication problem.

Servers need a way to share events.

Conceptually:

```text
                Event Distribution
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Server 1      Server 2      Server 3
          │                           │
          ▼                           ▼
         Bob                        Alice
```

This is another natural bridge toward the messaging and event-driven concepts we'll study later.

---

# 14. Where Long Polling Helps

Long polling is useful when:

- You need near-real-time server-to-client updates.
- WebSockets are unavailable or unnecessary.
- Existing infrastructure is heavily based around HTTP.
- Updates are relatively infrequent.
- Simplicity is more important than maximum efficiency.

Possible examples:

- Notifications
- Chat systems with moderate traffic
- Job status updates
- Legacy environments
- Compatibility with infrastructure that does not support persistent WebSocket connections

---

# 15. Where Long Polling Doesn't Help

Long polling becomes less attractive when:

### Updates are extremely frequent

Repeated request creation adds overhead.

### Bidirectional communication is continuous

WebSockets are usually more natural.

### Millions of clients must hold waiting requests

The infrastructure must efficiently manage a huge number of long-lived HTTP requests.

### Very low latency is required at high message rates

A persistent communication channel can be more efficient.

---

# 16. Mental Model

## Calling a Reception Desk

Traditional polling is like calling a hotel reception desk every five minutes.

```text
"Any package for me?"

"No."

Five minutes later:

"Any package?"

"No."
```

Long polling is like calling the receptionist and saying:

> "I'm waiting. Don't hang up. Tell me when my package arrives."

The receptionist keeps the call open.

Eventually:

```text
"Your package has arrived."
```

The call ends.

You immediately call again:

> "Let me know when the next one arrives."

WebSockets are more like having a permanent phone line open between you and the receptionist.

Either side can speak whenever necessary.

---

# 17. Tradeoffs

## Advantages

- Reduces unnecessary requests compared with frequent polling.
- Can provide near-real-time updates.
- Uses familiar HTTP infrastructure.
- Does not require a separate full-duplex communication model.
- Useful when updates are intermittent.

## Disadvantages

- Requests still need to be recreated after every response.
- Large numbers of waiting requests consume resources.
- Timeout and reconnection logic are required.
- Not naturally designed for frequent bidirectional communication.
- Horizontal scaling requires coordination between servers.
- Infrastructure timeouts can complicate implementation.

---

# 18. Common Interview Questions

## What is long polling?

Long polling is a technique where the client sends an HTTP request and the server delays the response until new data becomes available or a timeout occurs.

After receiving the response, the client sends another request and waits again.

---

## How is long polling different from polling?

With normal polling:

```text
Client asks

↓

Server responds immediately
```

With long polling:

```text
Client asks

↓

Server waits until data is available

↓

Server responds
```

The goal is to avoid repeatedly receiving useless "nothing changed" responses.

---

## How is long polling different from WebSockets?

Long polling repeatedly creates HTTP requests.

WebSockets establish one persistent connection.

Long polling is generally better suited for intermittent server-to-client updates, while WebSockets are better suited to frequent or bidirectional communication.

---

## Does long polling eliminate all unnecessary requests?

No.

Requests still occur when:

- The previous request completes.
- A timeout happens.
- A network failure requires reconnection.

It reduces unnecessary polling but does not eliminate request lifecycle overhead.

---

## What happens when a long-poll request times out?

The server returns a response indicating that no update arrived during the waiting period.

The client then typically immediately starts another long-poll request.

---

# 19. Before vs After Architecture

## Before — Polling

```text
Client                         Server

Any updates? ────────────────►

             ◄──────────────── Nothing

Wait

Any updates? ────────────────►

             ◄──────────────── Nothing

Wait

Any updates? ────────────────►

             ◄──────────────── New data
```

The client repeatedly checks.

---

## After — Long Polling

```text
Client                         Server

Any updates? ────────────────►

                                Wait...

                                Wait...

                                New event!

             ◄──────────────── New data


Any updates? ────────────────►

                                Wait...
```

The request waits until it has something useful to return.

---

# 20. Connections

Long polling improves traditional polling by changing:

```text
Ask repeatedly

↓

Ask once and wait
```

But there is still an inefficiency.

Every time the server sends an update:

```text
Response completes

↓

Client creates another request

↓

Server waits again
```

Now imagine a live dashboard.

The server produces:

```text
Update 1

Update 2

Update 3

Update 4
```

The client only wants to receive a continuous stream of updates.

The communication is primarily:

```text
Server ─────────► Client
```

The client does not need to send continuous messages back.

A full-duplex WebSocket may be more than we need.

Long polling repeatedly creates new requests.

So the next natural question is:

> **Can we establish one long-lived HTTP connection and continuously stream updates from the server to the client?**

That leads us to:

# Chapter 6 — Server-Sent Events (SSE)

---

# 21. Key Takeaways

- Traditional polling repeatedly asks the server for updates.
- Frequent polling reduces latency but creates unnecessary traffic.
- Long polling keeps an HTTP request open until data becomes available or a timeout occurs.
- After receiving a response, the client immediately opens another request.
- Long polling provides near-real-time behavior without requiring WebSockets.
- It is generally useful for intermittent server-to-client updates.
- It still has request lifecycle overhead and can create scaling challenges with many waiting connections.
- It is not ideal for frequent bidirectional communication.
- The next evolution is keeping one HTTP connection open and streaming multiple server-to-client updates through it: **Server-Sent Events**.
