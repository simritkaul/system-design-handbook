# Redis — Part 6: Failure Scenarios

So far, we've mostly looked at Redis when everything is behaving correctly:

```text
Application
    ↓
Redis Cluster
    ↓
Data
```

But HLD design is often more about answering:

> **"What happens when something goes wrong?"**

A Redis architecture isn't good because it works when every node is healthy.

It's good when we understand how the system behaves **when nodes fail, data becomes stale, traffic spikes, or Redis becomes unavailable**.

We'll approach failures by asking:

```text
Failure
   ↓
What breaks?
   ↓
What does the application observe?
   ↓
How does the architecture recover?
   ↓
What tradeoff did we accept?
```

---

# 1. Redis Node Failure

Let's start with the simplest failure.

```text
Application
    ↓
Redis Primary
```

The Redis node crashes.

```text
Application
    ↓
Redis 💥
```

If this is a standalone Redis instance:

```text
Redis unavailable
     ↓
Application requests fail
```

If Redis is only a cache, the application may be able to fall back:

```text
Redis MISS / unavailable
        ↓
     Database
        ↓
     Response
        ↓
Repopulate Redis
```

That's relatively manageable.

But if Redis is the source of truth:

```text
Redis unavailable
      ↓
Application cannot access data
```

Now it's a much more serious outage.

This is why our first architectural question should always be:

> **What role does Redis play in this system?**

---

# 2. Redis as a Cache vs Source of Truth

This distinction determines the blast radius.

### Redis as cache

```text
             ┌──── Redis ────┐
             │               │
Application ─┤               │
             │               │
             └──── Database ─┘
```

Redis failure:

```text
Redis 💥
   ↓
Cache misses
   ↓
Database
```

The system becomes slower, but the underlying data remains available.

---

### Redis as source of truth

```text
Application
     ↓
   Redis
     ↓
  only copy
```

Redis failure:

```text
Redis 💥
   ↓
Data unavailable
```

Now persistence, replication, failover, and recovery become critical.

**Same technology. Completely different failure consequences.**

---

# 3. Primary Failure With Replication

Now consider:

```text
             Primary A
             /       \
            ▼         ▼
       Replica B   Replica C
```

A dies:

```text
             Primary A 💥
                    X
                   / \
                  ▼   ▼
             Replica B  C
```

The system can promote one of the replicas:

```text
             Primary B
                  │
                  ▼
             Replica C
```

This is **failover**.

The objective is to minimize:

```text
Primary failure
      ↓
downtime
      ↓
recovery
```

---

# 4. But Failover Isn't Instant

This is an important interview detail.

There's usually a sequence:

```text
Primary fails
     ↓
Failure detected
     ↓
Determine appropriate replica
     ↓
Promote replica
     ↓
Redirect traffic
     ↓
Resume normal operation
```

During this period:

```text
Some requests
      ↓
may fail
```

So when someone says:

> "Redis has replicas, so there is zero downtime."

That's too strong.

A better statement is:

> **Replication and automatic failover can significantly reduce downtime, but failover itself takes time and can have consistency consequences.**

---

# 5. What Data Can Be Lost During Failover?

Recall replication lag.

Suppose:

```text
Primary:
A
B
C
D

Replica:
A
B
C
```

Then primary dies.

Replica becomes primary.

Its state is:

```text
A
B
C
```

`D` may be lost.

This gives us a very important concept:

> **Failover can preserve availability at the cost of potentially losing writes that had not yet been replicated.**

This is one of the fundamental availability-vs-durability tradeoffs in distributed systems.

---

# 6. Cache Stampede

Now let's look at a failure that has nothing to do with a Redis server crashing.

Suppose we have a popular cached object:

```text
product:123
```

It receives:

```text
100,000 requests/sec
```

Normally:

```text
Requests
   ↓
Redis HIT
   ↓
Response
```

Now its TTL expires.

```text
product:123
      ↓
   expires
```

Suddenly:

```text
100,000 requests
       ↓
     MISS
       ↓
Database
```

Instead of one database request, we may create:

```text
100,000 database requests
```

The database gets overwhelmed.

This is a **cache stampede**.

---

# 7. Why This Is Particularly Dangerous

The failure cascades:

```text
Cache expires
      ↓
Many cache misses
      ↓
Database traffic spikes
      ↓
Database becomes slow
      ↓
Application requests take longer
      ↓
More concurrent requests accumulate
      ↓
System becomes unstable
```

