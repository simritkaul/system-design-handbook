# Module 8, Chapter 5 — Rate Limiting

## 1. Goal

**Control how many requests a system allows from a client within a given period so that excessive traffic cannot overwhelm the system or unfairly consume its resources.**

---

# 2. The Problem

Imagine we are building an API for an application like Instagram.

Suppose our API has:

```text
POST /login
GET  /users/{id}
GET  /posts/{id}
POST /posts
```

Normally, a user might make:

```text
10 requests/second
```

That's completely reasonable.

But now imagine one client suddenly sends:

```text
10,000 requests/second
```

There are several possible reasons:

- A buggy client is stuck in a loop.
- Someone is scraping the API.
- A bot is attacking the system.
- A client is retrying aggressively.
- Someone is deliberately trying to exhaust our resources.

The important thing is that **our server doesn't care why the requests arrived**.

It still has to process them.

So:

```text
Client
   |
   | 10,000 req/sec
   v
Application Servers
   |
   +---- Database
   +---- Cache
   +---- Other Services
```

Those requests consume:

- CPU
- memory
- network bandwidth
- database connections
- database queries
- threads
- downstream service capacity

Eventually:

```text
Too many requests
        ↓
Resources exhausted
        ↓
Latency increases
        ↓
Requests time out
        ↓
Clients retry
        ↓
Even more requests
        ↓
System becomes unstable
```

This is especially dangerous because of something we've already studied.

### Remember Retry?

Suppose Service A calls Service B.

Service B becomes slow.

Service A times out and retries:

```text
A → B
A → B
A → B
```

Now imagine thousands of clients doing this simultaneously.

A temporary problem can become a **traffic amplification problem**.

So we need a mechanism that says:

> "Enough. This client is sending more traffic than we're willing to accept."

That's **rate limiting**.

---

# 3. Why Existing Solutions Fail

We've already learned several mechanisms for protecting a distributed system.

### Timeout

Timeout answers:

> "How long am I willing to wait?"

It doesn't stop clients from sending requests.

---

### Retry with Backoff

Retry answers:

> "What should I do if a request fails?"

It can actually **increase traffic** if poorly controlled.

---

### Circuit Breaker

Circuit breaker answers:

> "Should I temporarily stop calling an unhealthy downstream service?"

But imagine this:

```text
100,000 clients
      |
      v
   Service A
```

The problem may be that **Service A itself** is receiving too much traffic.

A circuit breaker doesn't fundamentally solve that.

---

### Bulkhead

Bulkhead answers:

> "How much of my resources should one workload be allowed to consume?"

For example:

```text
Service
 ├── Payment resources
 ├── Search resources
 └── Recommendation resources
```

It prevents one workload from consuming everything.

But we may still want to control the **incoming request rate** itself.

---

So we need another mechanism:

```text
             Rate Limiter
                  ↓
Client → [Allowed?] → Application
              ↓
           Rejected
```

---

# 4. The Big Idea

> **Rate limiting puts a controlled limit on how quickly requests are allowed to enter a system.**

That's the entire concept.

For example:

```text
100 requests / minute / user
```

means:

```text
Requests 1–100  → allowed
Request 101     → rejected
```

The limit could be based on:

- user
- API key
- IP address
- tenant
- endpoint
- application
- region
- or some combination

---

# 5. Detailed Explanation

Let's start with the simplest possible rate limiter.

Suppose we have:

```text
Maximum = 5 requests / second
```

A client sends:

```text
t=0.1   request
t=0.2   request
t=0.3   request
t=0.4   request
t=0.5   request
t=0.6   request
```

The first five are accepted.

The sixth is rejected.

```text
Client
  |
  | request
  v
+----------------+
| Rate Limiter   |
+----------------+
   |         |
 allowed   rejected
   |         |
   v         v
Server     429
```

The application server never needs to process the rejected request.

That is important.

Rate limiting is generally most useful **before expensive work happens**.

---

## What does "rate" actually mean?

This is where things become interesting.

Suppose I say:

> 100 requests per minute.

There are different ways to interpret that.

### Interpretation 1

A fixed one-minute window:

```text
12:00:00 → 12:01:00
```

Allow 100 requests.

Then:

```text
12:01:00 → 12:02:00
```

another 100.

But there is a subtle problem.

Suppose a client sends:

```text
12:00:59 → 100 requests
12:01:00 → 100 requests
```

That's:

```text
200 requests
```

in roughly:

```text
1 second
```

