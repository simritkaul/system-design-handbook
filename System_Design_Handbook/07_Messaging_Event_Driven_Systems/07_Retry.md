# Chapter 7 — Retry

## Goal

Understand why retries exist, how they improve reliability, why naive retries can make failures worse, and how to design retry strategies that balance **reliability, latency, and system stability**.

---

# 1. The Problem

Imagine an Order Service sending a request to the Payment Service.

```text
Order Service
      │
      │ Process Payment
      ▼
Payment Service
```

Normally:

```text
Request
   │
   ▼
Payment processed ✓
   │
   ▼
Response ✓
```

But distributed systems fail.

The Payment Service may be temporarily overloaded.

```text
Order Service
      │
      ▼
Payment Service ✕
```

Or there may be a short network failure.

```text
Order Service
      │
      ▼
   Network ✕
      │
      ▼
Payment Service
```

Or the service may briefly restart.

```text
Payment Service
      │
      ▼
Restarting...
```

In many of these situations, the failure is **temporary**.

If we simply give up:

```text
Request
   │
   ▼
Failure
   │
   ▼
Give up
```

then a temporary problem becomes a permanent business failure.

The payment was never attempted again.

The order fails.

The user sees an error.

But perhaps the Payment Service would have recovered 500 milliseconds later.

So we need a mechanism that says:

> **This operation failed for now. Try again.**

That mechanism is a **retry**.

---

# 2. Why Existing Solutions Fail

Our simplest system behaves like this:

```text
Request
   │
   ▼
Try once
   │
 ┌─┴───────┐
 │         │
Success   Failure
 │         │
Done      Give up
```

This works only if systems are perfectly reliable.

But real systems experience:

- Temporary network failures.
- Service restarts.
- Short traffic spikes.
- Transient database errors.
- Brief connection failures.
- Temporary resource exhaustion.

Suppose a service restarts for two seconds.

```text
Time

0s ────── Service unavailable
1s ────── Service unavailable
2s ────── Service recovered ✓
```

A request arrives at:

```text
0.5 seconds
```

Without retry:

```text
Request → Failure → User sees error
```

With retry:

```text
Attempt 1 → Failure
       │
       ▼
Wait
       │
       ▼
Attempt 2 → Failure
       │
       ▼
Wait
       │
       ▼
Attempt 3 → Success ✓
```

The system successfully survives a temporary failure.

So retries improve **resilience**.

But retries introduce a new problem.

---

# 3. The Big Idea

A retry means:

> **When an operation fails, attempt it again because the failure may be temporary.**

Conceptually:

```text
Operation
    │
    ▼
Attempt
    │
 ┌──┴──────┐
 │         │
Success   Failure
 │         │
Done      Retry
            │
            ▼
         Attempt again
```

The fundamental assumption is:

> Not every failure is permanent.

A retry gives the system another chance to succeed.

But this raises an important engineering question:

> How quickly should we retry?

The answer turns out to be extremely important.

---

# 4. The Naive Retry

Suppose a request fails.

The application immediately retries.

```text
Attempt 1 → Fail
Attempt 2 → Fail
Attempt 3 → Fail
Attempt 4 → Fail
Attempt 5 → Fail
```

Perhaps all of this happens within a few milliseconds.

Imagine 10,000 clients doing the same thing.

```text
                ┌────────────┐
Client 1 ──────►│            │
Client 2 ──────►│   Service  │
Client 3 ──────►│            │
...             └────────────┘
Client 10,000 ─►
```

The service becomes overloaded.

Requests start failing.

Every client immediately retries.

```text
Failures
   │
   ▼
Immediate retries
   │
   ▼
More requests
   │
   ▼
More overload
   │
   ▼
More failures
   │
   ▼
Even more retries
```

We have created a **retry storm**.

The retry mechanism, which was supposed to improve reliability, is now making the outage worse.

---

# 5. Retry Amplification

Let's understand why this becomes dangerous.

Suppose a service normally receives:

```text
1,000 requests/second
```

The service starts failing.

Every client retries each request three times.

Now:

```text
Original traffic
       │
       ▼
1,000 requests/sec

Retries
       │
       ▼
3,000 additional requests/sec
```

The failing service may now receive:

```text
4,000 requests/sec
```

But it was already struggling at 1,000.

The system becomes trapped in a feedback loop.

```text
Service struggles
      │
      ▼
More failures
      │
      ▼
More retries
      │
      ▼
More traffic
      │
      ▼
Service struggles even more
```

This is why:

> **A retry is additional traffic.**

It is not free.

Every retry consumes:

- Network capacity.
- CPU.
- Memory.
- Database connections.
- Threads or event-loop capacity.
- Queue capacity.

A good retry strategy must therefore help recovery rather than prevent it.

---

# 6. Retry After a Delay

The simplest improvement is:

```text
Failure
   │
   ▼
Wait
   │
   ▼
Retry
```

For example:

```text
Attempt 1 → Fail

Wait 1 second

Attempt 2 → Success ✓
```

The delay gives the failing system time to recover.

Instead of:

```text
Fail → Retry → Retry → Retry → Retry
```

we get:

```text
Fail
 │
 ▼
Wait
 │
 ▼
Retry
```

This already makes the system more stable.

But fixed delays create another problem.

---

# 7. The Problem With Fixed Retry Intervals

Suppose 100,000 clients fail at exactly the same time.

They all use:

```text
Retry after 5 seconds
```

So:

```text
Time 0
──────
100,000 failures

Time 5 seconds
──────────────
100,000 retries
```

The service receives another enormous spike.

```text
Clients
   │
   │ All retry simultaneously
   ▼
Service 💥
```

The clients are synchronized.

This is sometimes called the **thundering herd problem**.

Even though everyone waited, they all waited for the same amount of time.

The solution is to spread retries out.

But before that, we need a better delay strategy.

---

# 8. Exponential Backoff

Instead of retrying with the same delay every time:

```text
1 second
1 second
1 second
1 second
```

we gradually increase the waiting time.

For example:

```text
Attempt 1 → Fail
Wait 1 second

Attempt 2 → Fail
Wait 2 seconds

Attempt 3 → Fail
Wait 4 seconds

Attempt 4 → Fail
Wait 8 seconds
```

The delay grows exponentially.

Conceptually:

```text
Delay = Base Delay × 2^attempt
```

You do not need to memorize the formula.

The important idea is:

> **The longer a failure continues, the less aggressively we retry.**

This gives the failing system more time to recover.

---

## Visualizing Exponential Backoff

```text
Attempt

1 ── Wait 1s
2 ── Wait 2s
3 ── Wait 4s
4 ── Wait 8s
5 ── Wait 16s
```

Instead of generating constant pressure:

```text
Retry Retry Retry Retry Retry
```

the system gradually backs away.

```text
Retry
  │
  └─── wait longer ─── Retry
                         │
                         └──── wait longer ─── Retry
```

This is why it is called **backoff**.

The client progressively backs away from the failing service.

---

# 9. Jitter

Exponential backoff solves part of the synchronization problem.

But imagine every client starts at the same moment.

All of them may still retry at:

```text
1 second
2 seconds
4 seconds
8 seconds
```

They are still synchronized.

So instead of:

```text
Retry after exactly 4 seconds
```

we introduce randomness.

For example:

```text
Client A → Retry after 3.7s
Client B → Retry after 4.2s
Client C → Retry after 3.4s
Client D → Retry after 4.8s
```

This randomness is called **jitter**.

Now retries are distributed over time.

Instead of:

```text
100,000 retries
      │
      ▼
Exactly at 4 seconds
```

we get:

```text
Retries spread across time

3.1s ── some clients
3.5s ── some clients
3.9s ── some clients
4.2s ── some clients
4.7s ── some clients
```

This prevents retry spikes.

So a common robust pattern is:

```text
Retry
   +
Exponential Backoff
   +
Jitter
```

---

# 10. But Should Every Failure Be Retried?

No.

This is extremely important.

Consider:

```text
Create user
```

The server responds:

```text
400 Bad Request
```

The request is malformed.

