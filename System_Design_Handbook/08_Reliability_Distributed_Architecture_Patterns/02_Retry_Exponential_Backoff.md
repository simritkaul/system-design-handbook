# Chapter 2: Retry with Exponential Backoff

## 1. Goal

Understand **why distributed systems retry failed operations**, why blindly retrying can make an outage much worse, and how **exponential backoff and jitter** make retries safer.

We begin exactly where the previous chapter ended:

```text
Timeout
   │
   ▼
The operation took too long
   │
   ▼
Should we try again?
```

The answer is sometimes **yes**.

But the important engineering question is:

> **How do we retry without overwhelming a system that is already struggling?**

---

# 2. The Problem

Consider a simple architecture:

```text
User
  │
  ▼
Order Service
  │
  ▼
Payment Service
```

The Order Service sends:

```text
"Charge ₹1,000"
```

Normally:

```text
Order Service
      │
      │ Charge request
      ▼
Payment Service
      │
      │ Success
      ▼
Order Service
```

Everything works.

But distributed systems experience temporary failures.

Maybe:

- a network packet was lost
- the Payment Service briefly restarted
- a database connection temporarily failed
- a dependency was momentarily overloaded
- a network route briefly failed
- a transient infrastructure problem occurred

The Order Service sends the request.

```text
Order Service
      │
      │ Charge
      ▼
Payment Service
      X
```

No response arrives.

Eventually:

```text
Timeout
```

Now we have a decision.

Should the Order Service simply give up?

Sometimes that is unnecessary.

If the failure was temporary, trying again might succeed.

So we introduce:

# Retry

```text
Attempt 1
   │
   ▼
Failure
   │
   ▼
Attempt 2
   │
   ▼
Success
```

This is a powerful reliability mechanism.

But it has a dangerous side effect.

---

# 3. Why Existing Solutions Fail

Suppose a service is temporarily overloaded.

Imagine:

```text
100,000 requests
       │
       ▼
Payment Service
       │
       ▼
Overloaded
```

Requests start timing out.

Now imagine every caller follows this logic:

```text
if request fails:
    retry immediately
```

We now get:

```text
100,000 original requests
          │
          ▼
       timeout
          │
          ▼
100,000 retries
```

The service is now receiving approximately:

```text
200,000 requests
```

Instead of helping the service recover, the clients are sending **more traffic precisely when the service is already unhealthy**.

This can produce a vicious cycle:

```text
Service becomes overloaded
        │
        ▼
Requests become slow
        │
        ▼
Requests timeout
        │
        ▼
Clients retry
        │
        ▼
Traffic increases
        │
        ▼
Service becomes even more overloaded
        │
        ▼
More timeouts
        │
        ▼
More retries
        │
        ▼
...
```

This is called a **retry storm**.

And this gives us our first major principle:

> **Retries are not free. A retry is additional load on the system that is already failing.**

So merely adding retries is not enough.

We need to control **when** we retry.

---

# 4. The Big Idea

The fundamental idea is:

> **When a temporary failure occurs, retry the operation, but wait progressively longer between attempts.**

Instead of:

```text
Attempt 1
Failure
Attempt 2 immediately
Failure
Attempt 3 immediately
Failure
Attempt 4 immediately
```

we can do:

```text
Attempt 1
   │
 Failure
   │
 wait
   │
Attempt 2
   │
 Failure
   │
 wait longer
   │
Attempt 3
   │
 Failure
   │
 wait even longer
   │
Attempt 4
```

This is **exponential backoff**.

A conceptual sequence might look like:

```text
Attempt 1 → fail
wait 100 ms

Attempt 2 → fail
wait 200 ms

Attempt 3 → fail
wait 400 ms

Attempt 4 → fail
wait 800 ms
```

The exact numbers aren't important yet.

The important idea is:

```text
Failure
   │
   ▼
Wait
   │
   ▼
Retry
   │
   ▼
Failure
   │
   ▼
Wait longer
   │
   ▼
Retry
```

