# Chapter 5 — Service Discovery

---

# Goal

Understand how services in a distributed system dynamically find and communicate with each other as servers are constantly created, destroyed, and moved.

---

# The Problem

Throughout this module, we've built a modern request path.

```text
                 User
                   │
                   ▼
                  DNS
                   │
                   ▼
            Load Balancer
                   │
                   ▼
            Reverse Proxy
                   │
                   ▼
             API Gateway
                   │
                   ▼
             User Service
```

Everything looks complete.

The user reaches the correct service.

Problem solved...

Or is it?

---

Let's zoom in.

Suppose our API Gateway needs to call the User Service.

Maybe its address is:

```text
10.1.5.21
```

So we configure:

```text
User Service

↓

10.1.5.21
```

Easy.

Everything works.

---

Now imagine one day the User Service crashes.

The orchestration platform (Kubernetes, ECS, Nomad, etc.) immediately starts a replacement.

The new instance starts at:

```text
10.1.8.43
```

But the gateway still thinks:

```text
User Service

↓

10.1.5.21
```

Every request now fails.

---

Someone updates the configuration.

Everything works again.

---

Now imagine this happens every few minutes.

Because in modern cloud systems...

Servers are constantly:

- Starting
- Stopping
- Restarting
- Scaling up
- Scaling down
- Moving to different machines

Their IP addresses are constantly changing.

Hardcoding addresses simply doesn't work.

---

# A Bigger Problem

Now imagine an e-commerce system.

```text
User Service

10 instances

Order Service

15 instances

Inventory Service

20 instances

Payment Service

5 instances
```

Total:

```text
50 application instances
```

Tomorrow during a sale:

```text
200 instances
```

Tomorrow night:

```text
35 instances
```

Who updates all these IP addresses?

Nobody should.

The system must discover services automatically.

---

# Why Existing Solutions Fail

## DNS

DNS works wonderfully for users.

```text
amazon.com

↓

IP Address
```

But DNS isn't designed to track thousands of application instances that may appear or disappear every few seconds.

Traditional DNS records also tend to change much less frequently than containerized workloads.

---

## Load Balancer

A load balancer distributes traffic.

But first it must know:

> Which backend servers currently exist?

It cannot magically discover them.

---

## API Gateway

The gateway routes requests.

But it still needs to know:

```text
Where is User Service?
```

Someone has to provide that information.

---

# The Big Idea

**Service Discovery** is a mechanism that allows services to automatically find the current location of other services.

Instead of hardcoding IP addresses, services ask:

> "Where is the User Service right now?"

---

# Think About a Hotel

Imagine visiting a large hotel.

You ask the receptionist:

```text
Where is Conference Room A?
```

The receptionist replies:

```text
Second Floor
```

Tomorrow the conference room is moved.

You ask again.

Now the answer is:

```text
Third Floor
```

You never memorize the location.

You simply ask whenever you need it.

Service Discovery works exactly the same way.

---

# Service Registry

At the heart of Service Discovery is something called a **Service Registry**.

Think of it as a constantly updated directory.

Instead of storing people's phone numbers...

it stores running services.

Example:

```text
User Service

↓

10.1.8.43

10.1.8.51

10.1.9.11
```

Another entry:

```text
Payment Service

↓

10.1.7.10

10.1.7.18
```

Whenever services start or stop, the registry updates automatically.

---

# Basic Architecture

```text
                 Service Registry
                 ┌──────────────┐
                 │ User Service │
                 │ Order Service│
                 │ Payment Svc  │
                 └──────────────┘
                      ▲
                      │
          Registers itself
                      │
                 User Service

```

Meanwhile:

```text
API Gateway

↓

Ask Registry

↓

Where is User Service?

↓

Receive Current Addresses

↓

Send Request
```

Nobody hardcodes IP addresses anymore.

---

# How Does a Service Register Itself?

Suppose a new User Service instance starts.

```text
User Service

10.1.8.43
```

The first thing it does is tell the registry:

```text
Hi.

I'm User Service.

My address is

10.1.8.43
```

The registry stores it.

Now other services can discover it.

---

Suppose another instance starts.

```text
10.1.8.51
```

It registers too.

The registry now knows both instances.

---

Suppose one crashes.

It is removed from the registry.

Everyone immediately stops using it.

---

# Heartbeats

How does the registry know a service is still alive?

Services periodically send small heartbeat messages.

```text
User Service

↓

"I'm alive."
```

Every few seconds.

If the heartbeat stops arriving:

```text
Heartbeat

❌ Missing
```

The registry assumes the service has failed.

It removes it from the directory.

Future requests won't be routed there.

---

# Service Discovery Flow

Let's walk through a complete request.

---

### Step 1

User opens Instagram.

```text
Mobile App

↓

API Gateway
```

---

### Step 2

Gateway needs User Service.

Instead of using a hardcoded IP:

```text
Where is User Service?
```

---

### Step 3

Registry replies:

```text
User Service

↓

10.1.8.43

10.1.8.51

10.1.9.11
```

---

### Step 4

Gateway chooses one instance.

```text
10.1.8.51
```

---

### Step 5

Request succeeds.

---

Later...

One instance crashes.

The registry removes it.

Future lookups automatically return only healthy instances.

No configuration changes are required.

---

# Client-Side Service Discovery

There are two common approaches.

The first is **Client-Side Discovery**.

Here, the client asks the registry directly.

```text
Client

↓

Service Registry

↓

Current Servers

↓

Client chooses server

↓

Application
```

The client is responsible for:

- Querying the registry.
- Choosing a server.
- Retrying if needed.

This gives clients more control but also more responsibility.

