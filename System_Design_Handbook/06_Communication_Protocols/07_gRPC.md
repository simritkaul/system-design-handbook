# Chapter 7 — gRPC

## Goal

Understand why communication between backend services often has different requirements from communication between browsers and servers, and how gRPC provides a more efficient, strongly defined, and streaming-friendly communication model.

---

# 1. The Problem

So far, most of our communication models have been easy to visualize from the perspective of a user.

For example:

```text
Browser
   │
   ▼
API Server
```

The browser asks:

```text
GET /products/123
```

The server responds:

```json
{
  "id": 123,
  "name": "Laptop",
  "price": 50000
}
```

This works very well.

REST is simple.

JSON is human-readable.

HTTP is universally supported.

But now imagine a large backend system.

```text
                    User
                      │
                      ▼
                 API Gateway
                      │
                      ▼
                 Order Service
                  /     |      \
                 ▼      ▼       ▼
           Payment   Inventory  User
           Service    Service   Service
              │
              ▼
        Fraud Detection
           Service
```

One user request may trigger many internal service-to-service calls.

At this point, the communication requirements change.

We are no longer primarily asking:

> Can a browser easily call this API?

Instead, we ask:

> How efficiently can machines communicate with other machines?

Suppose the Order Service needs to call the Payment Service millions of times per day.

With a REST-style approach:

```text
Order Service
      │
      │ HTTP + JSON
      ▼
Payment Service
```

This is perfectly valid.

But eventually we may start asking:

- Do we need JSON to be human-readable?
- Is text serialization the most efficient format?
- How do we ensure both services agree on the request structure?
- What happens when an API contract changes?
- Can we generate client code automatically?
- Can services stream data efficiently?
- Can we reduce communication overhead for high-volume internal calls?

These questions lead to gRPC.

---

# 2. Why Existing Solutions Become Insufficient

REST was designed around a very natural model:

```text
Resource
   │
   ▼
HTTP Request
   │
   ▼
JSON Response
```

For example:

```text
GET /users/123
```

This is easy for humans to understand.

But imagine this internal call:

```text
GetUser(userId)
```

The Order Service already knows:

- It wants a user.
- It has the user's ID.
- Another service knows how to retrieve the user.

The communication problem is fundamentally:

```text
Service A
    │
    │ "Call this operation"
    ▼
Service B
```

This begins to look less like navigating resources and more like calling a function.

Inside a single application, we might write:

```text
userService.getUser(123)
```

But now `userService` lives on another machine.

Could we make remote communication feel conceptually like calling a function?

That is the central idea behind **Remote Procedure Call**, or RPC.

---

# 3. The Big Idea

> **Instead of thinking primarily in terms of HTTP resources, a service exposes operations that other services can call remotely.**

Conceptually:

```text
Order Service

getUser(123)
      │
      │
      ▼

User Service
```

The caller invokes an operation.

The remote service performs the work.

The result comes back.

```text
getUser(123)

       ↓

User
```

gRPC is a modern framework for building this kind of remote communication.

---

# 4. First, What Is RPC?

Before understanding gRPC, we need to understand the broader idea.

Imagine two functions in the same application.

```text
OrderService
      │
      │ getUser(123)
      ▼
UserService
```

Calling the function feels simple.

But what if the User Service runs here?

```text
Machine A

Order Service
```

while the User Service runs here?

```text
Machine B

User Service
```

Now:

```text
getUser(123)
```

is no longer a normal function call.

The system must:

1. Create a request.
2. Send it across the network.
3. Receive it on another machine.
4. Execute the operation.
5. Send the result back.

Conceptually:

```text
Order Service
     │
     │ getUser(123)
     ▼
Network
     │
     ▼
User Service
     │
     │ Execute
     ▼
Network
     │
     ▼
Order Service
```

RPC hides much of this communication complexity behind a function-like abstraction.

The developer thinks:

```text
getUser(123)
```

while the underlying system handles:

```text
Serialize
   ↓
Send
   ↓
Receive
   ↓
Deserialize
   ↓
Execute
   ↓
Serialize response
   ↓
Send back
```

That is the fundamental RPC idea.

---

# 5. Why Isn't a Remote Function Call Really Like a Local Function Call?

This is extremely important.

A local function call is usually:

```text
Fast
Predictable
Same process
Same memory
```

A remote call is:

```text
Network involved
      ↓
Latency
      ↓
Possible failure
      ↓
Timeout
      ↓
Partial failure
```