The system is effectively saying:

> "Maybe the dependency needs a little time to recover. Let's not immediately hit it again."

---

# 5. Why Do We Retry?

Not every failure deserves a retry.

This distinction is extremely important.

Consider:

### Transient failure

```text
Network temporarily unavailable
Service restarting
Temporary overload
Connection temporarily broken
```

Retrying may help.

---

### Permanent failure

```text
Invalid authentication
Invalid request
Insufficient permissions
Resource does not exist
Business rule violation
```

Retrying is unlikely to help.

For example:

```text
POST /payment

Response:
"Card is invalid"
```

Retrying ten times does not make the card valid.

So a good retry strategy begins by asking:

> **Is this failure potentially temporary?**

Conceptually:

```text
Request
   │
   ▼
Failure
   │
   ├───────────────┐
   │               │
Transient       Permanent
   │               │
   ▼               ▼
Retry           Don't retry
```

This is why retry logic should not simply be:

```text
if error:
    retry()
```

The type of failure matters.

---

# 6. Immediate Retry

Before exponential backoff, let's understand the simplest strategy.

Suppose:

```text
Attempt 1 → Failure
```

Immediately:

```text
Attempt 2
```

If that fails:

```text
Attempt 3
```

And so on.

```text
Time ─────────────────────────►

Attempt 1
   │
   X
   │
Attempt 2
   │
   X
   │
Attempt 3
   │
   X
   │
Attempt 4
```

This has one advantage:

> **Low recovery latency.**

If the failure was a tiny transient glitch lasting a few milliseconds, an immediate retry might succeed almost instantly.

But it has a serious weakness.

If the dependency is genuinely struggling:

```text
Attempt 1 → fail
Attempt 2 → immediately fail
Attempt 3 → immediately fail
Attempt 4 → immediately fail
```

we have simply created additional load.

So immediate retries are sometimes useful, but they need to be used carefully.

---

# 7. Exponential Backoff

Now let's introduce the safer strategy.

A simple conceptual formula is:

```text
delay = base × 2^attempt
```

For example, with:

```text
base = 100 ms
```

we might get:

```text
Attempt     Delay before next attempt
─────────────────────────────────────
1           100 ms
2           200 ms
3           400 ms
4           800 ms
5           1600 ms
```

Graphically:

```text
100ms
  │
  └──► 200ms
          │
          └──► 400ms
                   │
                   └──► 800ms
                            │
                            └──► 1600ms
```

The waiting period grows exponentially.

The benefit is that repeated failures cause the client to become progressively less aggressive.

Instead of:

```text
Retry
Retry
Retry
Retry
Retry
```

we get:

```text
Retry
   ↓
wait
Retry
   ↓
wait longer
Retry
   ↓
wait even longer
Retry
```

---

# 8. Why Does Waiting Help?

Imagine the Payment Service has temporarily crashed.

At:

```text
t = 0
```

it is unavailable.

A retry after:

```text
10 ms
```

may accomplish nothing.

But after:

```text
500 ms
```

perhaps:

- the service has restarted
- the database connection pool has recovered
- traffic has decreased
- the failed network route has recovered

The retry now has a better chance of succeeding.

So backoff provides something extremely valuable:

> **Recovery time.**

Instead of continuously hammering the dependency:

```text
Client
 │ │ │ │ │ │ │ │
 ▼ ▼ ▼ ▼ ▼ ▼ ▼ ▼
Service
```

we give it breathing room.

---

# 9. But Exponential Backoff Alone Isn't Enough

Now imagine 1 million clients all use exactly the same retry strategy.

Suppose every client experiences a failure at:

```text
12:00:00
```

They all calculate:

```text
Retry after 100 ms
```

So at:

```text
12:00:00.100
```

they all retry.

The service gets hit with a huge synchronized wave.

```text
                    1M clients
                       │
                       ▼
                   Failure
                       │
                       ▼
                   wait 100ms
                       │
                       ▼
                 ┌─────────────┐
                 │ 1M retries  │
                 └─────────────┘
                       │
                       ▼
                    Service
```