even though we technically respected:

```text
100 requests/minute
```

This is called the **boundary problem**.

So how we implement rate limiting matters.

And that gives us our first major family of algorithms.

---

# 6. Rate Limiting Algorithms

The major approaches we care about for system design are:

1. Fixed Window
2. Sliding Window
3. Token Bucket
4. Leaky Bucket

Let's understand why each exists.

---

# 6.1 Fixed Window

The simplest approach.

Divide time into fixed intervals.

For example:

```text
1 minute = one window
Limit = 100 requests
```

We maintain:

```text
Window       Requests
12:00–12:01    73
12:01–12:02    41
```

When a request arrives:

```text
Is current count < 100?

Yes → allow
No  → reject
```

Then the counter resets when the next window begins.

### Example

```text
12:00
 |
 | requests
 |
12:01
```

Within that window:

```text
0 ───────────────────── 100
                      ↑
                    limit
```

Once we hit 100:

```text
Request 101 → reject
```

---

## Advantages

Very simple.

```text
counter++
```

Easy to understand and relatively cheap to implement.

---

## Problem

The boundary issue we just saw.

```text
Window 1             Window 2
|------------------|------------------|
       100 requests     100 requests
             ↑
       boundary
```

A client can effectively burst:

```text
100 + 100
```

around the boundary.

So fixed windows provide a rough limit, but not a very smooth one.

---

# 6.2 Sliding Window

Instead of thinking:

> "Which fixed window are we in?"

we think:

> "How many requests happened during the last X seconds?"

Suppose our limit is:

```text
100 requests / 60 seconds
```

At:

```text
12:01:30
```

we ask:

> How many requests has this client made from 12:00:30 to 12:01:30?

If:

```text
< 100
```

allow.

If:

```text
≥ 100
```

reject.

The window moves continuously.

```text
Time →
──────────────────────────────────

             [------ 60 sec ------]
                         ↑
                      "now"
```

This removes much of the fixed-window boundary problem.

---

## But there's a cost

To implement a precise sliding window, we may need to remember request timestamps.

For example:

```text
User A

12:00:31
12:00:33
12:00:37
12:00:41
...
```

Then old timestamps can be removed as they leave the window.

For extremely high traffic, storing and processing individual timestamps can become expensive.

So we need a more efficient approach.

---

# 6.3 Token Bucket

This is one of the **most important rate-limiting algorithms for system design interviews**.

Imagine a bucket.

```text
       ┌──────────────┐
       │ ● ● ● ● ●    │
       │ ● ● ●        │
       │              │
       └──────────────┘
          capacity = 10
```

The bucket holds tokens.

Each request needs **one token**.

If a token exists:

```text
request → take token → allow
```

If there are no tokens:

```text
request → no token → reject
```

But tokens don't stay empty forever.

They are continuously added at a fixed rate.

For example:

```text
Bucket capacity = 10 tokens
Refill rate     = 5 tokens/sec
```

So:

```text
          5 tokens/sec
               ↓
         ┌──────────┐
         │ tokens   │
         └──────────┘
               |
               | 1 token
               v
            Request
```

---

## Why have both capacity and refill rate?

This gives us two useful properties:

### Refill rate controls sustained traffic

If:

```text
5 tokens/sec
```

then the client can sustainably send approximately:

```text
5 requests/sec
```

---

### Bucket capacity allows bursts

Suppose the bucket can hold:

```text
10 tokens
```

and the client has been idle.

The bucket fills:

```text
● ● ● ● ● ● ● ● ● ●
```

The client can suddenly send:

```text
10 requests
```

very quickly.

That's a **burst**.

After that, it must wait for tokens to refill.

This is extremely useful for APIs because real traffic isn't perfectly uniform.

A user might legitimately do:

```text
request
request
request
request
```

quickly and then do nothing for several seconds.

We don't necessarily want to reject that just because the average rate is low.

---

# 6.4 Leaky Bucket

Now imagine a bucket with a hole at the bottom.

Requests enter:

```text
        Requests
       ↓ ↓ ↓ ↓ ↓
      ┌─────────┐
      │         │
      │         │
      └────┬────┘
           ↓
        constant
          rate
```

The system processes requests at a relatively fixed rate.

For example:

```text
10 requests/sec
```

If requests arrive faster than that, they accumulate in the bucket/queue.

If the bucket becomes full:

```text
new request → reject
```

The important distinction is:

### Token Bucket

Controls **whether requests are allowed**, while permitting bursts up to bucket capacity.