Retrying the same request will likely produce:

```text
400 Bad Request
```

again.

And again.

And again.

Nothing changed.

This is a **permanent failure**, not a temporary one.

Now consider:

```text
503 Service Unavailable
```

This may be temporary.

A retry could succeed.

So before retrying, ask:

> **Is this failure likely to disappear if I wait and try again?**

---

# 11. Transient vs Permanent Failures

## Transient Failures

These may succeed later.

Examples:

```text
Temporary network failure
Service restarting
Temporary overload
Short database connection issue
Temporary timeout
```

Retries may help.

---

## Permanent Failures

Retrying the same operation usually does not fix these.

Examples:

```text
Invalid request
Missing required field
Invalid credentials
Unsupported operation
Business validation failure
```

Retries usually waste resources.

---

Conceptually:

```text
Failure
   │
   ▼
Is it transient?
   │
 ┌─┴───────┐
 │         │
Yes        No
 │          │
 ▼          ▼
Retry      Return failure
```

The difficulty is that classification is not always perfect.

For example, a timeout is ambiguous.

```text
Timeout
```

Could mean:

```text
Service is overloaded
```

or:

```text
Network temporarily failed
```

or:

```text
The operation actually succeeded, but the response was lost
```

This is where the previous chapter becomes important.

Retries may create duplicates.

So:

> **Retry logic and idempotency must work together.**

---

# 12. Retry + Idempotency

Suppose a payment request times out.

```text
Client
   │
   │ Charge ₹5,000
   ▼
Payment Service
   │
   ▼
Payment processed ✓
   │
   ✕ Response lost
```

The client sees:

```text
Timeout
```

It retries.

Without idempotency:

```text
Retry
   │
   ▼
Charge ₹5,000 again ✕
```

With an idempotency key:

```text
Attempt 1
Key = abc-123
```

and:

```text
Retry
Key = abc-123
```

The service recognizes:

```text
Same logical operation
```

and returns the previous result.

So:

```text
Retry
   +
Idempotency
   =
Safe recovery from uncertain failures
```

This is one of the most important combinations in distributed systems.

---

# 13. Maximum Retry Attempts

We should not retry forever.

Imagine a database is permanently unavailable.

```text
Retry 1
Retry 2
Retry 3
...
Retry 10,000
```

Eventually, the system must decide:

> This operation cannot currently be completed.

So retry policies usually define a maximum number of attempts.

For example:

```text
Attempt 1
Attempt 2
Attempt 3
Attempt 4
Attempt 5

Stop
```

The exact number depends on the use case.

A user-facing request may only have:

```text
1 or 2 retries
```

because the user cannot wait for 30 seconds.

A background job may retry for:

```text
minutes
hours
or even longer
```

because no user is waiting.

This introduces another important idea:

> **Retry policy depends on the latency requirements of the operation.**

---

# 14. Synchronous vs Asynchronous Retries

Retries behave differently depending on the architecture.

## Synchronous Request

```text
User
 │
 ▼
API
 │
 ▼
Payment Service
```

The user is waiting.

The retry budget is limited.

```text
Attempt
   │
Fail
   │
Wait
   │
Retry
   │
Fail
   │
Return error
```

We cannot keep the user waiting indefinitely.

---

## Asynchronous Job

```text
Producer
   │
   ▼
Queue
   │
   ▼
Worker
```

The worker can retry later.

```text
Process
   │
Fail
   │
▼
Wait 1 min
   │
Retry
```

If it fails again:

```text
Wait 5 min
   │
Retry
```

The user does not need to wait for the entire process.

So asynchronous systems allow more flexible retry strategies.

---

# 15. Retry Queues

Instead of having a worker sit idle while waiting:

```text
Worker
   │
   ▼
Process
   │
Fail
   │
Wait 10 minutes...
```

we can move the work somewhere else.

Conceptually:

```text
Main Queue
    │
    ▼
Worker
    │
    ├── Success → Done
    │
    └── Failure
           │
           ▼
       Retry Queue
           │
           │ Wait
           ▼
       Main Queue
```

This allows the worker to continue processing other messages.

