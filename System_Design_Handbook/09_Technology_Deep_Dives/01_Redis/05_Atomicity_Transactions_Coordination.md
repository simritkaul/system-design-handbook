# Redis — Part 5: Atomicity, Transactions & Coordination

We've now made Redis much more powerful:

```text
                Redis Cluster
        ┌──────────┼──────────┐
        ▼          ▼          ▼
     Shard A    Shard B    Shard C
        │          │          │
     Replica     Replica     Replica
```

But distributing data creates an important question:

> **If multiple operations need to happen together, how do we prevent another request from seeing or creating an inconsistent state?**

This is where **atomicity and coordination** enter the picture.

---

# 1. The Problem

Consider a simple bank account:

```text
Alice = ₹1000
```

We want to transfer ₹100:

```text
Alice → Bob
```

Conceptually, that's two operations:

```text
1. Alice = Alice - 100
2. Bob   = Bob + 100
```

Now imagine two requests execute concurrently.

```text
Request A:
    Alice -= 100

Request B:
    Alice -= 100
```

If the operations aren't handled correctly, both requests could read the same old value.

For example:

```text
Initial:
Alice = 1000

Request A reads → 1000
Request B reads → 1000

A writes → 900
B writes → 900
```

We expected:

```text
800
```

but got:

```text
900
```

This is the classic concurrency problem we've already encountered in your Java concurrency work.

The important question now is:

> **What guarantees does Redis give us when multiple clients operate concurrently?**

---

# 2. The Big Idea

> **Redis executes individual commands atomically, but combining multiple commands into a larger business operation requires additional coordination.**

That sentence is worth remembering.

For example:

```text
INCR counter
```

is an atomic operation.

But:

```text
GET counter
calculate something
SET counter
```

is **not automatically one atomic operation**.

This distinction is fundamental.

---

# 3. Why Individual Commands Can Be Atomic

Recall Redis's execution model.

At a high level:

```text
Client A ──┐
           │
Client B ──┼──→ Redis command processing
           │
Client C ──┘
```

Redis processes commands in a controlled execution loop.

Suppose we have:

```text
INCR counter
```

Conceptually:

```text
Read counter
     ↓
Increment
     ↓
Write counter
```

The operation isn't interrupted by another Redis command halfway through its execution.

So if:

```text
counter = 10
```

and three clients simultaneously execute:

```text
INCR counter
INCR counter
INCR counter
```

we can end up with:

```text
13
```

rather than multiple clients overwriting each other's result.

This is one reason Redis is so useful for counters and rate limiting.

---

# 4. Atomicity Doesn't Mean "Everything Is a Transaction"

This is where people often misunderstand Redis.

Consider:

```text
GET balance
SET balance balance - 100
```

These are two separate commands.

Redis can execute:

```text
GET
```

then another client's command can execute:

```text
SET
```

before our application sends its next command.

So:

```text
Client A
   │
   ├── GET balance
   │
   │       Client B
   │          │
   │          └── modifies balance
   │
   └── SET balance
```

The fact that each individual command is atomic doesn't make the **workflow** atomic.

This is exactly the distinction we encountered with:

```text
count++
```

in Java.

The whole logical operation:

```text
read → modify → write
```

may consist of multiple steps.

---

# 5. Example: Rate Limiting

Suppose we want:

> Allow at most 100 requests per minute.

A naïve implementation might do:

```text
GET request_count
```

then:

```text
if count < 100:
    SET request_count count + 1
```

Now two requests arrive simultaneously.

```text
Request A → GET → 99
Request B → GET → 99
```

Both conclude:

```text
99 < 100
```

Then both write:

```text
100
```

We've processed two requests even though our logical operation should have allowed only one.

The problem isn't that Redis's `GET` or `SET` is broken.

The problem is:

> **The entire read-check-update sequence wasn't atomic.**

---

# 6. The Better Approach

We want Redis itself to perform the operation:

```text
INCR request_count
```