Consider:

```text
paymentService.charge()
```

If this were a local function, the result would generally be immediate.

But remotely:

```text
Order Service
      │
      │ charge()
      ▼
      Network
      │
      ✕ Connection fails
```

Now what?

Did the Payment Service:

- Never receive the request?
- Receive it but not process it?
- Process the payment but fail to send the response?

This is a distributed systems problem.

So gRPC makes remote communication _feel_ like calling an operation, but good engineers must never forget:

> A remote call is still a network call.

This distinction becomes crucial when designing reliable distributed systems.

---

# 6. The gRPC Model

With gRPC, a service defines the operations it provides.

Conceptually:

```text
User Service

GetUser
CreateUser
UpdateUser
DeleteUser
```

Another service can call them.

```text
Order Service
      │
      ├── GetUser()
      │
      ├── CreateUser()
      │
      └── UpdateUser()
      ▼
User Service
```

The services agree on a formal contract.

For example:

```text
GetUser

Input:
    userId

Output:
    User
```

This agreement is important.

The caller knows exactly:

- What operation exists.
- What input it accepts.
- What output it returns.

This brings us to one of gRPC's most important ideas:

> **Contract-first communication.**

---

# 7. Contract-First Communication

Imagine two teams.

Team A owns the Order Service.

Team B owns the User Service.

The Order Service expects:

```text
GET /users/123
```

and expects a response like:

```json
{
  "id": 123,
  "name": "Alice"
}
```

But suppose Team B changes the response.

```json
{
  "user_id": 123,
  "full_name": "Alice"
}
```

Now Team A may break.

With any communication system, both sides need an agreement.

gRPC makes that agreement explicit.

Conceptually:

```text
                Shared Contract
                     │
         ┌───────────┴───────────┐
         ▼                       ▼
   Order Service             User Service
```

The contract defines:

```text
Service
    │
    ├── Operations
    │
    ├── Request structure
    │
    └── Response structure
```

From this contract, tools can generate code for both sides.

Conceptually:

```text
Service Contract
       │
       ├──────────────► Client Code
       │
       └──────────────► Server Code
```

This reduces the amount of manual communication code developers need to write.

---

# 8. Protocol Buffers: The Data Contract

gRPC commonly uses **Protocol Buffers**, often called Protobuf.

Before discussing how they work, understand the problem.

When two services communicate, they need to agree on data.

For example:

```text
User

id
name
email
```

JSON expresses this as text:

```json
{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com"
}
```

This is easy for humans to read.

But machines don't necessarily need human-readable data.

They need:

```text
Structured
Compact
Predictable
```

Protobuf defines the structure first.

Conceptually:

```text
User

id    → number
name  → text
email → text
```

Then that structured data can be serialized into a compact binary format for transmission.

So the flow becomes:

```text
Structured Object
        │
        ▼
Serialization
        │
        ▼
Compact Binary Data
        │
        ▼
Network
        │
        ▼
Deserialization
        │
        ▼
Structured Object
```

The important engineering idea is not:

> Binary is always better than JSON.

Instead:

> If machines communicate with each other at high volume, a compact, strongly defined representation can reduce overhead.

---

# 9. JSON vs Protobuf

Consider:

```text
{
  "id": 123,
  "name": "Alice"
}
```

JSON contains:

- Field names
- Text representation
- Syntax characters

This is useful for readability.

Protobuf uses an agreed-upon schema, so both sides already know what fields mean.

Conceptually:

```text
Schema known by both sides
          │
          ▼
Transmit values efficiently
```

The tradeoff is:

```text
JSON
 │
 ├── Human-readable
 ├── Easy to debug manually
 └── Flexible


Protobuf
 │
 ├── Compact
 ├── Strongly structured
 ├── Efficient serialization
 └── Requires schema/tooling
```

Again, neither is universally better.

The context matters.

---

# 10. Unary Communication

The simplest gRPC communication pattern is similar to a normal function call.

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

For example:

```text
GetUser(123)

        ↓

User
```

This is called a **unary call**.

One request.

One response.

It resembles traditional HTTP request-response communication.

---

# 11. But gRPC Can Also Stream

This is where gRPC becomes especially interesting.

Not every communication pattern is:

```text
One request
     ↓
One response
```

Sometimes one side needs to continuously send data.

gRPC supports several communication models.

---

## Client Streaming

The client sends multiple messages.

```text
Client
   │
   ├── Message 1
   ├── Message 2
   ├── Message 3
   ▼
Server
```

