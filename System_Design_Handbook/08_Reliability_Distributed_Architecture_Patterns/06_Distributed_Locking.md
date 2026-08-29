# Module 8 — Chapter 6: Distributed Locking

## 1. Goal

**Ensure that, across multiple machines, only the appropriate process can perform a particular operation on a shared resource at a time.**

---

# 2. The Problem

Let's say we're building an e-commerce system.

A product has:

```text
Product: iPhone
Stock: 1
```

Two users click **Buy** at almost exactly the same time.

Our application has multiple servers:

```text
                 Load Balancer
                  /          \
                 /            \
              Server A      Server B
                 |              |
                 +------┬-------+
                        |
                     Database
                        |
                    Stock = 1
```

The two requests arrive at different servers.

### Server A

Reads:

```text
Stock = 1
```

### Server B

Also reads:

```text
Stock = 1
```

Both conclude:

> "There's one item available. I can purchase it."

Then:

```text
Server A → decrease stock → 0
Server B → decrease stock → -1
```

Or perhaps both create an order before either update becomes visible.

We have just sold **one item twice**.

This is a classic **concurrency problem**.

---

# 3. Why Existing Solutions Fail

We've already learned several mechanisms in Module 8.

Could we use a timeout?

No.

A timeout only answers:

> "How long should I wait?"

It doesn't prevent two processes from entering the critical section simultaneously.

---

Could we use retry?

No.

Retry helps recover from transient failures.

It doesn't establish ownership.

---

Could we use a circuit breaker?

No.

There's no unhealthy dependency here.

Both servers are working perfectly.

---

Could we use rate limiting?

Still no.

Rate limiting might reduce:

```text
1000 requests/sec
```

to:

```text
100 requests/sec
```

but two requests can still arrive simultaneously.

We need something fundamentally different.

We need **coordination**.

---

# 4. The Big Idea

> **A distributed lock gives one process temporary ownership of a shared resource so competing processes don't perform the same critical operation simultaneously.**

Conceptually:

```text
Server A ───┐
            │
            v
       [ Distributed Lock ]
            │
            v
        Resource
```

If Server A acquires the lock:

```text
Server A → LOCK → Resource
Server B → WAIT / FAIL
```

Once A finishes:

```text
Server A
   |
   | unlock
   v
[Lock released]
   |
   v
Server B → can acquire
```

The crucial word is **distributed**.

A normal in-memory lock works only inside one process.

A distributed lock must coordinate **across multiple machines/processes**.

---

# 5. First Understand a Normal Lock

Before distributed locking, imagine a single application process.

We have:

```text
Thread A
Thread B
Thread C
   |
   v
Shared Resource
```

Suppose only one thread should modify it at a time.

We can use a lock:

```text
        Lock
         |
    +----+----+
    |         |
 Thread A   Thread B
    |
 acquires
    |
 critical section
    |
 releases
    |
         Thread B
         acquires
```

This works because all threads can coordinate through memory managed by the same process.

But now:

```text
Process A                 Process B
   |                         |
 Thread 1                  Thread 2
   |                         |
   +---------- ??? ----------+
              |
         Shared Resource
```

They don't share the same memory.

Process A's local lock is invisible to Process B.

---

# 6. Why a Local Lock Doesn't Work

Suppose we write:

```text
Server A:

lock(resource) {
    updateResource();
}
```

and Server B does the same.

We actually have:

```text
Server A memory          Server B memory

    Lock A                   Lock B
      |                        |
      v                        v
   Server A                 Server B
```

Both locks are independent.

Therefore:

```text
A → acquires Lock A
B → acquires Lock B
```

Both believe they have exclusive access.

So:

> **A local lock provides mutual exclusion within one process, not across a distributed system.**

---

# 7. What Does a Distributed Lock Need to Guarantee?

A useful distributed lock generally needs several properties.

## Mutual exclusion

At a given moment, we don't want:

```text
A → owns lock
B → owns lock
```

simultaneously.

We want:

```text
A → owns lock
B → doesn't
```

---

## Ownership

The system needs to know **who acquired the lock**.

For example:

```text
Lock:
resource = order:123
owner    = server-A
```

Why?

Because otherwise Server B might accidentally release A's lock.

---

## Expiration

This is extremely important.

Imagine:

```text
Server A
   |
   | acquires lock
   |
   v
Critical section
```

Then Server A crashes.

If the lock remains forever:

```text
Lock = permanently held
```

Nobody else can proceed.

Therefore distributed locks commonly have some form of **lease/expiration**:

```text
Lock acquired
     |
     v
valid for 30 seconds
     |
     v
expires unless renewed
```

This prevents a crashed process from holding the resource indefinitely.

---

# 8. The Lease Concept

A distributed lock is often better thought of as:

> **"You have ownership for this limited period."**

rather than:

> "You own this forever until you explicitly unlock."

For example:

```text
Server A acquires lock

12:00:00
    |
    | valid for 30 sec
    |
12:00:30
    |
    v
expires
```

If A is still working at:

```text
12:00:29
```

it might need to renew its lease.

```text
A
|
| renew
v
Lock
```

This introduces another interesting failure scenario we'll come back to.

---

# 9. The Critical Section

The actual operation protected by the lock is called the **critical section**.

For our inventory example:

```text
Acquire lock
      |
      v
Read stock
      |
      v
Check stock > 0
      |
      v
Decrease stock
      |
      v
Create order
      |
      v
Release lock
```

The goal is:

```text
Only ONE server
       |
       v
critical section
       |
       v
at a time
```

---

# 10. Example: Job Processing

Another very common use case.

Suppose we have:

```text
Payment processing job
```

and three application servers:

```text
Server A ─┐
Server B ─┼──→ Job Queue
Server C ─┘
```

Suppose all three see:

```text
Job #123 = pending
```

Without coordination:

```text
A → process #123
B → process #123
C → process #123
```

We could charge the customer three times.

With a distributed lock:

```text
             Job #123
                 |
       +---------+---------+
       |         |         |
       A         B         C
       |         |         |
       +---------+---------+
                 |
          Distributed Lock
```

Only one gets ownership:

```text
A → acquire → success
B → acquire → fail
C → acquire → fail
```

Then:

```text
A → process job
A → release lock
```

---

# 11. Another Use Case: Scheduled Jobs

Imagine every application server runs:

```text
cleanupExpiredOrders()
```

every minute.

With 20 servers:

```text
20 servers
    |
    +-- run same job
    +-- run same job
    +-- run same job
    ...
```

We may only want one server to perform the job.

A distributed lock can provide:

```text
             Scheduled Job
                   |
             Acquire Lock
                   |
          +--------+--------+
          |                 |
       Server A          Servers B–T
       succeeds            don't run
          |
          v
       Execute
```

This pattern is often called **leader-like job execution** or a **distributed scheduler pattern**, depending on the broader architecture.

---

# 12. How Do We Implement One?

At the conceptual level, we need some **shared coordination mechanism**.

```text
Server A ─┐
Server B ─┼──→ Shared Lock State
Server C ─┘
```

The shared system maintains something like:

```text
resource: inventory:123
owner: server-A
expiration: 12:30:30
```

Server A asks:

> "Can I acquire the lock for inventory:123?"

If nobody owns it:

```text
YES
```

If someone already owns it:

```text
NO
```

The exact technologies that provide this capability are something we'll study later.

For now, think only in terms of **shared coordination state**.

---

# 13. The Acquire Operation Must Be Atomic

Here's a subtle but extremely important point.

Imagine this naive approach:

```text
1. Check whether lock exists
2. If not, create lock
```

Two servers can do:

```text
Server A                  Server B

check lock → absent       check lock → absent
create lock               create lock
```

Now both think they won.

That's broken.

We need the operation to behave like:

> **"Create this lock only if nobody has already created it."**

as one indivisible operation.

Conceptually:

```text
if lock does not exist:
    create lock
    assign owner
    expiration
```

must happen atomically.

This is one of the core requirements of distributed coordination.

---

# 14. Lock Ownership

Suppose:

```text
Server A
   |
   | acquires lock
   v
Lock owner = A
```

Then A becomes slow.

The lease expires.

Server B acquires:

```text
Lock owner = B
```

Now imagine A wakes up and says:

```text
"Great, I'm done. I'll release the lock."
```

If A blindly deletes the lock:

```text
A → delete lock
```

it could accidentally release **B's lock**.

That's why lock ownership matters.

A safer conceptual operation is:

```text
Release lock only if owner == me
```

So:

```text
A → release(lock, owner=A)
```

If current owner is B:

```text
reject
```

This is a subtle detail that interviewers like because it exposes whether you understand distributed locking beyond the simple idea of "put a lock somewhere."

---

# 15. The Most Dangerous Problem: Lock Expiration

Let's examine this carefully.

Suppose:

```text
Lease = 10 seconds
```

Server A acquires the lock:

```text
t=0
A → lock
```

Then A starts a long operation.

```text
t=1
A → processing

t=5
A → processing

t=10
LOCK EXPIRES
```

Server B acquires it:

```text
t=11
B → lock
B → processing
```

But A is still running:

```text
A → processing
```

Now we have:

```text
A → believes it is working
B → believes it is working
```

We've lost mutual exclusion.

This is one of the **fundamental subtleties of distributed locks**.

A lease can prevent permanent deadlocks caused by crashed processes, but it also introduces the possibility that:

> **The original process continues executing after its lease has expired.**

---

# 16. Why This Means a Lock Isn't Magic

Consider:

```text
Acquire lock
    ↓
Do important operation
    ↓
Release lock
```

It is tempting to think:

> "The lock guarantees that only I can modify the resource."

Not necessarily.

The lock only provides coordination according to its guarantees.

If ownership expires while the process is still executing, another process may legitimately acquire the lock.

Therefore critical operations often need **additional safeguards**.

For example, the underlying data operation might itself enforce a condition such as:

```text
"Only update this resource if my ownership/version is still valid."
```

This is where concepts such as **fencing** become important.

---

# 17. Fencing Tokens

Let's introduce the idea conceptually.

Every time a lock is granted, the coordinator gives the owner a monotonically increasing number.

```text
Server A → lock → token 41

Server B → lock → token 42
```

Now the resource can enforce:

```text
Accept operation with token 42
Reject operation with token 41
```

So even if A wakes up after its lease has expired:

```text
A → old token 41
```

the resource knows:

```text
41 < 42
```

and refuses the stale operation.

This is called a **fencing token**.

The important idea is:

> **Don't merely trust that a process still owns the lock; give each ownership period an ordering identity that the protected resource can validate.**

This is one of the deeper concepts in distributed locking.

---

# 18. Distributed Lock vs Database Transaction

This is another very important distinction.

Suppose our stock is in a database.

Instead of using a distributed lock, perhaps the database can perform:

```text
UPDATE product
SET stock = stock - 1
WHERE product_id = 123
  AND stock > 0
```

This operation can be designed so that the check and update happen atomically.

Then perhaps we don't need a distributed lock at all.

This leads to an important rule:

> **Don't reach for distributed locks when the underlying datastore can safely express the invariant atomically.**

For many data-consistency problems, a database transaction or atomic conditional operation can be simpler and safer.

---

# 19. When Do We Actually Need Distributed Locking?

Distributed locking becomes useful when the thing we're coordinating isn't easily protected by a single atomic datastore operation.

For example:

```text
Server A
   |
   +-- call external API
   +-- update multiple systems
   +-- generate unique resource
   +-- perform expensive shared computation
```

We might need coordination across multiple application instances.

Another common example:

```text
Only one worker should perform this expensive operation at a time.
```

A distributed lock can be appropriate.

But always ask:

> **Can I solve this with an atomic operation, transaction, unique constraint, idempotency, or another coordination pattern instead?**

A distributed lock introduces significant complexity.

---

# 20. Distributed Locking vs Idempotency

These two concepts are often confused.

Suppose payment processing receives:

```text
processPayment(order123)
```

### Locking

Says:

> "Only one process should execute this critical section at a time."

### Idempotency

Says:

> "Even if this operation happens more than once, the final effect should be the same."

For example:

```text
Request
   |
   +-- Server A
   |
   +-- Server B
```

With a lock:

```text
A → executes
B → doesn't execute
```

With idempotency:

```text
A → executes payment
B → sees operation already completed
```

The second approach can sometimes be much more robust in distributed systems because duplicates are often unavoidable.

In many real designs, **locking and idempotency complement each other rather than replacing one another**.

---

# 21. Where It Helps

Distributed locks are useful when:

### 1. Only one worker should process something

```text
Job → one worker
```

### 2. A shared resource cannot safely be modified concurrently

```text
Resource
   ↑
one owner
```

### 3. Scheduled jobs must avoid duplicate execution

```text
20 servers
   ↓
one executes
```

### 4. Expensive work should be performed once

For example:

```text
Cache miss
   ↓
many servers
   ↓
only one computes expensive result
```

This can prevent a cache stampede.

---

# 22. Where It Doesn't Help

Distributed locks are **not** a universal solution.

### Don't use them for every database update

A database may already provide safer atomicity.

### Don't use them as a substitute for transactions

A lock doesn't automatically make a multi-step operation atomic.

### Don't assume a lock survives every failure

Network partitions, pauses, crashes, clock issues, lease expiration, and coordinator failures all matter.

### Don't hold locks unnecessarily long

Long-held locks create:

```text
Waiting
  ↓
Timeouts
  ↓
Retries
  ↓
More contention
```

which can make the system worse.

---

# 23. Tradeoffs

## Advantages

### Prevents concurrent conflicting work

Multiple servers can coordinate around one resource.

### Useful for singleton work

Only one worker can perform an operation.

### Can prevent duplicate expensive computation

Useful around cache population and similar workloads.

### Works across processes and machines

Unlike local locks.

---

## Disadvantages

### Distributed coordination is hard

Failures can occur between:

```text
acquire
↓
work
↓
release
```

### Lock contention

Many workers waiting for one lock can become a bottleneck.

### Deadlocks / stuck ownership

Especially if expiration isn't handled correctly.

### Lease expiration problems

The original process may continue running after losing ownership.

### Lock service becomes critical infrastructure

If every operation depends on it:

```text
Lock service failure
       ↓
many operations affected
```

### Performance overhead

Every critical operation may require coordination over the network.

---

# 24. Common Interview Questions

## Q1. Why can't I use a normal mutex across servers?

Because normal mutexes operate on shared local memory.

Different servers don't share that memory.

A distributed lock requires some shared coordination mechanism accessible to all participants.

---

## Q2. Why does a distributed lock need an expiration?

Because the lock holder can crash.

Without expiration:

```text
A → acquire
A → crash
```

and the lock could remain permanently held.

A lease allows another process to eventually recover ownership.

---

## Q3. Why is expiration dangerous?

Because:

```text
A → acquires
A → becomes slow
lease expires
B → acquires
A → continues working
```

Now both can execute.

That's why serious distributed-lock designs may need mechanisms such as **fencing tokens** or validation by the protected resource.

---

## Q4. Why should lock release verify ownership?

Because ownership may have changed.

Example:

```text
A → lock
A → lease expires

B → lock

A → wakes up
A → release
```

Without an ownership check, A could release B's lock.

---

## Q5. Should I always use a distributed lock when multiple servers modify the same data?

**No.**

First look for:

- atomic operations
- transactions
- unique constraints
- optimistic concurrency
- idempotency
- conditional writes

A distributed lock is often more complicated than necessary.

---

## Q6. What's the difference between a distributed lock and leader election?

A distributed lock typically answers:

> **"Who currently owns this resource/critical section?"**

Leader election answers:

> **"Which node should act as the leader for this group of nodes?"**

Leader election is a broader, longer-lived coordination problem.

We'll study it in **Chapter 7**.

---

# 25. Before vs After Architecture

### Without distributed coordination

```text
                 +----------------+
                 | Load Balancer  |
                 +-------+--------+
                         |
              +----------+----------+
              |          |          |
             S1         S2         S3
              |          |          |
              +----------+----------+
                         |
                    Shared Resource

          S1 ──→ modify ←── S2
                       ↑
                      S3

       Multiple servers can enter simultaneously
```

---

### With distributed locking

```text
                 +----------------+
                 | Load Balancer  |
                 +-------+--------+
                         |
              +----------+----------+
              |          |          |
             S1         S2         S3
              |          |          |
              +-----+----+----------+
                    |
              +-------------+
              | Distributed |
              |    Lock     |
              +------+------+
                     |
                     v
               Shared Resource
```

Now:

```text
S1 → acquire → success
S2 → acquire → fail
S3 → acquire → fail

S1 → critical section
S1 → release
```

---

# 26. The Deeper Connection

Look at how Module 8 is progressing:

```text
Timeout
   ↓
Don't wait forever

Retry + Backoff
   ↓
Recover from transient failures

Circuit Breaker
   ↓
Stop sending work to unhealthy dependencies

Bulkhead
   ↓
Isolate resource consumption

Rate Limiting
   ↓
Control incoming traffic

Distributed Locking
   ↓
Coordinate competing workers
```

We're moving from:

> **"How do I protect my system from too much or failed work?"**

toward:

> **"How do multiple machines coordinate when they need to act on the same thing?"**

And that naturally leads to our next chapter.

---

# 27. Connections

## Next: Chapter 7 — Leader Election

Imagine we have:

```text
             10 servers
          / / / | \ \ \
         v v v  v  v v v
       Distributed System
```

We don't merely want one server to lock a particular resource.

We want the cluster to agree:

> **"Server 7 is currently the leader."**

The leader might be responsible for:

- coordinating work
- assigning tasks
- performing scheduled operations
- maintaining cluster state
- making certain decisions

But now we have a harder question:

```text
Leader A
   ↓
crashes
   ↓
Who becomes leader?
```

And even more importantly:

```text
A thinks it is leader
        +
B thinks it is leader
        ↓
Two leaders
```

That is a serious distributed-systems failure.

**Chapter 7 — Leader Election** is where we'll tackle that problem.

---

# 28. Key Takeaways

1. **Distributed locking provides coordination across machines.**
2. A local mutex cannot coordinate independent servers.
3. The goal is generally **mutual exclusion** over a critical section.
4. Distributed locks usually need **ownership** and **expiration/leases**.
5. Lock acquisition must be **atomic**.
6. Lock release should verify that the caller still owns the lock.
7. Lease expiration creates a subtle danger: an old process may continue executing after losing ownership.
8. **Fencing tokens** can protect against stale lock holders.
9. Before using a distributed lock, check whether **transactions, atomic operations, unique constraints, or idempotency** solve the problem more simply.
10. A distributed lock is itself distributed infrastructure and therefore introduces failure modes and overhead.

### One sentence to remember

> **A distributed lock is not "a mutex on another server"; it is a temporary ownership agreement that multiple machines must coordinate around—and that agreement itself can fail.**