Suppose those retries fail.

Everyone waits:

```text
200 ms
```

Then:

```text
1M retries
```

Again.

This produces synchronized traffic bursts.

This is where **jitter** comes in.

---

# 10. Jitter

Jitter means introducing some randomness into the retry delay.

Instead of:

```text
Every client waits exactly 400 ms
```

we might have:

```text
Client A → 327 ms
Client B → 391 ms
Client C → 438 ms
Client D → 356 ms
Client E → 421 ms
...
```

Now retries are spread across time.

Instead of:

```text
       │
       ▼
████████████
Huge burst
```

we get something more like:

```text
       │
       ▼
██ █████ ██ ███ █ ████ ██
Retries spread over time
```

The goal is not randomness for its own sake.

The goal is:

> **Prevent many independent clients from retrying at exactly the same moment.**

This is why real distributed systems commonly combine:

```text
Retry
   +
Exponential Backoff
   +
Jitter
```

---

# 11. Backoff With Jitter

Conceptually:

```text
base delay
     │
     ▼
exponential growth
     │
     ▼
add randomness
     │
     ▼
actual retry delay
```

For example:

```text
Attempt 1
   │
   ▼
100 ms + random variation

Attempt 2
   │
   ▼
200 ms + random variation

Attempt 3
   │
   ▼
400 ms + random variation
```

The exact jitter algorithm can vary.

At the system-design level, the important thing to remember is:

> **Backoff reduces retry pressure over time; jitter prevents synchronized retry bursts.**

---

# 12. Retry Limits

Should we retry forever?

Absolutely not.

Consider:

```text
Request
   │
   ▼
Failure
   │
   ▼
Retry
   │
   ▼
Failure
   │
   ▼
Retry
   │
   ▼
Failure
   │
   ▼
Retry
   │
   ▼
...
```

Eventually the system needs to stop.

So retry policies usually have a limit.

For example:

```text
Maximum attempts = 3
```

or:

```text
Maximum retry duration = 2 seconds
```

or both.

Then:

```text
Attempt 1 → fail
Attempt 2 → fail
Attempt 3 → fail
                   │
                   ▼
              Stop retrying
```

This matters because retries consume:

- CPU
- memory
- network bandwidth
- connections
- threads
- downstream capacity

A retry is another piece of work.

---

# 13. Retry Budget

Now we can go one level deeper.

Imagine a service is handling:

```text
100,000 requests/sec
```

If every request is allowed to retry three times, the system could potentially create a huge amount of additional traffic.

So large systems often reason about a **retry budget**.

The intuition is:

> **Only allow a controlled amount of additional traffic to come from retries.**

For example, conceptually:

```text
Normal traffic
= 100,000 requests/sec

Allowed retry traffic
= limited percentage of normal traffic
```

The exact implementation can vary.

The principle is more important:

```text
Retries should be bounded.

They should not be allowed to
multiply traffic without control.
```

This is particularly important for high-scale systems.

---

# 14. Retry and Idempotency

Here is where our previous Module 7 concepts become extremely important.

Suppose:

```text
Order Service
      │
      │ Charge ₹1,000
      ▼
Payment Service
```

The Payment Service charges the customer.

But the response is lost.

```text
Order Service
      │
      │ Charge ₹1,000
      ▼
Payment Service
      │
      ▼
Payment succeeds
      │
      X
Response lost
```

The Order Service sees:

```text
Timeout
```

It does not know whether the payment happened.

So it retries:

```text
Order Service
      │
      │ Charge ₹1,000
      ▼
Payment Service
```

Without protection, we could get:

```text
Payment #1 → ₹1,000
Payment #2 → ₹1,000
```

The customer gets charged twice.

This is why:

> **Retries and idempotency are deeply connected.**

A retry strategy is only safe when the operation can tolerate the possibility of duplicate execution.

For example, the caller might send:

```text
Idempotency-Key: abc123
```

