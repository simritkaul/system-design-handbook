# Chapter 3: Circuit Breaker

## 1. Goal

Understand how a distributed system can **stop repeatedly calling an unhealthy dependency**, prevent cascading failures, and give that dependency time to recover.

Our journey so far:

```text
Timeout
   │
   ▼
Stop waiting forever
   │
   ▼
Retry
   │
   ▼
Exponential Backoff + Jitter
   │
   ▼
Avoid making transient failures worse
```

But imagine the dependency continues failing.

```text
Request → Timeout
Request → Timeout
Request → Timeout
Request → Timeout
Request → Timeout
...
```

Even with backoff, we are repeatedly discovering the same failure.

This leads us to:

> **Circuit Breaker**

---

# 2. The Problem

Imagine an application with several services:

```text
                         ┌──────────────┐
                         │ Payment      │
                         │ Service      │
                         └──────▲───────┘
                                │
                                │
User → API → Order Service ─────┘
```

Normally:

```text
Order Service
      │
      │ Payment request
      ▼
Payment Service
      │
      ▼
Success
```

Everything works.

Now imagine Payment Service has a serious outage.

```text
Order Service
      │
      │ Request
      ▼
Payment Service
      X
```

The Order Service waits.

```text
Timeout
```

Because the failure might be transient, it retries.

```text
Request
   │
 Timeout
   │
 Backoff
   │
 Retry
   │
 Timeout
   │
 Backoff
   │
 Retry
```

But Payment Service is completely down.

So this continues:

```text
Order Service
      │
      ├──── Request ────► X
      │
      ├──── Retry ──────► X
      │
      ├──── Retry ──────► X
      │
      └──── Retry ──────► X
```

Now multiply this by thousands of requests.

```text
10,000 users
     │
     ▼
Order Service
     │
     ▼
Payment Service
     X
```

The Order Service is wasting resources attempting operations that are highly unlikely to succeed.

And worse, those attempts can consume:

- threads
- connections
- CPU
- memory
- network capacity
- retry capacity

Eventually the Order Service itself can become unhealthy.

We have created:

```text
Payment Service failure
        │
        ▼
Repeated calls
        │
        ▼
Order Service resource exhaustion
        │
        ▼
Order Service becomes unhealthy
        │
        ▼
Upstream services affected
```

This is a **cascading failure**.

---

# 3. Why Existing Solutions Fail

We already have two mechanisms.

## Timeout

Timeout says:

> "Don't wait forever."

Good.

But:

```text
Request
   │
   ▼
Timeout
```

doesn't prevent the next request.

---

## Retry with Backoff

Retry says:

> "Maybe this was temporary. Try again."

And backoff says:

> "Don't immediately try again."

Good.

But suppose the dependency has been down for five minutes.

We might still have:

```text
Request
   ↓
Timeout
   ↓
Backoff
   ↓
Retry
   ↓
Timeout
   ↓
Backoff
   ↓
Retry
   ↓
...
```

We are still repeatedly sending traffic toward a dependency we already know is unhealthy.

So we need another idea.

Instead of asking:

> "Should this individual request retry?"

we want to ask:

> **"Should we even attempt to call this dependency right now?"**

That is a different problem.

---

# 4. The Big Idea

A **circuit breaker** monitors failures from a dependency.

If failures become sufficiently frequent, it temporarily **stops sending requests to that dependency**.

Conceptually:

```text
Healthy dependency
      │
      ▼
Requests allowed
      │
      ▼
Repeated failures
      │
      ▼
Circuit opens
      │
      ▼
Requests rejected immediately
      │
      ▼
Wait for recovery
      │
      ▼
Test dependency
      │
      ▼
Recover → close circuit
```

The crucial idea is:

> **When a dependency is clearly unhealthy, fail fast instead of repeatedly calling it.**

---

# 5. The Circuit Breaker Analogy

Think about the electrical circuit breaker in a house.

Normally:

```text
Electricity
    │
    ▼
Circuit
    │
    ▼
Appliances
```

Everything works.

But suppose there is a dangerous electrical fault.

Instead of allowing the fault to continue damaging the system:

```text
Power
  │
  ▼
Fault
  │
  ▼
Breaker trips
  │
  ▼
Circuit disconnected
```

The circuit breaker **interrupts the flow**.

The distributed-systems version follows the same intuition.

Normally:

```text
Order Service
      │
      ▼
Payment Service
```

After repeated failures:

```text
Order Service
      │
      X
   Circuit
   OPEN
      X
      │
Payment Service
```