Later, the failed message becomes available again.

```text
Message
   │
   ▼
Retry after delay
   │
   ▼
Process again
```

The exact implementation varies, but the architectural idea is:

> **Separate failed work from immediately processable work.**

---

# 16. Retry Budget

Imagine a service is failing.

Every request is retried three times.

Traffic multiplies.

A system may therefore limit how much retry traffic it allows.

Conceptually:

```text
Normal traffic
      +
Limited retry traffic
```

Instead of:

```text
Failures
   │
   ▼
Unlimited retries
```

we establish a budget.

For example:

```text
Retry traffic must not exceed
a certain fraction of normal traffic.
```

Once the budget is exhausted:

```text
Do not retry further
```

This prevents retries from consuming all system capacity.

The exact mechanism can vary, but the engineering principle is:

> **Retries must be treated as controlled additional load.**

---

# 17. When Retries Can Make Things Worse

Suppose a database is overloaded.

```text
Application
     │
     ▼
Database ✕ overloaded
```

Requests fail.

Applications retry.

```text
More requests
      │
      ▼
Database becomes more overloaded
```

Now failure duration increases.

Without retries:

```text
Failure
   │
   ▼
Traffic may reduce
   │
   ▼
Database recovers
```

With aggressive retries:

```text
Failure
   │
   ▼
More traffic
   │
   ▼
More failure
   │
   ▼
Even more traffic
```

This is why retry strategies are closely connected to:

- Circuit breakers.
- Rate limiting.
- Bulkheads.
- Load shedding.
- Backpressure.

We will study these reliability patterns later.

---

# 18. Real-World Usage

Retries appear almost everywhere.

## API Calls

```text
Service A
   │
   ▼
Service B
```

Temporary network failure:

```text
Retry with backoff
```

---

## Database Connections

A database may temporarily reject a connection.

```text
Connect
   │
   ▼
Failure
   │
   ▼
Wait
   │
   ▼
Reconnect
```

---

## Message Processing

```text
Queue
   │
   ▼
Consumer
   │
   ├── Success → ACK
   │
   └── Failure → Retry later
```

---

## Cloud Infrastructure

Temporary failures can happen when:

- A machine restarts.
- A service is being deployed.
- A network path changes.
- Capacity is temporarily constrained.

Retries help applications survive these short disruptions.

---

# 19. Where Retries Help

Retries are useful when:

### Failures Are Temporary

```text
Temporary outage
      ↓
Wait
      ↓
Retry
      ↓
Success
```

### Operations Are Idempotent

Repeated execution is safe.

### The System Has Time

Background processing can often wait longer.

### The Failure Rate Is Low

Occasional retries are manageable.

### The Downstream System Can Recover

Retries give it time rather than continuously increasing pressure.

---

# 20. Where Retries Don't Help

Retries are harmful when:

### The Error Is Permanent

```text
Invalid request
```

Retrying changes nothing.

### The Downstream System Is Already Overloaded

Aggressive retries can amplify the outage.

### The Operation Is Not Safe to Repeat

```text
Charge customer
```

without idempotency can create duplicate charges.

### Latency Is Strict

A user-facing operation may not have time for repeated long waits.

### Failure Is Widespread

Retrying millions of failed requests can create enormous additional traffic.

---

# 21. Mental Model

Imagine calling a restaurant.

The line is busy.

### No Retry

```text
Call once
   │
Busy
   │
Give up
```

You may never get through even though the line becomes free 10 seconds later.

---

### Immediate Retry

```text
Call
Busy
Call
Busy
Call
Busy
Call
Busy
```

You continuously add pressure.

---

### Backoff

```text
Call
Busy

Wait

Call
Busy

Wait longer

Call
Busy

Wait longer
```

You give the restaurant time to handle existing callers.

---

### Jitter

Now imagine thousands of people calling.

If everyone retries after exactly 10 seconds:

```text
10:00:10
────────
Everyone calls again
```

The line becomes overloaded again.

Instead:

```text
Person A → 8.2 sec
Person B → 10.1 sec
Person C → 11.4 sec
Person D → 9.6 sec
```