Then the Payment Service can recognize:

```text
"I have already processed abc123."
```

and avoid performing the operation twice.

This is one of the most important practical lessons from the combination of Module 7 and Module 8.

---

# 15. Not Every Operation Should Be Retried

Consider different operations.

### Read

```text
GET /user/123
```

If it times out, retrying is often relatively safe.

---

### Create

```text
POST /order
```

Potentially dangerous.

A retry could create:

```text
Order #123
Order #124
```

unless the operation has suitable idempotency protection.

---

### Payment

```text
Charge ₹1,000
```

Extremely important to reason carefully about retries.

The operation may have succeeded even though the response was lost.

---

### Delete

```text
DELETE /user/123
```

This can often be designed to be idempotent:

```text
User exists → delete
User doesn't exist → still desired final state
```

But the actual semantics depend on the API.

So the question isn't simply:

> "Can I retry this request?"

It is:

> **"What happens if this operation executes more than once?"**

---

# 16. Retry Amplification

Let's look at a larger architecture.

```text
User
  │
  ▼
API Service
  │
  ▼
Order Service
  │
  ▼
Payment Service
```

Suppose Payment Service is failing.

If every upstream layer independently retries:

```text
API Service
   │
   ├── retry
   ├── retry
   └── retry
        │
        ▼
Order Service
   │
   ├── retry
   ├── retry
   └── retry
        │
        ▼
Payment Service
```

The number of requests reaching Payment Service can grow dramatically.

This is called **retry amplification**.

The general principle:

> **Retries at multiple layers can multiply one another.**

Therefore retry ownership should be designed carefully.

You don't want every layer saying:

> "If something fails, I'll retry it."

Without coordination, reliability mechanisms themselves can become a source of instability.

---

# 17. Timeout + Retry

Now let's combine the first two chapters.

We started with:

```text
Request
   │
   ▼
Wait
   │
   ▼
Timeout
```

Now:

```text
Request
   │
   ▼
Wait
   │
   ▼
Timeout
   │
   ▼
Is failure transient?
   │
   ├── No ──► Fail
   │
   └── Yes
         │
         ▼
      Retry
         │
         ▼
      Failure?
         │
         ▼
   Exponential Backoff
         │
         ▼
       Retry
```

This gives us a basic reliability loop:

```text
             ┌───────────────┐
             │               │
             ▼               │
        Send request         │
             │               │
             ▼               │
          Wait               │
             │               │
      ┌──────┴──────┐        │
      │             │        │
   Success       Timeout     │
                    │        │
                    ▼        │
              Retryable?     │
                │     │      │
               No    Yes     │
                │     │      │
                ▼     ▼      │
              Fail  Backoff ─┘
```

But we still have a problem.

Imagine the dependency is completely down.

Our system keeps trying.

Even with backoff, many clients may continue sending requests.

Eventually we need a stronger mechanism:

> **If we already know a dependency is failing, why keep sending traffic to it at all?**

That is the problem the next chapter solves.

---

# 18. Before vs After Architecture

## Before: Immediate Retries

```text
Client
  │
  ├──── Request ────► Service
  │                     │
  │                  Failure
  │                     │
  ├──── Retry ───────► Service
  │                     │
  │                  Failure
  │                     │
  ├──── Retry ───────► Service
  │                     │
  │                  Failure
  │                     │
  └──── Retry ───────► Service
```

During an outage:

```text
Failure
  │
  ▼
Immediate retries
  │
  ▼
More load
  │
  ▼
More failures
```

---

## After: Exponential Backoff + Jitter

```text
Client
  │
  ├──── Request ─────► Service
  │                      │
  │                   Failure
  │                      │
  │                   wait
  │                  + jitter
  │                      │
  ├──── Retry ────────► Service
  │                      │
  │                   Failure
  │                      │
  │                 wait longer
  │                  + jitter
  │                      │
  ├──── Retry ────────► Service
```

Now clients progressively reduce their retry pressure.

---

# 19. Real-World Usage