The caller doesn't even attempt the network call.

It can fail fast or use another strategy.

---

# 6. The Three States

A classic circuit breaker has three conceptual states:

```text
             failures exceed threshold
        ┌──────────────────────────────┐
        │                              ▼
   ┌──────────┐                   ┌──────────┐
   │  CLOSED  │ ─────────────────►│   OPEN   │
   └──────────┘                   └────┬─────┘
        ▲                              │
        │                              │ wait
        │                              ▼
        │                         ┌───────────┐
        └─────────────────────────│ HALF-OPEN │
             recovery             └───────────┘
```

Let's understand each one.

---

# 7. State 1 — Closed

This is the normal state.

```text
Circuit = CLOSED
```

Requests are allowed.

```text
Order Service
      │
      ▼
Circuit Breaker
      │
      ▼
Payment Service
```

The circuit breaker observes what happens.

For example:

```text
Requests: 100
Failures: 2
```

That's probably normal.

The circuit remains closed.

```text
CLOSED
  │
  ├── Success
  ├── Success
  ├── Success
  ├── Failure
  ├── Success
  └── Success
```

The important thing:

> **Closed does not mean the dependency never fails.**

It means:

> **The system currently considers the dependency healthy enough to keep sending requests.**

---

# 8. Detecting Failure

How does the circuit breaker decide that a dependency is unhealthy?

There are different strategies.

For example:

### Failure count

```text
5 failures
within some period
```

might trigger the circuit.

---

### Failure percentage

Suppose:

```text
100 requests
60 failures
```

Failure rate:

```text
60%
```

The system might decide this is unhealthy.

---

### Consecutive failures

For example:

```text
Failure
Failure
Failure
Failure
Failure
```

could trigger the circuit.

---

### Time-based windows

The breaker might evaluate behavior over a recent time window:

```text
Last 10 seconds
```

rather than using lifetime statistics.

The exact algorithm is implementation-specific.

At the architecture level, the important idea is:

> **The circuit breaker observes recent dependency behavior and decides when failure is severe enough to stop sending traffic.**

---

# 9. State 2 — Open

Now suppose Payment Service starts failing badly.

```text
100 requests
70 failures
```

The circuit breaker decides:

```text
Payment Service is unhealthy.
```

It changes state:

```text
CLOSED
   │
   │ failure threshold exceeded
   ▼
OPEN
```

Now something fundamentally changes.

Before:

```text
Request
   │
   ▼
Circuit
   │
   ▼
Payment Service
```

After:

```text
Request
   │
   ▼
Circuit
   │
   X
   │
   ▼
Fail Fast
```

The request does **not** call Payment Service.

This is the defining behavior of the open state.

---

# 10. What Does "Fail Fast" Mean?

Suppose a request arrives while the circuit is open.

Without a circuit breaker:

```text
Request
   │
   ▼
Payment Service
   │
   ▼
Wait
   │
   ▼
Timeout
```

With an open circuit:

```text
Request
   │
   ▼
Circuit Breaker
   │
   X
   ▼
Immediate failure
```

The system does not waste time waiting.

This is called **fail fast**.

Depending on the business operation, the application might then:

- return an error
- use a fallback
- queue the operation
- degrade functionality
- tell the user to try later

The circuit breaker itself does not decide the business fallback.

It primarily says:

> **"Do not call this dependency right now."**

---

# 11. Why Open the Circuit?

This is the key question.

Why not just continue retrying?

Because once we have strong evidence that a dependency is unhealthy, continuing to send requests has little benefit and potentially significant cost.

Compare:

### Without Circuit Breaker

```text
Dependency unhealthy
        │
        ▼
Keep calling
        │
        ▼
Timeouts
        │
        ▼
Retries
        │
        ▼
Resource consumption
        │
        ▼
Caller becomes unhealthy
```

### With Circuit Breaker

```text
Dependency unhealthy
        │
        ▼
Detect repeated failures
        │
        ▼
Open circuit
        │
        ▼
Fail fast
        │
        ▼
Protect caller resources
```

So the circuit breaker creates **failure isolation**.

---

# 12. State 3 — Half-Open

Now we have another problem.

Suppose Payment Service was down at:

```text
12:00
```

Circuit opens.

```text
OPEN
```

Should we leave it open forever?

No.

The dependency may recover.

We need to periodically test it.

After some waiting period:

```text
OPEN
  │
  │ recovery wait
  ▼
HALF-OPEN
```

The half-open state means:

> **"We think the dependency might have recovered. Let's cautiously test it."**

