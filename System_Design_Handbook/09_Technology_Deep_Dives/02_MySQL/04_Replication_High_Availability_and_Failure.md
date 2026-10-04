# MySQL — Part 4: Replication, High Availability & Failure

We've now reached an important transition.

So far, we've mostly looked at **one MySQL instance**:

```text
Application
     ↓
   MySQL
     ↓
   Disk
```

But in a real production system, one machine creates two problems:

1. **Capacity** — eventually one machine may not handle all the traffic.
2. **Availability** — if that machine dies, the database becomes unavailable.

So we need to distribute MySQL across multiple machines.

And this takes us directly back to **Module 2 — Replication**.

---

# 1. The Problem

Imagine an e-commerce system:

```text
              Application
                   │
                   ▼
                MySQL
```

Everything depends on that instance.

Now:

```text
MySQL server crashes
       ↓
Orders unavailable
       ↓
Payments unavailable
       ↓
Application partially/fully unavailable
```

Even if the application has:

```text
10 servers
```

it doesn't matter if they all depend on one database.

We've created a **single point of failure**.

So our first requirement is:

> **If one database server fails, another server should be able to continue serving the system.**

---

# 2. Replication

The basic solution:

```text
                 Primary
                    │
             replicate data
              /           \
             ▼             ▼
         Replica A      Replica B
```

The primary handles writes.

Replicas maintain copies of the data.

So instead of:

```text
Application
     ↓
  MySQL A
```

we have:

```text
                  ┌── Replica A
                  │
Application ──────┼── Replica B
                  │
                  ▼
               Primary
```

The exact topology can become more sophisticated, but this is the foundation.

---

# 3. Why Have Replicas?

There are actually **two separate reasons**.

### Reason 1 — Availability

If the primary dies:

```text
Primary
   X
```

we can promote a replica:

```text
Replica A
    ↓
 becomes Primary
```

The system can continue.

---

### Reason 2 — Read Scaling

Suppose:

```text
100,000 reads/sec
10,000 writes/sec
```

We don't necessarily want all reads hitting the primary.

We can distribute reads:

```text
                 Primary
                /        \
               ▼          ▼
          Replica A   Replica B
             ↑            ↑
             │            │
           Reads        Reads
```

So replication can simultaneously provide:

```text
Availability
+
Read scalability
```

But these goals aren't identical.

That's important.

---

# 4. Replication Does NOT Mean Backup

This is one of the most important distinctions.

Suppose:

```text
Primary
   │
   ▼
Replica
```

Someone accidentally executes:

```sql
DROP TABLE orders;
```

The destructive change may replicate too.

So:

```text
Primary
   ↓
bad operation
   ↓
Replica
```

The replica is not magically a historical copy.

It's generally trying to follow the primary's state.

Therefore:

```text
Replication
≠
Backup
```

A backup is intended to let you recover from things such as:

```text
Accidental deletion
Corruption
Bad deployment
Logical mistakes
Historical recovery
```

Replication primarily helps with:

```text
Availability
Read scaling
Failure recovery
```

You typically want **both**.

---

# 5. How Does Replication Work?

At a high level:

```text
Application
     │
     ▼
  Primary
     │
     │ records changes
     ▼
Replication information
     │
     ▼
  Replica
     │
     ▼
Apply changes
```

The details of MySQL's replication mechanisms are deeper than we need for HLD.

The important conceptual model is:

> **The primary produces a sequence of database changes, and replicas consume/apply those changes to maintain a corresponding state.**

This is similar to concepts you've already seen in Kafka:

```text
Producer
   ↓
ordered changes
   ↓
Consumer
```

The implementation is different.

The architectural idea is familiar.

---

# 6. Replication Is Usually Asynchronous

This is crucial.

Suppose:

```text
Client
  │
  ▼
Primary
  │
  │ 100 ms later
  ▼
Replica
```

The primary can acknowledge a write before every replica has necessarily applied it.

This means:

```text
Primary:
email = new@example.com

Replica:
email = old@example.com
```

for some period of time.

That's **replication lag**.

---

# 7. Why Replication Lag Matters

Consider a user changing their profile:

```text
POST /profile
     ↓
Primary
     ↓
Success
```

Immediately afterward:

```text
GET /profile
     ↓
Replica
```

The replica may still have:

```text
old profile
```

The user sees:

> "My update didn't work."

But the write actually succeeded.

The problem is not data loss.

It's **where we read from**.

---

# 8. Read-After-Write Consistency

This is a classic distributed-systems problem.

The user expects:

```text
Write X
  ↓
Read X
  ↓
See X
```

But if:

```text
Write → Primary
Read  → Replica
```

and replication hasn't caught up:

```text
Write X
  ↓
Primary has X

Read
  ↓
Replica still has old value
```

So we need an architectural decision.

---

# 9. Solution 1 — Read From Primary

For operations requiring immediate visibility:

```text
Write
  ↓
Primary

Immediately after:
Read
  ↓
Primary
```

Simple.

But now we're putting more read traffic on the primary.

---

# 10. Solution 2 — Session Stickiness

We can sometimes route a user's reads to the primary for some period after a write.

Conceptually:

```text
User
  │
  ├── Write ──→ Primary
  │
  └── Read  ──→ Primary
```

Later:

```text
User
  │
  └── Read ──→ Replica
```

This can reduce stale reads while still allowing replicas to handle normal traffic.

But it introduces routing complexity.

---

# 11. Solution 3 — Wait for Replication

Another possibility is to only serve a read from a replica once it has caught up sufficiently.

Conceptually:

```text
Write
 ↓
Primary
 ↓
Replication
 ↓
Replica catches up
 ↓
Read
```

This can increase latency.

Again:

> **Consistency guarantees have a cost.**

---

# 12. Replication Lag Can Become Dangerous

Suppose the primary is processing:

```text
10,000 writes/sec
```

but a replica can only apply:

```text
5,000 changes/sec
```

Then:

```text
Incoming:
10k/sec

Replica:
5k/sec
```

The backlog grows.

Conceptually:

```text
Primary
  │
  │ 10k/sec
  ▼
Replica
  │
  │ 5k/sec
  ▼
Applied
```

Eventually:

```text
Replication lag ↑↑
```

So adding replicas doesn't automatically solve everything.

You need to understand:

> **Can replicas keep up with the change rate?**

---

# 13. Replication Topologies

The simplest topology is:

```text
              Primary
             /       \
            ▼         ▼
        Replica A  Replica B
```

This is easy to reason about.

But as the system grows, we might have:

```text
                 Primary
                /       \
               ▼         ▼
           Replica A   Replica B
              │
              ▼
          Replica C
```

Or other topologies.

Why might we do this?

Because replication traffic itself can become a concern.

Again, distributed architecture evolves because the previous solution introduces another bottleneck.

---

# 14. High Availability

Now let's focus specifically on availability.

Suppose:

```text
                 Primary
                /       \
               ▼         ▼
          Replica A   Replica B
```

Primary crashes.

We need:

```text
1. Detect failure
2. Choose a suitable replica
3. Promote it
4. Redirect writes
```

Conceptually:

```text
              Primary
                 X
                 │
                 ▼
           Failure detection
                 │
                 ▼
             Replica A
                 │
              Promote
                 │
                 ▼
              Primary
```

This is **failover**.

---

# 15. The Hard Part: Detecting Failure

Imagine the primary stops responding.

How do we know?

Possibilities:

```text
Heartbeat
Health checks
Connection failures
Timeouts
Monitoring signals
```

But here's the tricky part.

A timeout doesn't necessarily mean:

> "The server is dead."

It could mean:

```text
Network partition
Temporary overload
Slow disk
GC pause
Network congestion
```

So distributed systems have to distinguish:

```text
"server is actually dead"
```

from:

```text
"I currently can't reach the server."
```

This is a fundamental distributed-systems problem.

---

# 16. The Split-Brain Problem

This is one of the most important HA concepts.

Imagine:

```text
              Network partition
                 XXXXXXXXX
                /         \
               /           \
          Primary        Replica
```

The replica can't communicate with the primary.

The replica concludes:

> "Maybe the primary is dead."

It promotes itself.

But suppose the primary is actually still alive.

Now:

```text
Primary
   ↓
accepting writes

Replica
   ↓
also accepting writes
```

We now have **two primaries**.

That's split brain.

```text
        Client A
           ↓
       Primary A
           │
           X
           │
       Primary B
           ↑
        Client B
```

Both can diverge.

This is extremely dangerous for databases.

---

# 17. Why Automatic Failover Is Hard

You might think:

> "Just promote a replica whenever the primary stops responding."

But remember:

```text
Can't reach primary
```

doesn't prove:

```text
Primary is dead
```

Therefore safe failover requires more than:

```text
if ping fails → promote
```

You need coordination around:

```text
Who is allowed to be primary?
Who is allowed to accept writes?
How do we prevent two nodes from believing they're primary?
```

This connects directly to the **leader election** concept from Module 8.

---

# 18. Leader Election

We previously learned the abstract concept:

> **A distributed system needs a mechanism to decide which node is the leader.**

For a replicated database:

```text
             Cluster
          /     |      \
         A      B       C
          \     |      /
             Election
                ↓
             Leader
```

The leader is responsible for writes.