Notice:

> **Redis itself hasn't failed.**

The cache behaved exactly as configured.

The problem is the interaction between:

```text
TTL
+
high concurrency
+
database fallback
```

This is classic HLD reasoning: individual components can be healthy while the **system as a whole fails**.

---

# 8. Preventing Cache Stampede

Several strategies are possible.

### Stagger expiration

Instead of:

```text
Everything expires at 10:00
```

we introduce variation:

```text
10:00:13
10:00:41
10:01:05
10:01:28
```

Now requests are less likely to miss simultaneously.

---

### Request coalescing / single-flight

Instead of:

```text
100 requests miss
       ↓
100 database calls
```

we want:

```text
100 requests miss
       ↓
1 request loads data
       ↓
other requests wait/share result
```

Conceptually:

```text
          Cache MISS
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
      R1     R2     R3
       \      |     /
        \     |    /
         ▼    ▼   ▼
        One loader
             │
             ▼
          Database
```

---

### Refresh before expiration

We can proactively refresh frequently accessed data before it expires.

The broader lesson:

> **Caching introduces its own failure modes.**

We learned these conceptually in Module 3; now we're seeing how they manifest with a real technology.

---

# 9. Cache Breakdown / Hot Key

Consider a single key:

```text
homepage:trending
```

It receives:

```text
10 million requests/sec
```

Even if Redis has 100 nodes:

```text
Redis Cluster
 ┌────┬────┬────┬────┐
 │ A  │ B  │ C  │ D  │
 └────┴────┴────┴────┘
          ↑
          │
    homepage:trending
```

That key maps to one shard.

So one node might receive:

```text
10M requests/sec
```

while the others are barely busy.

This is the **hot key problem**.

---

# 10. Why Sharding Doesn't Automatically Solve Hot Keys

This is worth emphasizing.

Sharding solves:

> **How do I distribute many keys across machines?**

It does not necessarily solve:

> **How do I distribute traffic for one extremely popular key?**

A single key still has one logical owner.

Potential approaches include:

### Replicating the hot value

Instead of:

```text
hot-key → one node
```

we may intentionally maintain multiple copies:

```text
hot-key
  ├── Node A
  ├── Node B
  ├── Node C
  └── Node D
```

and distribute reads.

Or use an application-level strategy such as key replication/suffixing where appropriate.

But now we're trading:

```text
More read capacity
```

for:

```text
More copies
+
More memory
+
More invalidation complexity
```

Again, no free lunch.

---

# 11. Memory Exhaustion

Redis is fundamentally memory-oriented.

Eventually:

```text
RAM
████████████████████
        FULL
```

What happens next depends heavily on how Redis is configured and how we're using it.

If Redis is a cache, eviction may be acceptable:

```text
Memory full
    ↓
Evict entries
    ↓
Store new data
```

If Redis holds important persistent state:

```text
Memory full
    ↓
Evict important data
```

could be catastrophic.

So memory planning is not just:

> "How much data do we have?"

We need to consider:

```text
Dataset size
+
replication copies
+
metadata / overhead
+
growth
+
headroom
```

---

# 12. The Hidden Cost of Replication

Suppose we have:

```text
100 GB dataset
```

with:

```text
1 primary
2 replicas
```

Conceptually we're now storing approximately:

```text
100 GB × 3
```

before accounting for overhead.

So replication improves availability but increases:

- memory consumption
- network traffic
- infrastructure cost

This is another reason to ask:

> **How much redundancy do we actually need?**

---

# 13. Network Partition

Now let's look at a much nastier distributed failure.

Suppose:

```text
Primary A
     │
     │ network
     │
Replica B
```

The network connection between them breaks.

But neither machine necessarily crashed.

We have:

```text
A: "B is unreachable."

B: "A is unreachable."
```

This is different from:

```text
A crashed.
```

We have a **network partition**.

This is where distributed systems get difficult because a node cannot instantly distinguish:

```text
"The other node is dead."
```

from:

```text
"The other node is alive but unreachable."
```

---

# 14. Why Network Partitions Are Dangerous

Suppose B decides:

> "A must be dead."

and becomes primary.

But A is actually alive.

Now:

```text
Primary A
    │
    │
    X network partition
    │
    │
Primary B
```

