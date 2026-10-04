# Redis — Part 4: Scaling & Redis Cluster

So far, we've solved two different problems:

```text
Persistence
    → How do we recover data?

Replication
    → How do we survive a node failure?
```

But neither answers:

> **What if one Redis server simply cannot handle the amount of data or traffic our system has?**

That's today's problem.

---

# 1. The Problem

Imagine a system with:

```text
10 TB of data
```

but our Redis server has:

```text
256 GB RAM
```

We obviously can't put the entire dataset on one machine.

Or perhaps the dataset fits:

```text
Redis = 100 GB
Machine = 256 GB RAM
```

but we're receiving:

```text
5 million operations/sec
```

and one machine can't process that much traffic.

We need to distribute the workload.

Conceptually:

```text
              Before

Application
     │
     ▼
┌───────────┐
│   Redis   │
│  256 GB   │
└───────────┘
```

becomes:

```text
              After

              Application
                   │
          ┌────────┼────────┐
          ▼        ▼        ▼
      ┌───────┐ ┌───────┐ ┌───────┐
      │Redis A│ │Redis B│ │Redis C│
      └───────┘ └───────┘ └───────┘
```

But now we face a fundamental question:

> **How do we decide which Redis node should hold a particular key?**

That's where **sharding** comes in.

---

# 2. The Big Idea

> **Redis scaling through sharding divides the keyspace across multiple Redis nodes so that each node is responsible for only part of the dataset and workload.**

Instead of:

```text
Redis
 ├── user:1
 ├── user:2
 ├── user:3
 ├── user:4
 └── user:5
```

we might have:

```text
Redis A
 ├── user:1
 └── user:4

Redis B
 ├── user:2
 └── user:5

Redis C
 └── user:3
```

Now the total dataset is distributed.

---

# 3. Why Can't We Just Add Replicas?

This is an important distinction from the previous chapter.

Suppose we have:

```text
             Primary
            /       \
           ▼         ▼
      Replica A   Replica B
```

All three nodes contain approximately the same dataset.

If the dataset is:

```text
1 TB
```

then we're still storing approximately:

```text
1 TB
```

on the primary.

Adding replicas doesn't solve the **memory capacity problem**.

Similarly, if all writes go to the primary:

```text
             Primary
          5M writes/sec
            /       \
           ▼         ▼
      Replica A   Replica B
```

the primary can remain the bottleneck.

Replication gives us redundancy and potentially read scaling.

It doesn't fundamentally distribute ownership of the dataset.

---

# 4. Sharding Changes the Model

With sharding:

```text
             Dataset
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
    Shard A  Shard B  Shard C
      33%      33%      34%
```

Each shard owns a portion of the keys.

For example:

```text
Shard A
user:1
user:4
user:7

Shard B
user:2
user:5
user:8

Shard C
user:3
user:6
user:9
```

Now:

```text
Total capacity
   =
capacity of A
+
capacity of B
+
capacity of C
```

This is the basic scaling benefit.

---

# 5. How Does Redis Know Where a Key Lives?

This is the interesting part.

Suppose we receive:

```text
GET user:123
```

We need to determine:

```text
user:123 → which node?
```

One simple idea is hashing.

Conceptually:

```text
hash("user:123")
        ↓
      number
        ↓
determine shard
```

For example:

```text
hash("user:123") % 3 = 1
```

So:

```text
user:123 → Shard 1
```

Another key:

```text
hash("user:456") % 3 = 2
```

therefore:

```text
user:456 → Shard 2
```

This should sound familiar.

We learned this concept earlier:

> **Partitioning + hashing → distribute keys across nodes.**

Redis provides a concrete implementation of that idea.

---

# 6. Why Not Just Use `hash(key) % numberOfServers`?

It seems perfectly reasonable.

Suppose:

```text
3 servers
```

We calculate:

```text
hash(key) % 3
```

Now imagine we add a fourth server.

We calculate:

```text
hash(key) % 4
```

Almost every key's destination can change.

For example:

```text
Before:

hash(key) % 3
     ↓
Node B


After adding Node D:

hash(key) % 4
     ↓
Node C
```

Potentially a huge percentage of our data now needs to move.

This is one of the problems that led us to **consistent hashing** in Module 2.

Redis Cluster doesn't use the naive modulo approach.

Instead, it uses a fixed number of **hash slots**.

---

# 7. Redis Hash Slots

Redis Cluster divides the keyspace into:

> **16,384 hash slots**

Don't focus on the exact number as a magic number.

The important idea is:

```text
Key
 ↓
Hash
 ↓
Hash slot
 ↓
Redis node
```

Conceptually:

```text
16,384 slots

[0][1][2][3][4]...[16383]
```

The cluster assigns ranges of these slots to different nodes.

For example:

```text
Node A
slots 0 ─────── 5000

Node B
slots 5001 ──── 10000

Node C
slots 10001 ─── 16383
```

Now:

```text
user:123
   ↓
hash
   ↓
slot 7321
   ↓
Node B
```

---

# 8. Why Hash Slots Help

Suppose we add Node D.

We don't need to completely recalculate the destination of every key.

Instead, we can redistribute some slots:

```text
Before:

A → 0–5000
B → 5001–10000
C → 10001–16383
```

After:

```text
A → 0–4000
B → 4001–8000
C → 8001–12000
D → 12001–16383
```

Only the keys belonging to moved slots need to move.

This is much more manageable.

This is the concrete manifestation of a principle we already learned:

> **Good partitioning minimizes data movement when the cluster changes.**

---

# 9. Scaling Out

Suppose one Redis node has:

```text
256 GB RAM
```

and can handle:

```text
500k operations/sec
```

Three nodes potentially give us a much larger aggregate capacity:

```text
             Cluster
          /     |     \
         A      B      C
       256GB  256GB  256GB
```

Now the dataset can be distributed:

```text
1 GB → A
2 GB → B
3 GB → C
...
```

And traffic can also be distributed.

This is **horizontal scaling**.

Instead of buying one increasingly powerful machine:

```text
Scale Up
   ↓
Bigger machine
```

we add machines:

```text
Scale Out
   ↓
More nodes
```

---

# 10. But Now Replication Comes Back

Earlier we said:

> Sharding distributes data.

But we also said:

> Replication protects against node failure.

So production systems often combine them.

For example:

```text
                    Redis Cluster

             Shard A      Shard B      Shard C
                │             │            │
             Primary       Primary      Primary
                │             │            │
             Replica        Replica      Replica
```

More visually:

```text
             ┌─────────┐
             │Shard A  │
             │Primary  │
             └────┬────┘
                  │
             ┌────▼────┐
             │Replica A│
             └─────────┘


             ┌─────────┐
             │Shard B  │
             │Primary  │
             └────┬────┘
                  │
             ┌────▼────┐
             │Replica B│
             └─────────┘


             ┌─────────┐
             │Shard C  │
             │Primary  │
             └────┬────┘
                  │
             ┌────▼────┐
             │Replica C│
             └─────────┘
```

Now we're solving multiple problems simultaneously:

```text
Sharding
   → scale capacity

Replication
   → redundancy / failover

Persistence
   → recover state
```

---

# 11. What Happens When a Node Dies?

Suppose:

```text
Shard B Primary 💥
```

Its replica can potentially be promoted:

```text
Before:

Shard B
   Primary B 💥
       │
       ▼
   Replica B


After:

Shard B
   Primary B'
```

The important thing is that **only that shard's ownership needs to fail over**.

The entire cluster doesn't necessarily need to stop.

This is one of the major advantages of distributing the system.

---

# 12. The Cost: Distributed Complexity

We've made Redis more scalable.

But look what we've introduced:

```text
Single Redis

Application
    ↓
Redis
```

was simple.