Retries are extremely common in distributed infrastructure.

Consider a large service architecture:

```text
Service A
   │
   ▼
Service B
   │
   ▼
Service C
```

A temporary network or infrastructure failure should not necessarily cause the entire user request to fail immediately.

A controlled retry can improve availability.

For example:

```text
Service A
   │
   │ Request
   ▼
Service B
   X
   │
   │ temporary failure
   ▼
Backoff
   │
   ▼
Retry
   │
   ▼
Service B
   │
   ▼
Success
```

The important thing is that retry policies are typically designed around:

- which errors are retryable
- maximum attempts
- maximum retry duration
- backoff
- jitter
- idempotency
- overall request deadline

The exact mechanisms vary by system and technology.

The architectural principle remains the same.

---

# 20. Where Retry Helps

Retries are particularly useful for **transient failures**.

Examples:

```text
Temporary network failure
Temporary connection failure
Brief service restart
Transient infrastructure problem
Short-lived overload
```

They can significantly improve perceived reliability.

Instead of:

```text
Temporary failure
      │
      ▼
User sees error
```

we can sometimes achieve:

```text
Temporary failure
      │
      ▼
Retry
      │
      ▼
Success
      │
      ▼
User never notices
```

That is powerful.

---

# 21. Where Retry Doesn't Help

Retries are generally poor solutions for permanent failures.

For example:

```text
Invalid request
Unauthorized
Forbidden
Invalid data
Business rule violation
Resource permanently unavailable
```

Retrying these repeatedly just wastes resources.

More importantly, retrying certain operations can produce side effects.

For example:

```text
Create Order
```

could become:

```text
Create Order
Create Order
Create Order
```

if idempotency is not properly designed.

So:

> **Retry is a recovery mechanism, not a universal response to every error.**

---

# 22. Tradeoffs

## Advantages

### 1. Improves resilience to transient failures

Temporary failures can be hidden from users.

### 2. Reduces unnecessary request failures

A single failed attempt does not necessarily mean the entire operation has failed.

### 3. Works well with unreliable networks

Networks are not perfectly reliable.

### 4. Exponential backoff reduces pressure

Clients become progressively less aggressive when failures persist.

### 5. Jitter prevents synchronization

Independent clients are less likely to create synchronized retry waves.

---

## Disadvantages

### 1. Retries increase load

Every retry is additional work.

### 2. Can cause retry storms

Poorly designed retry policies can amplify outages.

### 3. Adds latency

Waiting between retries makes the overall operation take longer.

### 4. Can duplicate side effects

This is especially dangerous for payments, orders, and other writes.

### 5. Adds complexity

You need to reason about:

- retryable errors
- attempts
- backoff
- jitter
- deadlines
- idempotency
- retry budgets

---

# 23. Common Interview Questions

## "Why shouldn't we retry immediately?"

Because the dependency may already be overloaded or recovering.

Immediate retries add more traffic and can make the outage worse.

---

## "What is exponential backoff?"

A strategy where the delay between retries grows progressively, often approximately doubling after each failed attempt.

Conceptually:

```text
100 ms
200 ms
400 ms
800 ms
1600 ms
```

---

## "Why add jitter?"

To prevent many clients from retrying at the same exact time.

Without jitter:

```text
Failure
   │
   ▼
Everyone waits 400ms
   │
   ▼
Everyone retries
```

With jitter:

```text
Failure
   │
   ├── Client A → 350ms
   ├── Client B → 417ms
   ├── Client C → 382ms
   └── Client D → 451ms
```

Traffic becomes more distributed.

---

## "Should we retry every error?"

No.

Retry primarily makes sense for failures that are potentially transient.

Permanent failures should generally fail fast.

---

## "Why is retry dangerous for payment APIs?"

Because the first request may have succeeded even though its response was lost.

A retry could perform the payment again.

Therefore retrying side-effecting operations requires careful idempotency design.

---

## "What is a retry storm?"

A situation where many failed requests generate large numbers of retries, creating additional load and potentially worsening the underlying outage.