If it fails:

```text
Leader
   X
   ↓
Election
   ↓
New Leader
```

This is conceptually simple.

Making it safe in the presence of network failures is much harder.

---

# 19. Promotion Has Another Problem: Missing Data

Suppose:

```text
Primary:
Order #100 committed

Replica:
Only has up to Order #98
```

Primary crashes.

We promote the replica.

Now:

```text
Order #100
```

might not be present there.

So failover can involve a tradeoff:

```text
Fast failover
      vs
Maximum data preservation
```

This is one of the reasons replication configuration and failure handling matter.

---

# 20. Synchronous vs Asynchronous Replication

Now we can frame the fundamental tradeoff.

### Asynchronous

```text
Primary
  │
  ├── acknowledge client
  │
  └── replicate later
```

Advantages:

```text
Lower write latency
Primary doesn't have to wait for replicas
```

But:

```text
Replica may lag
Recent writes may not exist on replica during failure
```

---

### Synchronous

Conceptually:

```text
Primary
   │
   ├── replicate
   ▼
Replica confirms
   │
   ▼
Primary acknowledges client
```

Now the primary waits for replication before considering the operation sufficiently replicated.

Benefits:

```text
Stronger durability/consistency guarantees
```

Costs:

```text
Higher latency
Availability can depend on replica/network health
```

Again:

> **Stronger guarantees require coordination.**

---

# 21. A Useful HLD Question

Suppose you're designing a banking system.

Would you be comfortable saying:

> "We'll asynchronously replicate the primary and it's fine if the most recent transaction disappears during a primary failure."

Probably not.

You might prioritize stronger durability.

Now imagine:

```text
Social media likes
```

Would you accept:

> "A tiny amount of replication lag is okay."

Probably yes.

So the technology choice isn't simply:

```text
Sync = good
Async = bad
```

It's:

> **What guarantees does the business actually require?**

---

# 22. Replication vs Sharding

These two are often confused.

### Replication

Copies the same data:

```text id="z8ekgm"
Primary
   │
   ├── Replica A
   └── Replica B

Same dataset
```

Primarily useful for:

```text
Availability
Read scaling
```

---

### Sharding

Splits the dataset:

```text id="3o5vqt"
Shard A → Users 1–1M
Shard B → Users 1M–2M
Shard C → Users 2M–3M
```

Primarily useful for:

```text
Write scaling
Storage scaling
Dataset scaling
```

And of course you can combine them:

```text id="sm6kgr"
Shard A
 ├── Primary
 ├── Replica
 └── Replica

Shard B
 ├── Primary
 ├── Replica
 └── Replica
```

Now we're building a genuinely distributed database architecture.

---

# 23. Read Scaling vs Write Scaling

This distinction should become instinctive.

Suppose:

```text id="n1h8fq"
1 Primary
100k reads/sec
5k writes/sec
```

Replicas may help enormously:

```text
Primary → 5k writes/sec
Replicas → distribute reads
```

But if we have:

```text id="nqg3iw"
50k writes/sec
```

replicas don't magically make the primary accept 10× more writes.

The writes still converge on the primary.

That's when we start considering:

```text
Sharding
Partitioning
Write distribution
```

So:

> **Replication primarily duplicates capacity; sharding divides responsibility.**

Excellent interview phrase to remember.

---

# 24. Connection to Redis

You've already seen almost the exact same architectural questions with Redis.

Redis:

```text
Primary
   ↓
Replicas
```

MySQL:

```text
Primary
   ↓
Replicas
```

Both encounter:

```text
Replication lag
Failover
Read scaling
Stale reads
Split brain
Leader election
```

The underlying distributed-system concepts are the same.

The technology-specific details differ.

That's exactly why we learned the concepts before reaching Module 9.

---

# 25. Connection to Module 8

Remember:

```text
Leader Election
```

We learned that abstractly.

Now:

```text
MySQL cluster
     ↓
Who is primary?
     ↓
Leader election / failover mechanisms
```

Remember:

```text
Timeout
```

Now:

```text
Primary hasn't responded
     ↓
Timeout
     ↓
Is it actually dead?
```

Remember:

```text
Retry
```

Now:

```text
Primary fails
     ↓
client retries
     ↓
new primary
```

But there's a dangerous subtlety.

---

# 26. Retry + Database Writes

Suppose:

```text
Client
  ↓
MySQL
  ↓
COMMIT succeeds
  ↓
network response lost
```

The client sees:

```text
"Request failed."
```

So it retries.

But the original transaction actually succeeded.

Now we could execute the operation twice.

This is why retries around writes require **idempotency**.

Which you've already learned in Module 7/8.

For example:

```text
request_id = abc123
```