rather than:

```text
GET
↓
application logic
↓
SET
```

Now each increment happens atomically.

Conceptually:

```text
Request A ──┐
Request B ──┼──→ Redis
Request C ──┘       │
                    ▼
                INCR INCR INCR
```

Redis serializes the individual operations.

This gives us a useful design principle:

> **Whenever possible, prefer a single atomic datastore operation over a read-modify-write sequence in the application.**

This isn't unique to Redis.

It's a general distributed-systems principle.

---

# 7. What If We Need Multiple Operations?

Now imagine we genuinely need:

```text
Operation 1
Operation 2
Operation 3
```

and they must execute as one logical unit.

Redis provides **transactions** for this.

Conceptually:

```text
BEGIN

  Operation 1
  Operation 2
  Operation 3

COMMIT
```

The important property at a high level is:

> The queued commands are executed together without other commands being interleaved between them during execution.

For example:

```text
Client A:

MULTI
SET A 100
SET B 200
EXEC
```

Conceptually:

```text
MULTI
  ↓
Queue commands
  ↓
EXEC
  ↓
Execute them together
```

---

# 8. But Redis Transactions Aren't Database Transactions in Every Sense

This is an important HLD nuance.

When people hear:

> "Transaction"

they may immediately think:

```text
ACID
```

and assume Redis behaves like a relational database transaction.

That's too simplistic.

Redis transactions primarily provide **atomic execution of a group of commands**.

They aren't a magical guarantee that:

```text
10 operations
+
network failures
+
application crashes
+
arbitrary distributed nodes
```

will behave exactly like a fully general relational transaction system.

The exact semantics differ.

So in an interview, don't casually say:

> "Redis transactions give us full ACID transactions."

That's misleading.

Instead:

> **"Redis can group commands for atomic execution, but its transaction model is different from the transactional model of a relational database."**

---

# 9. Optimistic Concurrency

There's another common situation.

Suppose:

```text
balance = 1000
```

Two clients want to modify it.

We don't necessarily want to blindly execute both operations.

We might want:

> "Only update this value if nobody changed it since I last read it."

This is **optimistic concurrency control**.

Conceptually:

```text
Read value
   ↓
Remember version/state
   ↓
Do some work
   ↓
Before writing:
"Has somebody changed it?"
```

If yes:

```text
Abort / retry
```

If no:

```text
Perform update
```

Redis provides mechanisms for this style of coordination.

The important architectural idea is:

> **Instead of locking the data pessimistically, detect conflicting modifications and retry.**

---

# 10. Pessimistic vs Optimistic

This connects directly to concurrency concepts we've already studied.

### Pessimistic

```text
Lock
 ↓
Nobody else modifies it
 ↓
Perform operation
 ↓
Unlock
```

### Optimistic

```text
Read
 ↓
Assume no conflict
 ↓
Check before writing
 ↓
If conflict → retry
```

Neither is universally better.

The workload determines which makes sense.

If conflicts are rare:

```text
Optimistic
    ↓
often efficient
```

If conflicts are common:

```text
Repeated retries
    ↓
may become expensive
```

---

# 11. Redis and Distributed Locks

Now we arrive at another important Redis use case.

Suppose:

```text
Application Server A
Application Server B
```

Both want to perform:

```text
Generate monthly report
```

But we want only one server doing it.

We need a distributed lock:

```text
             Redis
               │
        "report-lock"
               │
       ┌───────┴───────┐
       ▼               ▼
   Server A          Server B
     │                  │
     └── acquire ──┐    │
                   │    └── denied
                   ▼
                executes
```

Redis can be used to coordinate this kind of distributed access.

Again, the important point is not:

> "Redis is a lock."

Rather:

> **Redis provides primitives that can be used to build coordination mechanisms.**

---

# 12. Why Locks Need Expiration

Imagine Server A acquires:

```text
report-lock
```

Then crashes:

```text
Server A 💥
```

If the lock remains forever:

```text
report-lock
   ↓
nobody can acquire it
```

We've created a deadlock-like availability problem.

So distributed locks often need a lease/expiration:

```text
Acquire lock
     ↓
Lock valid for 30 seconds
     ↓
If owner disappears
     ↓
Lock eventually expires
```

This is another reason Redis's TTL capability is useful beyond caching.

---

# 13. But Distributed Locks Are Tricky

Here's where we need to be careful.

It's tempting to say:

> "Just use Redis as a distributed lock and everything is solved."

Not quite.

Distributed locking has difficult questions around:

- node failures
- network partitions
- lock expiration
- client pauses
- delayed messages
- ownership
- failover
- whether the lock is still valid when work completes

For example:

```text
Server A
   ↓
acquires lock
   ↓
gets paused for a long time
   ↓
lock expires
   ↓
Server B acquires lock
```

Now:

```text
A thinks:
"I still own the lock."

B thinks:
"I own the lock."
```

This is why distributed locks deserve careful design.

The important HLD lesson is:

> **A lock is not merely a key with a TTL. The correctness of the protected operation matters.**

---

# 14. Server-Side Execution

Another way to solve multi-step atomic workflows is to move the logic closer to the data.

Instead of:

```text
Application

GET
 ↓
network
 ↓
calculate
 ↓
network
 ↓
SET
```

we can conceptually do:

```text
Application
     │
     │ send operation
     ▼
   Redis
     │
     ├── read
     ├── calculate
     └── write
```

The entire operation executes inside Redis.

This is the role of Redis's **server-side scripting mechanisms**.

At the HLD level, the important idea is:

> **If a multi-step operation must be atomic, executing the entire operation inside Redis can avoid exposing intermediate state to other clients.**

This can be extremely useful.

But it introduces a new tradeoff.

---

# 15. Don't Put Arbitrary Heavy Computation Inside Redis

Remember our earlier execution model:

```text
Redis
   ↓
command processing
```

If you execute a very expensive operation there:

```text
Heavy computation
       ↓
Redis occupied
       ↓
Other commands wait
       ↓
Latency increases
```

So server-side execution should generally be:

```text
Short
+
bounded
+
data-oriented
```

rather than:

```text
Huge computation
+
long-running algorithm
```

This connects directly to what we learned earlier:

> **A fast system can still become slow if one operation monopolizes its execution path.**

---

# 16. Pipelining Is Different

One more distinction that commonly causes confusion.

Suppose we have:

```text
SET A
SET B
SET C
```

Sending these individually means:

```text
Application → Redis
Application → Redis
Application → Redis
```

There are multiple network round trips.

Pipelining lets us send several commands together:

```text
Application
    │
    │ SET A
    │ SET B
    │ SET C
    ▼
  Redis
```

This reduces network overhead.

But:

> **Pipelining does not automatically make the commands one atomic transaction.**

That's a very important distinction.

```text
Pipelining
   → network efficiency

Transaction / atomic execution
   → consistency of grouped operations
```

Different problem, different mechanism.

---

# 17. Now Add Sharding

Here's where our earlier Redis Cluster discussion becomes important.

Suppose:

```text
user:123 → Shard A
user:456 → Shard B
```

We want:

```text
Update user:123
+
Update user:456
```

as one atomic operation.

Now we're asking for coordination across:

```text
Shard A
      +
Shard B
```

This is fundamentally harder.

A single-node atomic operation is relatively straightforward.

A cross-shard atomic transaction requires distributed coordination.

And that's exactly the kind of complexity we learned about in:

- distributed transactions
- 2PC
- Saga
- consistency

So Redis Cluster deliberately doesn't turn every arbitrary cross-shard operation into a giant distributed transaction system.

Instead, good Redis data modeling tries to keep operations local to a shard where possible.

---

# 18. This Changes How We Design Redis Keys

Remember:

```text
cart:{user123}:items
cart:{user123}:total
```