For example:

```text
Circuit = HALF-OPEN

Allow a small number of requests
            │
            ▼
       Payment Service
```

If the test succeeds:

```text
HALF-OPEN
     │
     │ success
     ▼
 CLOSED
```

Traffic resumes normally.

---

# 13. What If the Test Fails?

Suppose:

```text
HALF-OPEN
    │
    ▼
Test request
    │
    X
    ▼
Failure
```

The dependency is apparently still unhealthy.

So:

```text
HALF-OPEN
     │
     │ failure
     ▼
   OPEN
```

And we stop sending normal traffic again.

So the complete lifecycle is:

```text
             failures
        ┌───────────────┐
        │               ▼
     CLOSED ───────► OPEN
        ▲               │
        │               │ wait
        │               ▼
        └──────── HALF-OPEN
             success
```

Or more intuitively:

```text
CLOSED
  │
  │ "Everything looks okay."
  │
  ▼
OPEN
  │
  │ "Something is seriously wrong."
  │
  ▼
HALF-OPEN
  │
  │ "Let's carefully test recovery."
  │
  ├── Success ──► CLOSED
  │
  └── Failure ──► OPEN
```

---

# 14. Why Not Immediately Close After One Successful Request?

Because one successful request does not necessarily mean the dependency has fully recovered.

Imagine Payment Service was receiving:

```text
10,000 requests/sec
```

After an outage, it comes back.

One test request succeeds.

Does that prove it can handle:

```text
10,000 requests/sec
```

Not necessarily.

The service might still be recovering.

Therefore systems may gradually restore traffic rather than immediately flooding the dependency.

Conceptually:

```text
Half-open
    │
    ▼
Small test
    │
 Success
    ▼
More traffic
    │
 Success
    ▼
More traffic
    │
 Success
    ▼
Normal traffic
```

The exact strategy depends on the implementation and architecture.

The broader idea is:

> **Recovery should also be controlled.**

---

# 15. Circuit Breaker vs Retry

This distinction is extremely important in interviews.

They solve related but different problems.

### Retry

Asks:

> **"This particular attempt failed. Should I try again?"**

```text
Request
   │
   ▼
Failure
   │
   ▼
Backoff
   │
   ▼
Retry
```

---

### Circuit Breaker

Asks:

> **"This dependency appears unhealthy. Should I even attempt this call?"**

```text
Many failures
      │
      ▼
Circuit opens
      │
      ▼
Don't call dependency
```

So:

```text
Retry
=
Recover from potentially transient individual failures

Circuit Breaker
=
Protect the caller from a persistently unhealthy dependency
```

They can work together.

---

# 16. Retry + Circuit Breaker

A realistic conceptual flow might look like:

```text
                 Request
                    │
                    ▼
             Circuit Breaker
                    │
             ┌──────┴──────┐
             │             │
          CLOSED         OPEN
             │             │
             ▼             ▼
          Attempt        Fail Fast
             │
             ▼
          Timeout
             │
             ▼
       Retry if allowed
             │
             ▼
       Repeated failures
             │
             ▼
       Open circuit
```

This is an important distinction:

> **Retry handles individual failures; circuit breaking handles persistent dependency failure.**

---

# 17. A Concrete Example

Suppose an Order Service calls Payment Service.

Circuit begins:

```text
CLOSED
```

Requests arrive:

```text
Request 1 → Success
Request 2 → Success
Request 3 → Success
Request 4 → Timeout
Request 5 → Success
Request 6 → Success
```

No problem.

Circuit stays:

```text
CLOSED
```

Now Payment Service experiences a major problem.

```text
Request 7 → Timeout
Request 8 → Timeout
Request 9 → Timeout
Request 10 → Timeout
Request 11 → Timeout
```

Failure threshold is reached.

```text
CLOSED
   │
   ▼
OPEN
```

Now:

```text
Request 12
    │
    ▼
Circuit Breaker
    │
    X
    ▼
Fail Fast
```

No call is made.

Same for:

```text
Request 13 → Fail Fast
Request 14 → Fail Fast
Request 15 → Fail Fast
```

Payment Service gets breathing room.

After some period:

```text
OPEN
  │
  ▼
HALF-OPEN
```

A small test is allowed.

```text
Test → Success
```

Circuit becomes:

```text
CLOSED
```

Normal traffic resumes.

---

# 18. Cascading Failure

This is one of the most important reasons circuit breakers exist.

Imagine:

```text
              Payment
                │
                X
                │
User → API → Order
             │
             X
             │
          Inventory
```

Suppose Payment becomes extremely slow.

Order Service waits for Payment.

Its threads become occupied.

```text
Order Service

Worker 1 → waiting for Payment
Worker 2 → waiting for Payment
Worker 3 → waiting for Payment
...
Worker N → waiting for Payment
```

Soon:

```text
Order Service
      │
      ▼
No available workers
```

Now users cannot even perform operations that don't necessarily depend on the failing component.

This is the essence of cascading failure:

> **One unhealthy component consumes enough shared resources to make otherwise healthy components unhealthy too.**

Circuit breakers help by cutting off the failing dependency.

```text
Payment unhealthy
      │
      ▼
Circuit opens
      │
      ▼
Order Service stops making calls
      │
      ▼
Order Service preserves resources
```

But there is an important limitation:

> **Circuit breakers protect the caller from a dependency; they don't repair the dependency itself.**

---

# 19. Circuit Breaker Doesn't Mean "Reject Everything"

This is a subtle but important point.

Suppose:

```text
Payment Service
```

is unhealthy.

The circuit for **Payment Service calls** opens.

That does not necessarily mean:

```text
Entire Order Service = unavailable
```

Instead:

```text
Order Service
   │
   ├── Payment → Circuit OPEN
   │
   ├── Inventory → Circuit CLOSED
   │
   └── Catalog → Circuit CLOSED
```

The Order Service may continue serving functionality that does not depend on Payment.

This is an important reliability principle:

> **Isolate failure rather than allowing one dependency to determine the health of the entire system.**

---

# 20. Fallbacks

Once the circuit is open, the application has choices.

For example:

```text
Circuit OPEN
     │
     ▼
Fail Fast
     │
     ├── Return error
     │
     ├── Use cached data
     │
     ├── Queue operation
     │
     └── Degrade functionality
```

Consider a recommendation system.

```text
Product Page
     │
     ├── Product data
     │
     └── Recommendation Service
```

If recommendations are unavailable, perhaps we can still show the product.

```text
Product page
     │
     ├── Product information ✓
     │
     └── Recommendations ✗
```

The user gets a degraded experience rather than a completely failed page.

This is called **graceful degradation**.

But again:

> The circuit breaker doesn't automatically provide the fallback. The application architecture must decide what degradation is acceptable.

---

# 21. Circuit Breaker Thresholds

A circuit breaker needs some way to decide:

> "Enough failures have occurred. Open the circuit."

Possible signals include:

```text
Failure count
Failure percentage
Consecutive failures
Timeout rate
Latency threshold
```

For example:

```text
Last 100 requests

Success = 95
Failure = 5
```

Maybe the circuit remains closed.

But:

```text
Success = 30
Failure = 70
```

could trigger the circuit.

The important thing is not memorizing a particular threshold.

There is no universal:

```text
"Open after exactly 5 failures."
```

Instead, thresholds should reflect:

- dependency behavior
- traffic volume
- acceptable failure rate
- recovery characteristics
- business requirements

---

# 22. What About Latency?

A dependency doesn't necessarily have to return errors to be unhealthy.

Suppose:

```text
Payment Service

99% requests → 100 ms
1% requests → 20 seconds
```

Technically, most requests succeed.

But those extremely slow requests can still consume resources.

A circuit breaker may therefore consider **latency** as part of its health signal.

For example:

```text
Dependency latency becomes abnormally high
          │
          ▼
Consider dependency unhealthy
```

This is a useful connection to our previous chapter.

We learned:

> **Slow can be as dangerous as failed.**

Circuit breakers can incorporate that insight.

---

# 23. Before vs After Architecture

## Before Circuit Breaker

```text
User
 │
 ▼
Order Service
 │
 ├── Request ─────► Payment
 │                    X
 │
 ├── Retry ────────► Payment
 │                    X
 │
 ├── Retry ────────► Payment
 │                    X
 │
 └── Retry ────────► Payment
                      X
```

Result:

```text
Payment failure
      │
      ▼
Repeated attempts
      │
      ▼
Order resources consumed
      │
      ▼
Potential cascading failure
```

---

## After Circuit Breaker

```text
User
 │
 ▼
Order Service
 │
 ▼
Circuit Breaker
 │
 ├── CLOSED ─────► Payment
 │
 │                 X
 │
 │       failures exceed threshold
 │                 │
 │                 ▼
 │               OPEN
 │
 └── OPEN ───────► Fail Fast
```

Now:

```text
Payment failure
      │
      ▼
Circuit opens
      │
      ▼
Calls fail immediately
      │
      ▼
Caller resources protected
```

---

# 24. Where It Helps

Circuit breakers are particularly useful when:

### 1. A service depends on another service

```text
A → B
```

If B becomes unhealthy, A can isolate itself.

---

### 2. A dependency is expensive to call

For example:

```text
External API
```

Repeated failures can consume network and connection resources.

---

### 3. The dependency can remain unhealthy for a while

If an outage lasts minutes rather than milliseconds, continuously retrying is wasteful.

---

### 4. The caller has meaningful fallback behavior

For example:

```text
Recommendation Service unavailable
       │
       ▼
Show product without recommendations
```

Circuit breakers become particularly powerful when graceful degradation is possible.

---

# 25. Where It Doesn't Help

Circuit breakers are not a universal solution.

### 1. They don't fix the underlying dependency

They only reduce traffic toward it.

---

### 2. They can reject legitimate requests

Suppose the breaker opens because of a temporary spike in failures.

Some subsequent requests may have succeeded, but they are rejected anyway.

That's a tradeoff.

---

### 3. Bad thresholds can cause instability

If the breaker opens and closes too aggressively:

```text
CLOSED
  ↓
OPEN
  ↓
CLOSED
  ↓
OPEN
  ↓
CLOSED
```

the system can become unstable.

This behavior is sometimes described as **flapping**.

---

### 4. They don't replace timeouts

Without timeouts, determining whether a dependency is unhealthy may itself take too long.

Circuit breakers and timeouts solve different layers of the problem.

---

### 5. They don't replace retry strategy

Some failures are transient and worth retrying.

A circuit breaker alone doesn't solve that.

---

# 26. Mental Model

Think about a restaurant with a supplier.

```text
Restaurant
    │
    ▼
Food Supplier
```

Normally:

```text
Restaurant orders ingredients
        │
        ▼
Supplier delivers
```

One day the supplier's warehouse has a major problem.

Orders keep getting delayed.

If the restaurant keeps calling every five minutes:

```text
Call
Call
Call
Call
Call
...
```

the restaurant wastes staff time and phone capacity.

Instead, after enough failed attempts, the restaurant decides:

> "We're going to stop calling this supplier for a while."

That's:

```text
Circuit OPEN
```

After some time:

> "Let's make one small test order."

That's:

```text
HALF-OPEN
```

If the supplier delivers successfully:

```text
Circuit CLOSED
```

If it fails again:

```text
Circuit OPEN
```

The restaurant didn't repair the supplier.

It simply prevented the supplier's failure from consuming all of its own resources.

That's exactly the intuition behind a circuit breaker.

---

# 27. Tradeoffs

## Advantages

### 1. Prevents repeated calls to unhealthy dependencies

The caller stops wasting resources.

### 2. Enables fast failure

Requests don't need to wait for repeated timeouts.

### 3. Helps prevent cascading failures

Failure is contained within a smaller part of the architecture.

### 4. Gives dependencies time to recover

Traffic can temporarily decrease.

### 5. Supports graceful degradation

Applications can use fallbacks when appropriate.

---

## Disadvantages

### 1. Adds complexity

The system must track dependency health and circuit state.

### 2. Can reject requests unnecessarily

Some requests might succeed even while the circuit is open.

### 3. Threshold tuning is difficult

Poor configuration can lead to either:

```text
Too tolerant → dependency keeps hurting caller
```

or:

```text
Too aggressive → circuit opens unnecessarily
```

### 4. Doesn't repair dependencies

It is a protective mechanism, not a recovery mechanism.

### 5. Can interact badly with other mechanisms

You need to reason about:

```text
Timeouts
Retries
Backoff
Circuit Breakers
```

together.

---

# 28. Common Interview Questions

## "What problem does a circuit breaker solve?"

It prevents a caller from repeatedly sending requests to an unhealthy dependency, helping protect caller resources and prevent cascading failures.

---

## "What are the three states?"

```text
CLOSED
OPEN
HALF-OPEN
```

**Closed:** normal traffic.

**Open:** calls are rejected/fail fast.

**Half-open:** limited requests test whether the dependency has recovered.

---

## "What's the difference between retry and circuit breaker?"

Retry asks:

> "Should I try this failed operation again?"

Circuit breaker asks:

> "Should I call this dependency at all right now?"

---

## "Why do we need half-open?"

Because leaving the circuit open forever would prevent the system from discovering recovery.

Half-open provides a controlled way to test the dependency.