The application/database can ensure:

```text
abc123
```

is processed only once.

Again, Module 9 is connecting real technologies to concepts you've already learned.

---

# 27. Failure Scenario: Primary Dies During a Write

Let's make this concrete.

```text
Client
  │
  ▼
Primary
  │
  ├── transaction
  ├── commit
  │
  X crash
```

The client doesn't know:

```text
Did it commit?
```

Possibilities:

```text
Commit never happened
Commit happened but response was lost
Commit happened and replication happened
Commit happened but replica didn't receive it
```

This is why distributed failure handling is hard.

The application can't always simply say:

> "No response = operation failed."

The operation may have succeeded.

---

# 28. This Is Why Databases Need Careful Failure Semantics

A mature HLD discussion should distinguish:

```text
Availability
Durability
Consistency
Recoverability
```

For example:

```text
Async replication
→ good availability
→ good read scaling
→ potential lag
→ possible loss of newest writes during certain failures

Synchronous replication
→ stronger durability
→ higher coordination/latency
→ potentially more sensitivity to replica/network failures
```

There isn't one universally correct configuration.

---

# 29. A Realistic MySQL Architecture

For a moderately large application, we might end up with:

```text
                       Application
                           │
                    DB access layer
                           │
              ┌────────────┴────────────┐
              │                         │
            Writes                    Reads
              │                         │
              ▼                         ▼
          ┌────────┐              ┌──────────┐
          │Primary │─────────────▶│ Replica 1│
          └────────┘              └──────────┘
              │                   ┌──────────┐
              └──────────────────▶│ Replica 2│
                                  └──────────┘
```

With:

```text
Backups
   +
Monitoring
   +
Failover mechanism
```

And if the dataset/write volume becomes too large:

```text
                 Application
                     │
                Shard Router
             /       |       \
            ▼        ▼        ▼
         Shard A  Shard B  Shard C
           │         │        │
         P + R      P + R    P + R
```

That's a realistic evolution.

---

# 30. The Architectural Decision Tree

When designing with MySQL, think:

```text
                 MySQL
                   │
            Is one instance enough?
              /           \
            Yes            No
             │              │
             ▼              ▼
          Simple       What is limiting us?
                           │
                 ┌─────────┼─────────┐
                 ▼         ▼         ▼
               Reads     Writes    Storage
                 │         │         │
                 ▼         ▼         ▼
             Replicas   Sharding   Sharding
```

But before adding anything:

```text
Can indexes solve it?
Can query optimization solve it?
Can caching solve it?
Can read replicas solve it?
```

Don't jump to distributed architecture unnecessarily.

---

# 31. The Most Important MySQL Insight So Far

We can now summarize MySQL's evolution:

```text
Single MySQL
     │
     ├── Indexes
     │      ↓
     │   efficient reads
     │
     ├── Transactions
     │      ↓
     │   correctness
     │
     ├── Replication
     │      ↓
     │   availability + read scaling
     │
     └── Sharding
            ↓
       horizontal scale
```

But every step introduces tradeoffs:

```text
Indexes
→ write overhead

Transactions
→ coordination

Locks
→ contention

Replication
→ lag

Failover
→ distributed coordination

Sharding
→ cross-shard complexity
```

This is the core of technology-level system design.

---

# What You Should Be Able to Defend in an Interview

If I ask:

> **"Why are you using MySQL replicas?"**

Don't say:

> "For scalability."

Say:

> "Our workload is read-heavy, so replicas allow us to distribute read traffic while keeping writes on the primary. We accept replication lag, so reads requiring immediate read-after-write consistency will either go to the primary or use an appropriate routing strategy."

That's an **architectural answer**.

If I ask:

> **"Why not just add more replicas to handle writes?"**

You should immediately say:

> "Because the replicas are copies of the same dataset; they don't independently accept the same write workload in the normal primary-replica model. To distribute write responsibility, we'd need to partition/shard the data or use a different architecture."

That's the distinction I want you to internalize.

---

## Where We Go Next

We've now covered the major MySQL building blocks:

```text
Relational model
      ↓
Indexes / B+ Trees
      ↓
Query execution
      ↓
Transactions
      ↓
Isolation
      ↓
Locking + MVCC
      ↓
Replication
      ↓
High Availability
```

The next major question is:

> **"Okay, but when should I actually choose MySQL over PostgreSQL, MongoDB, Cassandra, DynamoDB, etc.?"**

That's where we'll shift from **"How does MySQL work?"** to **"How do I defend MySQL as a technology choice?"**

We'll compare **MySQL vs PostgreSQL** first, because they're the closest alternatives and the comparison forces us to understand what actually matters when selecting a relational database.
