# Chapter 2 — Load Balancers

---

# Goal

Understand how incoming requests are distributed across multiple servers so that no single server becomes overloaded.

---

# The Problem

In the previous chapter, we learned that DNS helps users find the IP address of a service.

Suppose DNS resolves:

```text
www.netflix.com
        ↓
203.0.113.10
```

Your browser now knows where to send the request.

Everything seems perfect.

But let's look behind the scenes.

Imagine Netflix has only one application server.

```text
                 Users
                   │
                   │
            5 Million Requests
                   │
                   ▼
           +----------------+
           | App Server #1  |
           +----------------+
```

Every request from every user reaches this one machine.

Initially, everything works fine.

But Netflix suddenly releases the final season of a popular show.

Within minutes:

- Millions of users open the app.
- Millions of videos are requested.
- Millions of API calls arrive.

That single server now has to process every request.

Soon:

- CPU reaches 100%.
- Memory fills up.
- Network bandwidth becomes saturated.
- Response times increase.
- Requests begin timing out.
- Eventually the server crashes.

Now nobody can watch Netflix.

---

The obvious solution is:

> "Let's add another server."

So we build this:

```text
           Users
             │
             ▼
      +--------------+
      | Server #1    |
      +--------------+

      +--------------+
      | Server #2    |
      +--------------+
```

But now another question appears.

How do users know which server to contact?

Should half the users type:

```text
netflix-server-1.com
```

and the other half type:

```text
netflix-server-2.com
```

Of course not.

Users should always visit:

```text
www.netflix.com
```

The complexity should remain hidden.

---

# Why Existing Solutions Fail

Could DNS solve this?

Maybe DNS returns multiple IP addresses.

Example:

```text
www.netflix.com

↓

10.0.0.1

10.0.0.2

10.0.0.3
```

This does happen.

However, DNS has limitations.

DNS doesn't know:

- Which server is overloaded.
- Which server just crashed.
- Which server is under maintenance.
- Which server is geographically closest in real time.
- Which server has the least CPU usage.

DNS answers are also cached.

If a server crashes right after its IP is cached, users may continue trying to reach it until the cache expires.

DNS is great for finding a service.

It is not designed to intelligently distribute live traffic.

We need something smarter.

---

# The Big Idea

A **Load Balancer** sits in front of multiple servers and decides which server should handle each incoming request.

Instead of users choosing a server, the load balancer chooses for them.

```text
Users
   │
   ▼
Load Balancer
   │
 ┌─┼───────────┐
 ▼ ▼           ▼
S1 S2         S3
```

To users, the service still looks like one website.

Behind the scenes, requests are distributed across many machines.

---

# Before Load Balancing

```text
              Users
                 │
                 ▼
          +---------------+
          | App Server    |
          +---------------+
```

Problems:

- Single point of failure
- Limited CPU
- Limited RAM
- Limited bandwidth
- Cannot scale

---

# After Load Balancing

```text
                 Users
                    │
                    ▼
          +------------------+
          | Load Balancer    |
          +------------------+
           │       │       │
           ▼       ▼       ▼
      +------+ +------+ +------+
      | App1 | | App2 | | App3 |
      +------+ +------+ +------+
```

Now traffic is spread across multiple machines.

---

# A Simple Restaurant Analogy

Imagine a restaurant with one waiter.

Every customer must speak to that waiter.

```text
Customers

↓

Waiter

↓

Kitchen
```

As more customers arrive:

- Orders pile up.
- People wait longer.
- Mistakes increase.

Now imagine hiring a host.

```text
Customers

↓

Host

↓

Waiter A

Waiter B

Waiter C
```

The host decides who serves each customer.

Customers don't care which waiter they get.

They simply want good service.

The host is the load balancer.

---

# How a Load Balancer Works

A user sends a request.

```text
GET /movies
```

The request first reaches the load balancer.

The load balancer chooses a backend server.

```text
User

↓

Load Balancer

↓

Server 2
```

Server 2 processes the request.

The response travels back through the load balancer.