Netflix's Eureka ecosystem is a well-known example of this pattern.

---

# Server-Side Service Discovery

In the second approach, the client doesn't know about the registry.

Instead:

```text
Client

↓

Load Balancer

↓

Registry

↓

Application
```

The load balancer or proxy queries the registry.

The client simply sends requests as usual.

This keeps clients simpler.

Modern cloud platforms often prefer this approach.

---

# Which One is Better?

Neither is universally better.

## Client-Side Discovery

Advantages:

- Less work for the load balancer.
- Client can choose sophisticated routing strategies.
- Fewer network hops.

Disadvantages:

- Every client needs discovery logic.
- Every client must understand the registry.
- More complex client libraries.

---

## Server-Side Discovery

Advantages:

- Simpler clients.
- Centralized routing.
- Easier to evolve infrastructure.

Disadvantages:

- Load balancer or proxy becomes more sophisticated.
- Another infrastructure component handles discovery.

---

# Service Discovery in Kubernetes

Suppose Kubernetes creates a Deployment.

```text
User Service

↓

5 Pods
```

Pods are constantly created and destroyed.

Instead of applications remembering Pod IPs...

Kubernetes creates a stable Service.

```text
user-service
```

Applications simply call:

```text
http://user-service
```

Kubernetes automatically routes the request to one of the healthy Pods behind that Service.

This is built-in service discovery.

Applications don't need to know Pod IPs at all.

---

# Why Not Just Use DNS?

This is a very common interview question.

The answer is:

Modern systems actually **do** use DNS—but in a specialized way.

For example, Kubernetes automatically creates DNS names like:

```text
user-service.default.svc.cluster.local
```

Applications query this stable name instead of individual Pod IPs.

Behind the scenes, Kubernetes updates the mapping whenever Pods change.

So DNS remains part of the solution, but it is backed by a dynamic service registry rather than manually managed records.

---

# Real-World Usage

## Netflix

Uses service discovery so thousands of microservice instances can find each other without fixed IP addresses.

---

## Uber

Continuously discovers newly created service instances as autoscaling adjusts capacity.

---

## Kubernetes

Provides built-in service discovery through Services and cluster DNS.

---

## Amazon

Cloud-native workloads rely on service discovery because instances and containers are constantly changing.

---

# Where It Helps

Service Discovery is valuable because it:

- Eliminates hardcoded IP addresses.
- Supports autoscaling.
- Automatically adapts to failures.
- Simplifies deployment.
- Allows services to move freely.
- Enables dynamic cloud infrastructure.

---

# Where It Doesn't Help

Service Discovery cannot:

- Replace load balancing.
- Replace authentication.
- Replace API gateways.
- Improve slow application code.
- Replace databases.

Its only responsibility is helping services locate one another.

---

# Mental Model

Imagine ordering food through a delivery app.

You don't memorize which driver is assigned today.

Every time you order, the app finds an available nearby driver.

Tomorrow it may be a different driver.

The day after, another one.

You always ask the system.

You never remember individual drivers.

Service Discovery works exactly the same way.

Services ask:

```text
Where is Order Service today?
```

instead of remembering yesterday's address.

---

# Tradeoffs

## Advantages

- Removes hardcoded service addresses.
- Enables dynamic scaling.
- Automatically handles instance failures.
- Simplifies deployments.
- Supports cloud-native architectures.

## Disadvantages

- Requires additional infrastructure (registry or orchestration platform).
- Discovery failures can impact communication between services.
- Registry consistency and availability become important.
- Frequent lookups can add overhead, so clients often cache discovery results briefly.

---

# Common Interview Questions

### Why do we need Service Discovery?

Because service instances in modern distributed systems are constantly created, destroyed, and moved. Hardcoded IP addresses quickly become outdated.

---

### What is a Service Registry?

A central directory that keeps track of currently running service instances and their network locations.

---

### What are heartbeats?

Periodic messages sent by services to indicate they are still healthy. Missing heartbeats usually cause the registry to remove the service instance.

---

### What is the difference between Client-Side and Server-Side Service Discovery?

In client-side discovery, the client queries the registry and chooses a service instance. In server-side discovery, the client sends requests to a proxy or load balancer, which performs the discovery.

---

### Does Kubernetes use Service Discovery?

Yes. Kubernetes provides built-in service discovery using Services and cluster DNS, allowing applications to communicate through stable service names instead of changing Pod IPs.

---

# Before vs After Architecture

## Before

```text
API Gateway

↓

10.1.5.21
```

Problems:

- Hardcoded IP address.
- Breaks when the service moves.
- Difficult to support autoscaling.

---

## After

```text
                 API Gateway
                      │
                      ▼
             Service Registry
                      │
          ┌───────────┴───────────┐
          ▼           ▼           ▼
      User Svc    User Svc    User Svc
      10.1.8.43   10.1.8.51   10.1.9.11
```

Benefits:

- Services register automatically.
- Failed instances disappear automatically.
- New instances become available immediately.
- No manual configuration updates.

---

# Connections

We've now completed the entire journey of a user's request through a modern distributed system:

```text
User
  │
  ▼
DNS
  │
  ▼
Load Balancer
  │
  ▼
Reverse Proxy
  │
  ▼
API Gateway
  │
  ▼
Service Discovery
  │
  ▼
Application Service
```

We've answered **how a request reaches the correct service**.

The next question is equally fundamental:

Once two machines know how to find each other...

**How do they actually communicate over the network?**

Should they use TCP or UDP?

How does HTTPS work?

What happens after you press Enter in a browser?

Those questions begin our next module:

> **Module 5 — Networking Foundations**: Understanding the networking concepts every system designer needs before exploring service-to-service communication.