We potentially have two writers.

This is **split brain**.

Now:

```text
Client 1 → A → write X

Client 2 → B → write Y
```

The system has divergent state.

This is why automatic failover needs coordination and safeguards.

---

# 15. Stale Reads

We've already seen replication lag, but let's view it from the application's perspective.

Suppose:

```text
User changes:
name = "Simrit"
```

Write goes to primary.

Immediately afterward:

```text
GET user
```

goes to a replica.

If replication hasn't caught up:

```text
Replica → old name
```

The user sees stale information.

This is sometimes acceptable.

For example:

```text
View count
```

might tolerate a small delay.

But:

```text
Bank account balance
```

usually requires much stronger guarantees.

Therefore:

> **Consistency requirements should influence whether and how we read from replicas.**

---

# 16. Redis Failure Doesn't Always Mean Application Failure

This is an important architecture principle.

Suppose Redis is being used only for:

```text
recommendations cache
```

Redis dies.

The application might do:

```text
Redis unavailable
      ↓
Fallback
      ↓
Recommendation service / database
```

System:

```text
Slower
but functional
```

Compare that with:

```text
Redis = session store
```

If sessions are unavailable:

```text
Redis unavailable
      ↓
Users cannot authenticate
```

Potentially much larger impact.

Or:

```text
Redis = source of truth
```

Now failure can mean:

```text
Data unavailable
```

So resilience isn't solely a property of Redis.

It's a property of:

```text
Redis
+
how the application depends on Redis
```

---

# 17. Failure Isolation

Suppose one Redis shard becomes overloaded:

```text
Shard A → 100k ops/sec
Shard B → 100k ops/sec
Shard C → 50M ops/sec 💥
```

A good architecture tries to prevent:

```text
Shard C failure
      ↓
Entire application fails
```

Instead:

```text
Shard C failure
      ↓
Only affected workload degrades
```

This is the same principle we learned in Module 8:

> **Bulkhead / failure isolation.**

Redis cluster architecture can help with isolation, but the application still needs to understand what happens when a particular shard is unavailable.

---

# 18. Persistence Failure

We've discussed Redis persistence.

But what if persistence itself fails?

For example:

```text
Redis
   ↓
Persistence
   ↓
Disk unavailable
```

The in-memory system may continue working temporarily.

But now:

```text
Current state
     ↓
not safely persisted
```

If the process subsequently crashes:

```text
Redis 💥
   ↓
Recovery point may be older
```

So production monitoring should care about more than:

> "Is Redis responding?"

We also care about:

- memory pressure
- replication health
- replication lag
- persistence health
- failover status
- command latency
- throughput
- hot keys
- connection pressure

You don't need to become a Redis operator for HLD interviews, but you should understand **what can go wrong**.

---

# 19. A Failure Matrix

Let's put the major scenarios together.

| Failure                          | What happens                    | Typical architectural response                 |
| -------------------------------- | ------------------------------- | ---------------------------------------------- |
| Redis process crashes            | Node unavailable                | Restart/recover                                |
| Primary node dies                | Writes interrupted              | Replica failover                               |
| Replica lag                      | Replica has older data          | Understand/tolerate consistency lag            |
| Cache expires under huge load    | DB gets flooded                 | Single-flight, staggered TTL, refresh          |
| Hot key                          | One shard overloaded            | Replicate/distribute hot reads                 |
| Memory exhaustion                | Writes/eviction affected        | Capacity planning, eviction strategy           |
| Network partition                | Nodes can't communicate         | Failover coordination / split-brain prevention |
| Persistence failure              | Recovery point may become older | Monitor persistence, redundancy                |
| Entire Redis cluster unavailable | Application dependency breaks   | Fallback/degraded mode where possible          |

This is the kind of table I'd want you to be able to mentally reconstruct during an interview.

---

# 20. The Most Important Question: What Does the Application Do?

This is perhaps the most valuable Redis HLD lesson so far.

Suppose Redis goes down.

Don't immediately say:

> "We need replicas."

First ask:

> **"What does the application need to do if Redis is unavailable?"**

There are several possibilities.

### Case 1 — Redis is a cache

```text
Redis unavailable
      ↓
Database
```

Potentially acceptable.

---

### Case 2 — Redis is acceleration only

```text
Redis unavailable
      ↓
Slower application
      ↓
Still functional
```