---

## "What is retry amplification?"

When retries occur across multiple layers of a distributed system and multiply the amount of traffic reaching a failing dependency.

---

## "What is a retry budget?"

A mechanism or policy that limits how much additional traffic a system is willing to generate through retries.

The goal is to prevent retries from overwhelming the system.

---

# 24. Mental Model

Imagine a restaurant kitchen.

The kitchen is overwhelmed.

```text
100 orders
   │
   ▼
Kitchen
   │
   ▼
Overloaded
```

Now imagine every waiter whose order is delayed repeatedly walks into the kitchen and asks:

> "Is mine ready?"

Every few seconds.

The kitchen gets even less productive.

That is **immediate retry**.

A better system says:

> "The kitchen is busy. Let's wait a little before asking again."

And if the kitchen remains overwhelmed:

```text
Wait
   │
   ▼
Ask again
   │
   ▼
Wait longer
   │
   ▼
Ask again
```

That's **exponential backoff**.

Now imagine 100 waiters all use exactly the same timer.

They all walk into the kitchen simultaneously every 400 ms.

Still bad.

So we add some randomness:

```text
Waiter A → 350 ms
Waiter B → 420 ms
Waiter C → 390 ms
Waiter D → 460 ms
```

Now the kitchen receives requests more gradually.

That's **jitter**.

The analogy captures the entire concept:

```text
Retry
  +
Backoff
  +
Jitter
```

---

# 25. The Deeper Engineering Lesson

The most important thing to understand is that **reliability mechanisms themselves consume resources**.

This is easy to overlook.

We might think:

```text
Service fails
   │
   ▼
Retry
   │
   ▼
More reliable!
```

But the actual system is:

```text
Service fails
   │
   ▼
Retry
   │
   ▼
More traffic
   │
   ▼
More resource consumption
   │
   ▼
Potentially more failures
```

So reliability engineering is not simply:

> "Try harder."

It is:

> **"Recover intelligently without making the failure worse."**

This principle will become even more important in the next chapter.

---

# 26. Connections

We have now built:

```text
Chapter 1
Timeout
   │
   ▼
Stop waiting after a reasonable deadline
```

Then:

```text
Chapter 2
Retry
   │
   ▼
Try again when failure may be temporary
```

And we made retries safer:

```text
Retry
   │
   ├── Exponential Backoff
   │       ↓
   │   Give dependency time to recover
   │
   └── Jitter
           ↓
       Avoid synchronized retry storms
```

But consider a dependency that is completely down.

Suppose:

```text
Payment Service
      │
      ▼
Completely unavailable
```

Every client keeps doing:

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
```

Even if the retry strategy is well designed, we're still sending traffic toward a dependency we already strongly suspect is unhealthy.

So the next question becomes:

> **Can we temporarily stop calling a dependency that is clearly failing, and only test it again after giving it time to recover?**

That leads naturally to:

```text
Timeout
   │
   ▼
Retry + Backoff
   │
   ▼
Dependency keeps failing
   │
   ▼
Stop sending requests temporarily
   │
   ▼
Circuit Breaker
```

# 27. Key Takeaways

- Retries help recover from **transient failures**.
- Not every failure should be retried.
- Immediate retries can overload a struggling dependency.
- **Exponential backoff** progressively increases the waiting time between retries.
- **Jitter** adds randomness so many clients don't retry simultaneously.
- Retry attempts should be bounded.
- Retry budgets help prevent retries from multiplying traffic uncontrollably.
- Retries can create **retry storms** and **retry amplification**.
- Side-effecting operations require careful **idempotency** design before retrying.
- A timeout tells us **when to stop waiting**.
- A retry tells us **whether to try again**.
- Backoff and jitter tell us **how aggressively to try again**.

And now we have reached the next reliability problem:

> **What if the dependency keeps failing?**

We don't want every request to keep discovering that fact independently.

**Next: Module 8, Chapter 3 — Circuit Breaker.**