Eventually:

```text
Server
   │
   ▼
One response
```

For example, imagine uploading many records for processing.

---

## Server Streaming

The client sends one request.

```text
Client
   │
   │ Request
   ▼
Server
```

Then the server sends multiple responses.

```text
Server
   │
   ├── Response 1
   ├── Response 2
   ├── Response 3
   ▼
Client
```

For example:

```text
Give me live processing results.
```

---

## Bidirectional Streaming

Both sides continuously send messages.

```text
Client ◄──────────────► Server

Message 1 ───────────►

             ◄──────── Message A

Message 2 ───────────►

             ◄──────── Message B
```

This begins to resemble the communication model we saw with WebSockets.

But the primary use case is often different.

WebSockets are commonly associated with interactive client-server applications.

gRPC streaming is especially useful for structured service-to-service communication.

---

# 12. Why HTTP/2 Matters

Earlier, we learned about HTTP Keep-Alive.

The goal was:

```text
Avoid creating a new connection for every request.
```

HTTP/2 goes further.

One of its important ideas is **multiplexing**.

Imagine several requests.

With an older model, communication could conceptually become constrained by how requests and responses share connections.

HTTP/2 allows multiple streams of communication over the same underlying connection.

Conceptually:

```text
One Connection

├── Stream 1 → Request A
├── Stream 2 → Request B
├── Stream 3 → Request C
└── Stream 4 → Request D
```

gRPC uses HTTP/2 as its transport foundation in its standard model.

This gives it useful capabilities for:

- Multiple concurrent calls.
- Streaming.
- Efficient connection reuse.
- Structured communication between services.

We will study HTTP/2 itself in detail in a later chapter.

For now, the important connection is:

> gRPC is not simply "REST with binary data."

It combines several ideas:

```text
Remote Procedure Calls
        +
Strong Contracts
        +
Efficient Serialization
        +
HTTP/2
        +
Streaming
```

---

# 13. Before vs After Architecture

## Before — Generic HTTP + JSON Communication

```text
Order Service
      │
      │ HTTP Request
      │ JSON
      ▼
User Service
      │
      │ JSON Response
      ▼
Order Service
```

The services communicate through URLs and data formats.

---

## After — Contract-Based RPC

```text
          Shared Service Contract
                   │
          ┌────────┴────────┐
          ▼                 ▼
    Order Service      User Service
          │                 ▲
          │   GetUser()     │
          └─────────────────┘
```

The communication is defined as operations with explicit input and output types.

---

# 14. Where gRPC Helps

gRPC is particularly useful when:

### Services communicate frequently

```text
Service A
   │
   ├──► Service B
   ├──► Service C
   └──► Service D
```

Reducing serialization and communication overhead can matter.

---

### You control both client and server

In an internal microservice environment:

```text
Your Team's Service
        │
        ▼
Another Internal Service
```

Both sides can adopt the same contract and tooling.

---

### Strong contracts are valuable

Large organizations may have:

```text
100+ services
```

Explicit interfaces make service communication easier to manage.

---

### Streaming is required

gRPC provides structured support for:

- Client streaming
- Server streaming
- Bidirectional streaming

---

# 15. Where gRPC Doesn't Help

gRPC is not automatically the best choice.

## Public browser APIs

Browsers historically integrate naturally with HTTP and REST-style APIs.

A simple public API like:

```text
GET /weather?city=Delhi
```

is easy to understand and consume.

REST may be the better developer experience.

---

## Human-readable debugging is important

With JSON:

```json
{
  "status": "success"
}
```

you can easily inspect the data.

Binary protocols require appropriate tooling.

---

## Simple APIs

For a small system:

```text
Client
   │
   ▼
Simple API
```

introducing schema generation and additional tooling may be unnecessary complexity.

---

# 16. Real-World Usage

gRPC fits naturally into large distributed architectures.

For example:

```text
                    API Gateway
                         │
              HTTP / JSON for clients
                         │
                         ▼
                  Order Service
                    /        \
                   ▼          ▼
             User Service  Payment Service
                   │          │
                   └────┬─────┘
                        ▼
                  gRPC Communication
```

This is a common architectural pattern conceptually:

```text
External World
      │
      ▼
Simple, widely compatible APIs

Internal System
      │
      ▼
Efficient, strongly defined service communication
```

The important point is not that every company must use this pattern.

It is that different boundaries can have different requirements.

---

# 17. Mental Model

## Calling a Specialist Through a Receptionist