The same hash tag can keep related keys on the same slot.

That means:

```text
cart:{user123}:items
cart:{user123}:total
       │
       ▼
   same shard
```

Now operations involving both can potentially remain within one Redis node.

This leads to a broader HLD principle:

> **When using a distributed datastore, design your data model around the operations you need to perform.**

Don't first design arbitrary data structures and then ask the datastore to somehow make all operations transactional.

---

# 19. Redis Atomicity — The Mental Model

At this point, think in layers:

```text
                         Redis
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
    Single command     Multiple commands   Distributed
       atomicity          coordination     coordination
          │                │                │
          ▼                ▼                ▼
        INCR          Transactions      Cross-shard
        SET           Optimistic         operations
        ZADD          concurrency
```

The further right we go:

```text
More coordination
        ↓
More complexity
        ↓
More failure scenarios
```

That's the fundamental distributed-systems story.

---

# 20. When Should We Use These Mechanisms?

### Use atomic commands when:

```text
One operation
+
Needs to be indivisible
```

Examples:

```text
INCR counter
SADD member
ZADD score
```

---

### Use transactions/grouped execution when:

```text
Several Redis operations
+
Need atomic execution
```

---

### Use optimistic concurrency when:

```text
Conflicts are possible
+
We can detect them
+
Retry is acceptable
```

---

### Use distributed locks when:

```text
Multiple application instances
+
Exactly one should perform some work
```

But only after carefully considering the failure semantics.

---

### Use server-side execution when:

```text
Several related operations
+
Must execute atomically
+
Logic is small and data-local
```

---

# 21. The Tradeoff

We've gained powerful coordination capabilities.

But each step adds complexity:

```text
Single command
    ↓
Simple
Fast

Multiple commands
    ↓
Transaction / coordination
More complexity

Cross-shard operation
    ↓
Distributed coordination
Much more complexity
```

Therefore a strong Redis design tries to make the **common operation simple and local**.

For example:

```text
Good:

user:{id}
   ↓
one shard
   ↓
atomic operation
```

is generally much easier than:

```text
Request
  ↓
Shard A
  ↓
Shard B
  ↓
Shard C
  ↓
coordinate
  ↓
rollback/retry
```

---

# Key Takeaways

1. **Individual Redis commands are atomic**, but a sequence of commands isn't automatically one atomic operation.

2. Prefer a **single atomic Redis command** when it can express the required operation.

3. Redis transactions can group multiple commands for atomic execution, but don't equate them with full relational ACID transactions.

4. **Pipelining ≠ transactions.**
   - Pipelining → reduce network round trips.
   - Transactions/grouped execution → coordinate command execution.

5. Optimistic concurrency lets us detect conflicting modifications rather than locking everything upfront.

6. Redis can be used to implement **distributed coordination and locks**, but distributed locks have subtle failure modes.

7. TTL is useful for leases/locks, but expiration doesn't magically guarantee correctness.

8. Server-side execution can make multi-step operations atomic while avoiding multiple network round trips—but expensive work can block other Redis commands.

9. **Cross-shard atomicity is fundamentally harder** than single-shard atomicity.

10. This is why **key design matters** in Redis Cluster:

```text
Related data
    ↓
Same hash slot
    ↓
Same shard
    ↓
Easier atomic operations
```

The deeper principle is:

> **The more distributed your operation becomes, the more coordination you need—and coordination is where distributed-system complexity starts to explode.**

---

## Next: Redis Failure Scenarios

We've now covered:

```text
Fundamentals
    ↓
Persistence
    ↓
Replication / HA
    ↓
Sharding / Cluster
    ↓
Atomicity / Coordination
```

The next step is to stop looking at Redis when everything is working and ask the much more valuable HLD question:

> **"Okay, now let's break it."**

We'll walk through **node failure, replication lag, failover, cache stampede, hot keys, memory exhaustion, network partitions, stale data, and recovery**, and reason about what each failure does to the overall application.