### Leaky Bucket

Controls **the rate at which requests leave**, producing a smoother output rate.

---

# 7. Token Bucket vs Leaky Bucket

This is a common interview question.

|              | Token Bucket            | Leaky Bucket           |
| ------------ | ----------------------- | ---------------------- |
| Main idea    | Requests consume tokens | Requests enter a queue |
| Bursts       | Allows bursts           | Smooths bursts         |
| Output rate  | Can be bursty           | More uniform           |
| Empty system | Tokens accumulate       | Queue stays empty      |
| Useful for   | API rate limiting       | Traffic shaping        |

Think:

```text
Token Bucket:

     burst
   ↓↓↓↓↓↓↓
──────────────→
```

versus:

```text
Leaky Bucket:

↓↓↓↓↓↓↓↓↓↓↓↓
      |
      v
   1/sec
   1/sec
   1/sec
```

---

# 8. Where Does the Rate Limiter Live?

This is an important system-design decision.

We could put it:

```text
Client
  |
  v
Rate Limiter
  |
  v
Application
```

Or:

```text
Client
  |
  v
Load Balancer
  |
  v
Rate Limiter
  |
  v
Application
```

Or potentially at an API gateway / edge layer.

The principle is:

> **Reject excessive traffic as early as practical, before it consumes expensive downstream resources.**

---

# 9. Distributed Rate Limiting

Now we hit the interesting distributed-systems problem.

Suppose we have:

```text
                Load Balancer
                 /    |    \
                /     |     \
              S1      S2     S3
```

Suppose the limit is:

```text
100 requests/minute/user
```

User A sends:

```text
S1 → 40 requests
S2 → 30 requests
S3 → 50 requests
```

Each server independently sees:

```text
S1 = 40
S2 = 30
S3 = 50
```

No individual server thinks the user exceeded 100.

But globally:

```text
40 + 30 + 50 = 120
```

The user should have been limited.

This means a simple local counter is insufficient when requests can hit multiple servers.

---

# 10. Centralized Rate-Limiting State

We can introduce shared state.

```text
              Load Balancer
                   |
          +--------+--------+
          |        |        |
         S1       S2       S3
          |        |        |
          +--------+--------+
                   |
                   v
             Shared Counter
```

Every server checks the same state.

For example:

```text
User A → current count = 97
```

S1 receives request:

```text
97 → 98
```

S2 receives request:

```text
98 → 99
```

S3 receives request:

```text
99 → 100
```

Next request:

```text
100 → reject
```

Now the limit is global.

But we've introduced another problem:

> **The shared state itself must be extremely fast and highly available.**

If every API request needs to synchronously contact the shared rate-limit state:

```text
Request
   |
   v
Server
   |
   v
Shared State
   |
   v
Server
   |
   v
Application
```

then rate limiting itself becomes part of our request latency and potentially a bottleneck.

We'll discuss the technology used for this later in Module 9.

For now, the concept is what matters:

> Distributed rate limiting usually requires coordination or carefully designed approximation.

---

# 11. What Exactly Are We Limiting?

A sophisticated system doesn't necessarily use one global limit.

We can have multiple dimensions.

For example:

```text
Per IP:
1000 requests/minute

Per user:
200 requests/minute

Per API:
100 requests/minute

Per endpoint:
20 requests/second

Global:
1 million requests/second
```

A request might have to pass multiple limits:

```text
             Request
                |
        +-------+-------+
        |       |       |
      IP      User    Endpoint
     limit    limit     limit
        |       |       |
        +-------+-------+
                |
             Allowed?
```

This is particularly useful for multi-tenant systems.

Suppose we have:

```text
Tenant A → normal customer
Tenant B → enterprise customer
```

We might give them different limits:

```text
Tenant A → 1,000 req/min
Tenant B → 100,000 req/min
```

Rate limiting therefore becomes partly a **resource allocation policy**.

---

# 12. What Happens When We Reject a Request?

Usually, the client should be told clearly that it has exceeded the limit.

Conceptually:

```text
HTTP 429
Too Many Requests
```

We may also tell the client approximately when it can try again.

For example:

```text
Retry-After: ...
```

The exact protocol details aren't the important part here.

The design principle is:

> **Don't make clients guess whether they should retry. Give them enough information to behave correctly.**

And this connects directly to our previous chapter on retries.

If a client receives a rate-limit response and immediately does:

```text
retry
retry
retry
retry
```