Now:

```text
Application
    │
    ├── Shard A
    ├── Shard B
    ├── Shard C
    ├── Replica A
    ├── Replica B
    └── Replica C
```

We now have to reason about:

- routing
- slot ownership
- rebalancing
- node failures
- failover
- replication lag
- cluster membership
- resharding
- cross-shard operations

This is the classic distributed-systems tradeoff:

> **More scalability comes with more coordination and failure modes.**

---

# 13. Cross-Shard Operations

Here's a particularly important limitation.

Suppose we have:

```text
user:123 → Shard A
user:456 → Shard B
```

Now the application wants to perform an operation involving both:

```text
user:123
+
user:456
```

We have a problem.

They're on different nodes.

```text
          Application
           /       \
          ▼         ▼
      Shard A    Shard B
       user123    user456
```

A simple single-node atomic operation is no longer enough.

This should immediately connect to our earlier learning:

> **Distributed operations are harder than local operations.**

Sharding therefore changes what operations are easy.

---

# 14. Designing Keys Becomes Important

This is one of the most practical Redis Cluster lessons.

Suppose we have:

```text
cart:user:123
cart:user:456
```

If related data needs to participate in the same atomic operation, we'd ideally want it placed on the same shard.

Redis provides a mechanism called **hash tags** for controlling which portion of the key is used for slot calculation.

Conceptually:

```text
cart:{user123}:items
cart:{user123}:total
```

The `{user123}` portion can cause related keys to map to the same hash slot.

So:

```text
cart:{user123}:items
          │
          ├──── same hash tag
          │
cart:{user123}:total
```

→ same slot → same shard.

This enables certain multi-key operations that would otherwise cross shards.

The broader lesson is more important than the syntax:

> **In a distributed data store, your data model and key design can influence what operations remain cheap and atomic.**

---

# 15. Hot Keys

Here's another problem.

Suppose we have:

```text
10 million keys
```

and they're beautifully distributed.

But one key receives:

```text
50 million requests/sec
```

For example:

```text
trending:worldcup
```

The hash function doesn't care.

That key maps to one shard:

```text
                Cluster
          ┌──────┬──────┬──────┐
          ▼      ▼      ▼
        Shard A Shard B Shard C
                        ↑
                        │
                 50M requests
```

Shard C becomes overloaded.

This is a **hot key**.

Sharding distributes **keys**, not necessarily traffic evenly.

That's a crucial distinction.

---

# 16. Hot Keys Are Different From Uneven Data

Imagine:

```text
Shard A → 10 GB
Shard B → 10 GB
Shard C → 10 GB
```

Storage is balanced.

But:

```text
Shard A → 100k ops/sec
Shard B → 100k ops/sec
Shard C → 50M ops/sec
```

Traffic is not balanced.

So when evaluating a distributed datastore, don't only ask:

> "Is the data evenly distributed?"

Also ask:

> **"Is the workload evenly distributed?"**

This is a classic HLD interview point.

---

# 17. Sharding Does Not Solve Everything

We've increased:

```text
Dataset capacity
+
Aggregate throughput
```

But some operations become harder.

For example:

```text
Single shard:
GET user:123

Easy.
```

versus:

```text
Multiple shards:
Find all users whose score > 5000
```

That may require:

```text
Query Shard A
Query Shard B
Query Shard C
...
Combine results
```

Now the application or coordination layer has more work.

So the fundamental tradeoff is:

```text
                Sharding
                   │
       ┌───────────┴───────────┐
       ▼                       ▼
More scale                  More complexity
More capacity               Cross-shard operations
More throughput             Rebalancing
                             Hot spots
```

---

# 18. Redis Scaling Journey

Let's look at how Redis has evolved through the problems we've encountered.

### Step 1 — Single instance

```text
Application
    ↓
Redis
```

Problem:

> One machine's capacity is limited.

---

### Step 2 — Persistence

```text
Application
    ↓
Redis
    ↓
Persistent storage
```