```text
Server 2

↓

Load Balancer

↓

User
```

From the user's perspective, they are only communicating with Netflix.

They never know which server handled their request.

---

# Why Not Let Every Server Handle Requests Directly?

Suppose users randomly connect to servers.

```text
User A → Server 1

User B → Server 1

User C → Server 1

User D → Server 2

User E → Server 3
```

Server 1 may become overloaded while the others remain mostly idle.

A load balancer keeps work evenly distributed.

---

# Health Checks

One of the biggest advantages of a load balancer is that it continuously checks whether backend servers are healthy.

Suppose we have three servers.

```text
App1

App2

App3
```

Every few seconds, the load balancer sends a small request.

```text
"Are you alive?"
```

If App2 responds:

```text
200 OK
```

Everything is fine.

If App2 stops responding:

```text
Timeout
```

The load balancer marks it as unhealthy.

Traffic immediately stops going to App2.

```text
                 Users
                    │
                    ▼
             Load Balancer
              │          │
              ▼          ▼
            App1       App3

        (App2 removed)
```

Users continue using the application without noticing.

This is one reason highly available systems can survive individual server failures.

---

# Load Balancing Algorithms

A load balancer needs a strategy to choose the next server.

Different situations require different strategies.

---

## 1. Round Robin

The simplest algorithm.

Requests are distributed one after another.

```text
Request 1 → Server 1

Request 2 → Server 2

Request 3 → Server 3

Request 4 → Server 1

Request 5 → Server 2
```

Like dealing playing cards.

### Advantages

- Extremely simple
- Fair when all servers are identical

### Problems

Not every request is equally expensive.

One request might take:

```text
5 milliseconds
```

Another might take:

```text
5 seconds
```

Round Robin ignores this.

---

## 2. Weighted Round Robin

Suppose one server is much more powerful.

```text
Server 1

32 CPUs

Server 2

8 CPUs

Server 3

8 CPUs
```

Giving each server the same number of requests wastes resources.

Instead:

```text
Weight

Server1 = 4

Server2 = 1

Server3 = 1
```

Traffic becomes:

```text
S1

S1

S1

S1

S2

S3

Repeat
```

More powerful servers receive more work.

---

## 3. Least Connections

Instead of counting requests...

Count active users.

Example:

```text
Server 1

150 Active Connections

Server 2

12 Active Connections

Server 3

20 Active Connections
```

The next request goes to:

```text
Server 2
```

because it is handling the least work.

This is useful for long-lived connections like WebSockets.

---

## 4. Least Response Time

Instead of active users...

Measure response speed.

```text
Server 1

20 ms

Server 2

60 ms

Server 3

15 ms
```

Choose the fastest one.

This adapts to real server performance.

---

## 5. IP Hash

Sometimes the same user should consistently reach the same server.

The load balancer computes:

```text
hash(Client IP)
```

Example:

```text
192.168.1.5

↓

Server 2
```

The same client usually ends up on the same backend.

This is one way to implement session affinity.

---

# Stateless vs Stateful Applications

Load balancing is easiest when applications are **stateless**.

A stateless application stores no user-specific information in its own memory between requests.

Example:

```text
Request 1

↓

App2

↓

Database

↓

Response
```

The next request can go to **any** server because all important data lives in a shared database or cache.

```text
Request 2

↓

App1

↓

Database

↓

Response
```

Everything still works.

---

Now consider a stateful application.

Suppose a user's shopping cart is stored only in Server 2's memory.

```text
Server 2

Cart:
Laptop
Headphones
```

The first request goes to Server 2.

The second request goes to Server 1.

Server 1 has never seen that cart.

The user suddenly sees an empty cart.

The application appears broken.

This is why modern distributed systems prefer stateless services whenever possible.

We'll discuss session management in more detail later.

---

# Layer 4 vs Layer 7 Load Balancers

There are two broad categories.

---

## Layer 4 (Transport Layer)

Makes decisions using network information.

Examples:

- Source IP
- Destination IP
- Port
- TCP/UDP

It does **not** inspect the HTTP request itself.