we've created another traffic problem.

Rate limiting and retry policies therefore need to be designed together.

---

# 13. Rate Limiting vs Throttling

These terms are often used interchangeably, but there's a useful distinction.

### Rate limiting

Controls the maximum rate of requests.

```text
100 req/min
```

Excess requests might be rejected.

### Throttling

More broadly means **slowing or controlling traffic/resource consumption**.

It might:

- delay requests
- queue requests
- reduce processing rate
- reject requests

So:

> Rate limiting is one mechanism for throttling traffic.

In interviews, don't get hung up on terminology; explain the actual behavior you're designing.

---

# 14. Real-World Usage

Rate limiting is extremely common in:

- public APIs
- authentication endpoints
- payment APIs
- SaaS platforms
- cloud APIs
- messaging systems
- search APIs
- social platforms

A particularly important example is **login**.

Imagine:

```text
POST /login
```

Without protection:

```text
Attacker
   |
   +-- password attempt
   +-- password attempt
   +-- password attempt
   +-- ...
```

Millions of attempts could be made against an account or service.

A rate limiter can constrain attempts:

```text
IP / account / endpoint
        ↓
limited attempts
        ↓
authentication service
```

Notice something important:

**Rate limiting is not only about protecting infrastructure.**

It can also enforce **business and security policies**.

---

# 15. Where It Helps

Rate limiting is especially useful for:

### 1. Protecting expensive services

```text
API → Database
```

Prevent excessive database work.

### 2. Preventing abuse

Bots, scraping, brute-force attempts, etc.

### 3. Fairness

Prevent one customer from consuming all shared capacity.

```text
Tenant A ─┐
Tenant B ─┼→ shared system
Tenant C ─┘
```

### 4. Protecting downstream systems

Even if your application can process:

```text
100k req/sec
```

a downstream service might only handle:

```text
20k req/sec
```

You can rate-limit calls before reaching it.

### 5. Cost control

More requests often mean more:

- compute
- database operations
- external API calls
- bandwidth

Limiting unnecessary traffic can therefore control cost.

---

# 16. Where It Doesn't Help

Rate limiting is not a universal reliability mechanism.

### It doesn't fix inefficient code

If one request takes:

```text
10 seconds
```

then even:

```text
10 req/sec
```

might overwhelm the service.

---

### It doesn't guarantee availability

A system can still fail because of:

- database failure
- network partitions
- bugs
- hardware failures
- dependency failures

---

### It doesn't eliminate DDoS risk by itself

A rate limiter running inside your infrastructure may itself become overwhelmed by huge traffic volumes.

For very large attacks, traffic may need to be filtered much earlier, closer to the network edge.

---

### It can reject legitimate traffic

A poorly chosen limit can cause:

```text
legitimate user
      ↓
429 Too Many Requests
```

So limits need to reflect actual workload characteristics.

---

# 17. Mental Model

Imagine a nightclub.

There is a maximum safe capacity:

```text
100 people
```

The bouncer stands at the entrance.

```text
People
 ↓
[Bouncer]
 ↓
Club
```

When the club has space:

```text
person → allowed
```

When it is full:

```text
person → wait/reject
```

But rate limiting is slightly different from simply checking capacity.

Imagine the bouncer also has a rule:

> "We will admit at most 10 people per minute."

Now we're controlling **the rate of entry**, not merely the number currently inside.

And with a token bucket:

> "Normally admit 10 people/minute, but if we've been quiet, we can admit a small burst immediately."

That's a pretty good intuition for the difference between sustained rate and burst capacity.

---

# 18. Tradeoffs

## Advantages

### Protects system capacity

Prevents uncontrolled request volume from reaching expensive components.

### Improves fairness

One client cannot easily consume all resources.

### Helps with abuse

Useful against brute force, scraping, excessive API usage, etc.

### Predictable resource usage

Makes traffic patterns easier to control.

### Can enforce business policies

Different users/tenants can receive different quotas.

---

## Disadvantages

### Adds complexity

Especially in distributed systems.

### Requires state

Depending on the algorithm, we may need counters, timestamps, tokens, or queues.

### Distributed coordination is difficult

Multiple servers must agree on the effective limit.

### Incorrect limits hurt users

Too strict:

```text
legitimate → rejected
```

Too loose:

```text
system → overloaded
```

### The rate limiter itself becomes critical infrastructure

If every request depends on it:

```text
Rate Limiter DOWN
       ↓
What happens to the entire API?
```