Imagine you're in a large company.

You need a legal opinion.

With REST-like communication, you might think:

```text
Go to Department X
Send Form Y
Receive Document Z
```

With RPC, you think:

```text
Call LegalDepartment.getAdvice()
```

The underlying process may still involve:

- People
- Phone lines
- Waiting
- Failures

But the interface is expressed as:

> Call this operation with these inputs.

gRPC takes this further by ensuring everyone agrees in advance on:

```text
What can be called
What inputs are required
What outputs will return
```

It is like having a standardized internal company directory with precisely defined forms for every department.

---

# 18. Tradeoffs

## Advantages

- Strongly defined service contracts.
- Efficient binary serialization.
- Code generation can reduce boilerplate.
- Supports unary and streaming communication.
- Well suited to internal service-to-service communication.
- HTTP/2 enables efficient multiplexed communication.

## Disadvantages

- More tooling and schema management.
- Less human-readable than JSON.
- Harder to manually test without appropriate tools.
- Can be unnecessary for simple systems.
- Browser and public API support may require additional considerations.
- A remote procedure call can misleadingly feel like a local function call, even though network failures and latency still exist.

---

# 19. Common Interview Questions

## What problem does gRPC solve?

gRPC provides a structured and efficient way for distributed services to communicate using explicitly defined contracts, efficient serialization, and support for streaming.

It is especially useful when machine-to-machine communication is frequent.

---

## How is gRPC different from REST?

REST commonly models APIs around resources.

```text
/users/123
/orders/456
```

gRPC models APIs around operations.

```text
GetUser()
CreateOrder()
ChargePayment()
```

REST commonly uses human-readable formats such as JSON.

gRPC commonly uses Protobuf for compact structured serialization.

The choice depends on the communication boundary and requirements.

---

## Is gRPC always faster than REST?

No.

Performance depends on:

- Payload size.
- Serialization cost.
- Network latency.
- Server processing.
- Connection reuse.
- Infrastructure.

For high-volume internal communication, gRPC's compact serialization and HTTP/2 capabilities can be advantageous.

But for a simple API, the performance difference may not justify the additional complexity.

---

## What is Protocol Buffers?

Protocol Buffers are a schema-based mechanism for defining structured data and serializing it into a compact binary representation.

Both sides agree on the schema.

---

## Does gRPC support streaming?

Yes.

The major communication patterns are:

```text
Unary
Client Streaming
Server Streaming
Bidirectional Streaming
```

---

## Why shouldn't developers treat a remote gRPC call exactly like a local function call?

Because the network introduces:

- Latency
- Timeouts
- Partial failures
- Retries
- Uncertain outcomes

A function call inside the same process and a call to another machine have fundamentally different failure characteristics.

---

# 20. Connections

We now have several communication models.

```text
REST
 │
 └── Resource-oriented APIs
```

```text
GraphQL
 │
 └── Client requests exactly the data it needs
```

```text
WebSockets
 │
 └── Persistent bidirectional communication
```

```text
Long Polling
 │
 └── Client waits for server updates
```

```text
SSE
 │
 └── Persistent server-to-client event stream
```

```text
gRPC
 │
 └── Strongly defined, efficient service-to-service communication
```

But there is a missing piece.

Earlier, we learned that HTTP/1.1 improved connection reuse with:

```text
Keep-Alive
```

Now gRPC introduced another idea:

```text
One connection

↓

Multiple independent streams
```

How is this possible?

Why was HTTP/2 created?

What limitations of HTTP/1.1 made multiplexing necessary?

And what does this mean for ordinary REST APIs, browsers, images, JavaScript, and web applications?

That naturally leads to:

# Chapter 8 — HTTP/2

---

# 21. Key Takeaways

- Backend service communication often has different requirements from browser-facing APIs.
- RPC models communication as calling remote operations.
- gRPC is built around explicit service contracts.
- Protocol Buffers provide strongly structured, compact serialization.
- gRPC supports unary, client-streaming, server-streaming, and bidirectional-streaming communication.
- gRPC commonly uses HTTP/2, enabling efficient multiplexed streams over a connection.
- gRPC is particularly useful for frequent internal service-to-service communication.
- It is not automatically better than REST; public APIs and simple systems may benefit more from HTTP and JSON.
- Most importantly: **a remote call is never truly just a local function call**. Network latency and partial failures still exist.
- The next chapter explains the protocol capability that makes modern multiplexed communication possible: **HTTP/2**.