Maybe acceptable temporarily.

---

### Case 3 — Redis stores critical state

```text
Redis unavailable
      ↓
Cannot perform operation
```

We need strong HA and recovery guarantees.

---

### Case 4 — Redis is source of truth

```text
Redis unavailable
      ↓
Actual business data unavailable
```

Now durability and recovery become fundamental.

So:

> **The correct failure strategy depends on the role Redis plays in the architecture.**

---

# 21. The Redis Reliability Picture

At this point, we can finally see how all our pieces fit together:

```text
                         Redis
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
   Persistence        Replication         Sharding
        │                  │                  │
   Recover data       Survive node      Scale capacity
                      failure
        │                  │                  │
        └──────────────┬───┴──────────────────┘
                       ▼
                  High resilience
                       │
                       ▼
                But still need:
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
          App       Failure     Capacity
        fallback    handling    planning
```

Redis doesn't magically make the entire application reliable.

It gives us mechanisms.

**The architecture around Redis determines whether those mechanisms actually protect the system.**

---

# 22. Redis's Fundamental Tradeoffs

We can now summarize the technology much more meaningfully.

### Why Redis is attractive

```text
In-memory
   ↓
Very low latency

Data structures
   ↓
Efficient operations

TTL
   ↓
Temporary state

Atomic operations
   ↓
Counters / coordination

Replication
   ↓
Availability

Sharding
   ↓
Horizontal scaling
```

### What we pay for

```text
RAM
   ↓
Expensive capacity

Replication
   ↓
More memory/network/cost

Sharding
   ↓
Distributed complexity

Replication lag
   ↓
Potential stale reads

Failover
   ↓
Potential write loss / temporary downtime

Hot keys
   ↓
Uneven workload

Cache usage
   ↓
Invalidation complexity
```

This is much more useful than memorizing:

> "Redis is fast."

---

# 23. Redis Decision Framework

When someone says:

> "Should we use Redis?"

You should mentally walk through:

```text
                Workload
                   │
                   ▼
        Is very low latency important?
                   │
                  Yes
                   │
                   ▼
       Is access mostly key/data-structure
              oriented?
                   │
                  Yes
                   │
                   ▼
      Does the workload benefit from
       in-memory/shared state?
                   │
                  Yes
                   │
                   ▼
            Redis is a candidate
                   │
                   ▼
        ┌─────────────────────┐
        │ Now evaluate:       │
        │                     │
        │ Durability          │
        │ Availability        │
        │ Dataset size        │
        │ Traffic             │
        │ Hot keys            │
        │ Consistency         │
        │ Cost                │
        │ Alternatives        │
        └─────────────────────┘
```

That's the HLD mindset we're after.

---

# Key Takeaways

The failure scenarios I'd especially want you to retain are:

1. **Redis failure is not necessarily application failure.** It depends on what Redis is doing.

2. **Replication + failover** can reduce downtime when a Redis node dies, but failover isn't instantaneous and may lose unreplicated writes.

3. **Replication lag** can cause stale reads.

4. **Cache stampede** happens when many requests simultaneously miss the cache and overwhelm the underlying system.

5. **Hot keys** can overload one shard even when the overall dataset is well distributed.

6. **Memory exhaustion** is particularly important because Redis relies heavily on RAM.

7. **Network partitions** can create split-brain risks and are much harder than simple machine failures.

8. **Replication is not backup.** A bad write can be replicated everywhere.

9. Application-level **fallback and graceful degradation** are often just as important as Redis's own HA mechanisms.

10. The real HLD question is:

> **"If Redis fails, what exactly are we willing to lose: availability, freshness, performance, or data?"**

That answer should drive the Redis architecture.

---

## Where we've reached

Our Redis journey is now:

```text
Fundamentals
     ↓
Persistence
     ↓
Replication & HA
     ↓
Sharding & Cluster
     ↓
Atomicity & Coordination
     ↓
Failure Scenarios
```

The next useful step is **Redis in real HLD use cases**.

Rather than learning more Redis features in isolation, we'll take systems such as:

- distributed rate limiter
- session store
- leaderboard
- caching layer
- distributed lock
- counters
- real-time state

and ask:

> **Would Redis actually be the right technology here? Why? How would we design it? What can go wrong?**

That is where the individual Redis concepts start turning into **technology-selection skill**.