We must design its failure behavior carefully.

---

# 19. Common Interview Questions

## Q1. Why can't we just rate-limit independently on each application server?

Because requests are distributed.

If the global limit is:

```text
100 req/min
```

and there are three servers, independently allowing 100 on each means the client could actually make:

```text
300 req/min
```

So local limits and global limits are different policies.

---

## Q2. Token bucket vs fixed window?

**Fixed window** counts requests within fixed time boundaries.

**Token bucket** models permission to make requests using replenishing tokens.

Token bucket is generally better when we want:

- sustained rate control
- controlled bursts

---

## Q3. Why allow bursts?

Because real traffic isn't perfectly uniform.

A user might legitimately make several requests very quickly.

For example:

```text
Load page
  ↓
request profile
request posts
request notifications
request recommendations
```

Rejecting every short burst can make an API unnecessarily restrictive.

---

## Q4. Where should rate limiting happen?

Preferably as early as practical:

```text
Internet
   ↓
Edge / Gateway
   ↓
Rate Limiter
   ↓
Application
   ↓
Database
```

The earlier we reject traffic, the fewer resources it consumes.

But different limits can exist at different layers.

---

## Q5. What happens if the rate limiter goes down?

There are two broad choices.

### Fail open

```text
Rate limiter unavailable
       ↓
Allow requests
```

Better availability, worse protection.

### Fail closed

```text
Rate limiter unavailable
       ↓
Reject requests
```

Better protection, worse availability.

The right choice depends on the system.

For something like a critical internal service, availability might dominate.

For a security-sensitive endpoint, fail-closed behavior may be more appropriate.

This is a classic distributed-systems tradeoff:

> **Do we prefer availability or protection when the control mechanism itself fails?**

---

# 20. Before vs After Architecture

### Before rate limiting

```text
                 ┌─────────────┐
Users ──────────→│ Application  │
                 └──────┬──────┘
                        |
                        v
                    Database
```

Any amount of traffic reaches the application.

---

### After rate limiting

```text
                 ┌──────────────┐
Users ──────────→│ Rate Limiter │
                 └──────┬───────┘
                        |
                 ┌──────┴───────┐
                 │              │
              Allowed         Rejected
                 |               |
                 v               v
          ┌─────────────┐       429
          │ Application │
          └──────┬──────┘
                 |
                 v
             Database
```

The architecture has gained a **traffic admission control layer**.

---

# 21. The Deeper Connection

Let's connect this chapter to the previous four.

We now have:

```text
Timeout
   ↓
Don't wait forever

Retry + Backoff
   ↓
Recover from transient failures carefully

Circuit Breaker
   ↓
Stop calling an unhealthy dependency

Bulkhead
   ↓
Prevent one workload from consuming all resources

Rate Limiting
   ↓
Control how much traffic is allowed in
```

Notice how Module 8 is building toward one larger idea:

> **A reliable distributed system doesn't simply process every request it receives. It actively controls failure, resource consumption, and communication.**

That's the mental model I want you to retain.

---

# 22. Connections

Our next chapter is:

## Chapter 6 — Distributed Locking

Rate limiting asks:

> **"How many requests are allowed to proceed?"**

Distributed locking asks a different question:

> **"Which one of these distributed processes is allowed to perform this operation?"**

For example:

```text
Two servers
     |
     | both try to process
     | the same job
     v
Shared resource
```

We may need to ensure:

```text
Server A → allowed
Server B → wait/reject
```

That introduces **mutual exclusion across machines**, which is much harder than a normal in-process lock.

---

# 23. Key Takeaways

If you remember only these things:

1. **Rate limiting controls how quickly requests enter a system.**
2. It protects capacity, prevents abuse, and improves fairness.
3. **Fixed Window** is simple but has boundary problems.
4. **Sliding Window** gives a more accurate moving-window limit but can require more state.
5. **Token Bucket** allows controlled bursts while maintaining a sustainable rate.
6. **Leaky Bucket** smooths traffic toward a relatively constant output rate.
7. In distributed systems, local counters aren't enough when the limit is global.
8. Rate limiting should generally happen **before expensive work**.
9. Rate limiting and **retry behavior must be designed together**.
10. The rate limiter itself becomes critical infrastructure, so its failure behavior matters.

### One sentence to remember

> **Rate limiting is the system's admission control: before doing expensive work, decide whether this request has earned the right to consume resources.**

---

**Next:** Module 8, Chapter 6 — **Distributed Locking**.