Problem:

> What if the Redis process/machine fails?

---

### Step 3 — Replication

```text
             Primary
             /     \
            ▼       ▼
        Replica   Replica
```

Problem:

> What if the dataset or write workload exceeds one machine?

---

### Step 4 — Sharding

```text
       ┌──────┬──────┬──────┐
       ▼      ▼      ▼      ▼
     Shard A Shard B Shard C ...
```

Problem:

> Now we need redundancy for each shard.

---

### Step 5 — Sharded + Replicated

```text
       Shard A        Shard B        Shard C
          │              │              │
       Primary          Primary        Primary
          │              │              │
       Replica          Replica        Replica
```

This is much closer to what we'd expect from a serious distributed Redis deployment.

---

# 19. What Redis Cluster Gives Us

At a high level, Redis Cluster provides mechanisms for:

- distributing keys across nodes
- routing requests to the appropriate node
- detecting node failures
- replica-based failover
- adding/removing capacity
- moving slots between nodes

But remember the philosophy of this course:

We don't need to memorize Redis Cluster internals.

The HLD-level understanding is:

```text
Redis Cluster
      │
      ├── Partition keyspace
      ├── Distribute workload
      ├── Replicate shards
      ├── Fail over failed nodes
      └── Rebalance as cluster changes
```

That's the architectural model you need to defend.

---

# 20. The Bigger Picture

We can now describe Redis at a much higher level:

```text
                         Redis
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
      In-memory        Persistence       Replication
          │                │                │
      Fast access      Recovery          Redundancy
          │
          ▼
       Sharding
          │
          ▼
    Horizontal scale
```

And this connects almost perfectly to the concepts from Modules 1–8.

---

# 21. The HLD Decision

Imagine you're designing a high-traffic application and someone proposes:

> "Let's put everything into one Redis instance."

You should immediately ask:

### Dataset size?

```text
10 GB?
500 GB?
10 TB?
```

### Traffic?

```text
100k ops/sec?
10M ops/sec?
```

### Read/write ratio?

```text
99:1?
50:50?
```

### Failure requirements?

```text
Can Redis be temporarily unavailable?
```

### Durability?

```text
Can data be reconstructed?
```

### Access pattern?

```text
Simple key lookup?
Complex cross-key operations?
```

### Hot keys?

```text
Could one key receive disproportionate traffic?
```

### Scaling strategy?

```text
Vertical?
Replication?
Sharding?
Both?
```

That is **technology-level HLD reasoning**.

---

# Key Takeaways

The important things to retain:

1. **Replication doesn't solve capacity.** It creates copies; sharding distributes ownership.

2. **Sharding divides the keyspace across nodes**, allowing Redis to scale horizontally.

3. Redis Cluster uses **hash slots** to map keys to nodes.

4. Hash slots make cluster resizing and resharding more manageable than naïve `hash(key) % N`.

5. **Sharding increases capacity and aggregate throughput**, but introduces distributed-system complexity.

6. Cross-shard operations are harder because the relevant data may live on different machines.

7. **Key design matters** because related keys may need to live on the same shard.

8. **Hot keys can overload a single shard even when the overall dataset is perfectly balanced.**

9. Production Redis architecture often combines:

   ```text
   Sharding
       +
   Replication
       +
   Persistence
       +
   Failover
   ```

10. The fundamental scaling question is not simply:
    > "Can Redis scale?"

It's:

> **"What exactly are we scaling — data capacity, read throughput, write throughput, or availability — and which mechanism addresses that constraint?"**

---

### Next: Redis Atomicity, Transactions & Coordination

We've made Redis distributed. That creates the next interesting problem:

> **When Redis is spread across multiple nodes, what does it actually mean for an operation to be atomic?**

We'll look at individual command atomicity, multi-command operations, transactions, optimistic concurrency, Lua/server-side execution at the HLD level, and where these guarantees break down across shards.