---

## "Can a circuit breaker prevent cascading failures?"

Yes, it can help by preventing a failing dependency from consuming all of the caller's resources.

But it is one part of a broader reliability strategy.

---

## "Does a circuit breaker replace a timeout?"

No.

Timeout:

```text
How long do I wait?
```

Circuit breaker:

```text
Should I make this call at all?
```

They complement each other.

---

## "What happens if the dependency recovers while the circuit is open?"

The circuit doesn't necessarily know immediately.

After a configured recovery period, it moves to:

```text
HALF-OPEN
```

and allows limited test traffic.

If successful:

```text
HALF-OPEN → CLOSED
```

---

## "Can circuit breakers make things worse?"

Yes.

An incorrectly configured breaker can:

- open too aggressively
- reject healthy traffic
- flap between states
- hide useful recovery signals

Reliability mechanisms themselves must be designed carefully.

---

# 29. The Deeper Engineering Lesson

There is a bigger idea hiding underneath the circuit breaker pattern.

Consider what we've learned so far:

### Timeout

```text
"I won't wait forever."
```

### Retry

```text
"I'll give the operation another chance."
```

### Backoff

```text
"I'll wait before trying again."
```

### Jitter

```text
"I won't synchronize my retries with everyone else."
```

### Circuit Breaker

```text
"I have enough evidence that this dependency is unhealthy.
I'm going to stop calling it for now."
```

Notice the progression.

We're gradually moving from **recovering from individual failures** toward **protecting the entire system from failure propagation**.

```text
Individual request
       │
       ▼
Timeout
       │
       ▼
Retry
       │
       ▼
Backoff + Jitter
       │
       ▼
Persistent failure
       │
       ▼
Circuit Breaker
       │
       ▼
Failure isolation
```

This is a fundamental reliability mindset:

> **Don't just make individual requests more resilient. Make the architecture resilient to failure propagation.**

---

# 30. Connections

Now we reach the next problem.

Suppose an Order Service depends on three things:

```text
Order Service
    │
    ├────────► Payment Service
    │
    ├────────► Inventory Service
    │
    └────────► Recommendation Service
```

Payment Service is failing.

The circuit breaker opens:

```text
Payment → OPEN
```

Great.

But what if the Order Service has a shared resource pool?

For example:

```text
Order Service
     │
     ▼
100 worker threads
```

Now suppose Payment requests consume many workers.

Even with a circuit breaker, **before the breaker opens**, those requests may already consume resources.

Or perhaps the breaker hasn't detected the problem yet.

And what if Recommendation Service also starts becoming slow?

Now multiple dependencies compete for the same resources.

```text
                 100 workers
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
    Payment       Inventory    Recommendation
       X              X              Slow
```

One dependency can still potentially consume resources needed by others.

So we need another reliability principle:

> **How do we isolate resources so that failure in one part of the system cannot consume everything?**

That leads naturally to:

```text
Timeout
   │
   ▼
Retry + Backoff
   │
   ▼
Circuit Breaker
   │
   ▼
Failure isolation at dependency level
   │
   ▼
But what about shared resources?
   │
   ▼
Bulkhead
```

# 31. Key Takeaways

- A circuit breaker protects a caller from a **persistently unhealthy dependency**.
- It prevents unnecessary calls once there is strong evidence of failure.
- The classic states are:
  - **Closed** — requests flow normally.
  - **Open** — requests fail fast without calling the dependency.
  - **Half-open** — limited traffic tests recovery.

- Retry and circuit breaker solve different problems.
- Retry helps with potentially transient failures.
- Circuit breaker handles persistent dependency failure.
- Circuit breakers can help prevent **cascading failures**.
- They can allow graceful degradation when fallback behavior exists.
- They do not repair the unhealthy dependency.
- They do not replace timeouts or retry policies.
- Thresholds must be chosen carefully.
- Recovery should also be controlled rather than immediately restoring full traffic.

The reliability story is now becoming:

```text
                    Failure
                       │
                       ▼
                    Timeout
                       │
                       ▼
                  Should retry?
                       │
                       ▼
              Retry + Backoff
                       │
                       ▼
              Still failing?
                       │
                       ▼
              Circuit Breaker
                       │
                       ▼
             Stop the traffic
                       │
                       ▼
           Protect the caller
```

But there is still a deeper problem.

**What if one dependency can consume so many shared resources that it starves everything else before we even get a chance to protect ourselves?**

That is the problem behind the next chapter:

# **Chapter 4 — Bulkhead**