The calls are spread out.

That is the intuition behind:

> **Exponential backoff with jitter.**

---

# 22. Tradeoffs

## Advantages

### Improves Reliability

Temporary failures do not automatically become permanent failures.

### Handles Transient Problems

Systems can recover without immediate user intervention.

### Useful in Distributed Systems

Short network and infrastructure failures are common.

### Works Well With Asynchronous Processing

Failed work can often be retried later.

---

## Disadvantages

### Can Create Duplicate Operations

Retries must be combined with idempotency when necessary.

### Increases Traffic

Every retry consumes resources.

### Can Cause Retry Storms

Poorly designed retries can worsen outages.

### Increases Latency

Waiting before retries delays completion.

### Requires Failure Classification

Not every error should be retried.

---

# 23. Common Interview Questions

## Why do we need retries?

Because many distributed system failures are temporary.

A retry allows an operation to succeed after a short network failure, restart, or temporary overload.

---

## Why can retries be dangerous?

Retries create additional traffic.

During an outage:

```text
Failures
   ↓
Retries
   ↓
More load
   ↓
More failures
```

This can create a retry storm.

---

## What is exponential backoff?

The retry delay increases after each failure.

Example:

```text
1s
2s
4s
8s
```

The system gradually reduces pressure on the failing dependency.

---

## What is jitter?

Randomness added to retry timing.

Instead of every client retrying simultaneously, retries are spread across time.

---

## Why should retries and idempotency be used together?

A timeout does not always mean the original operation failed.

It may have succeeded while the response was lost.

Retrying without idempotency can therefore create duplicate business effects.

---

## Should every error be retried?

No.

Retry failures that are likely temporary.

Do not blindly retry permanent failures such as invalid requests or business validation errors.

---

## How many times should a request be retried?

There is no universal number.

The retry count depends on:

- User latency requirements.
- Importance of the operation.
- Whether it is synchronous or asynchronous.
- Downstream capacity.
- Failure type.

---

# 24. Before vs After Architecture

## Before: No Retry

```text
Order Service
      │
      ▼
Payment Service
      │
      ✕ Temporary failure

Result:
Order fails
```

---

## After: Controlled Retry

```text
Order Service
      │
      ▼
Payment Service
      │
      ✕
      │
      ▼
Wait
      │
      ▼
Retry
      │
      ▼
Payment Service ✓
```

---

## Better: Controlled Retry With Backoff

```text
Attempt 1
   │
   ✕
   │
Wait 1s
   │
Attempt 2
   │
   ✕
   │
Wait 2s + jitter
   │
Attempt 3
   │
   ✓
```

Now the system is more likely to recover without creating unnecessary pressure.

---

# 25. Connections

We have now solved an important problem.

```text
Temporary failure
      │
      ▼
Retry
```

But what if retries keep failing?

Imagine:

```text
Attempt 1 ✕
Attempt 2 ✕
Attempt 3 ✕
Attempt 4 ✕
Attempt 5 ✕
```

At some point, the message or job may still be important.

We don't necessarily want to simply discard it.

But we also don't want it to block the main processing flow forever.

So the next question is:

> **Where should permanently failing messages go?**

That leads us to:

# Chapter 8 — Dead Letter Queues

---

# 26. Key Takeaways

- Retries exist because many distributed system failures are temporary.
- A retry should not be immediate and unlimited.
- Immediate retries can create retry storms.
- Fixed retry intervals can synchronize clients and create traffic spikes.
- Exponential backoff gradually reduces pressure on a failing dependency.
- Jitter spreads retries across time.
- Not every failure should be retried.
- Retries can create duplicate operations, so they often need idempotency.
- Retry policies should consider latency requirements and downstream capacity.
- A common pattern is:

```text
Retry
   +
Exponential Backoff
   +
Jitter
   +
Maximum Attempts
   +
Idempotency
```

Retries help us recover from temporary failures.

But eventually, some messages will continue failing.

Instead of retrying forever or losing them, we need a place to isolate and inspect those failures.

**Next: Chapter 8 — Dead Letter Queues.**