Think of it as forwarding sealed envelopes based only on the address written outside.

---

## Layer 7 (Application Layer)

Can inspect the contents of the request.

Example:

```text
GET /images
```

or

```text
GET /payments
```

It understands HTTP.

This enables much smarter routing.

Example:

```text
/images/*

↓

Image Servers

/payments/*

↓

Payment Servers
```

We'll revisit this idea in the next chapter on Reverse Proxies and later in API Gateways.

---

# Real-World Usage

## Netflix

Distributes millions of streaming requests across thousands of backend servers.

---

## Amazon

Balances shopping, checkout, recommendation, and search traffic across large server fleets.

---

## Google

Routes search requests across globally distributed infrastructure.

---

## Uber

Distributes ride requests across many backend services while removing unhealthy instances automatically.

---

# Where It Helps

Load balancers are valuable because they:

- Distribute traffic evenly.
- Improve scalability.
- Increase availability.
- Hide backend complexity from users.
- Remove failed servers automatically.
- Allow servers to be added or removed with minimal disruption.

---

# Where It Doesn't Help

A load balancer cannot:

- Reduce slow database queries.
- Fix inefficient application code.
- Replace caching.
- Replace replication or sharding.
- Solve business logic problems.

It only decides **where** requests should go.

---

# Mental Model

Imagine a busy airport.

Passengers arrive continuously.

Instead of choosing any security lane themselves, an airport employee directs them to the shortest available queue.

```text
Passengers

↓

Airport Staff

↓

Lane A

Lane B

Lane C
```

The airport staff is the load balancer.

The security lanes are the backend servers.

---

# Tradeoffs

## Advantages

- Eliminates single points of failure for application servers.
- Makes horizontal scaling straightforward.
- Improves resource utilization by spreading work.
- Automatically avoids unhealthy servers through health checks.
- Provides a single entry point for clients.

## Disadvantages

- The load balancer itself must be highly available; otherwise it can become a bottleneck or single point of failure.
- Adds an extra network hop, introducing a small amount of latency.
- Stateful applications may require session affinity or external session storage.
- Advanced routing algorithms can increase operational complexity.

---

# Common Interview Questions

### Why do we need a load balancer if DNS already exists?

DNS maps a domain name to an IP address. A load balancer distributes live traffic among backend servers and can react immediately to server health and load.

---

### What happens if one application server crashes?

Health checks detect the failure, and the load balancer stops sending new requests to that server.

---

### Why are stateless services preferred?

Because any request can be handled by any server, making scaling and failover much simpler.

---

### When would Least Connections be better than Round Robin?

When requests stay open for a long time (for example, WebSockets), because the number of active connections better reflects server load than simply counting requests.

---

### What is the difference between Layer 4 and Layer 7 load balancing?

Layer 4 routes based on transport-layer information like IPs and ports. Layer 7 understands application protocols like HTTP and can route based on URLs, headers, or other request content.

---

# Before vs After Architecture

## Before

```text
                 Users
                    │
                    ▼
             +-------------+
             | App Server  |
             +-------------+
```

Problems:

- One server handles everything.
- Server failure causes downtime.
- Scaling requires upgrading a single machine.

---

## After

```text
                  Users
                     │
                     ▼
            +----------------+
            | Load Balancer  |
            +----------------+
              │     │      │
              ▼     ▼      ▼
          +------+ +------+ +------+
          | App1 | | App2 | | App3 |
          +------+ +------+ +------+
```

Benefits:

- Traffic is distributed.
- Servers can fail independently.
- Capacity increases by adding more servers.

---

# Connections

We now have a system where:

- DNS directs users to a load balancer.
- The load balancer selects an application server.

But another interesting question arises.

What if the load balancer wants to:

- Cache static files before they reach the application?
- Compress responses?
- Terminate HTTPS connections?
- Hide internal servers from the public internet?
- Route requests based on URLs like `/api` or `/images`?

These responsibilities go beyond simple traffic distribution.

They are handled by another component that often sits in front of application servers:

> **Chapter 3 — Reverse Proxy**
