## Redis — Part 2: Persistence & Durability

We’ve established the basic model:

> Redis keeps data primarily in memory to provide very fast access.

That immediately creates our next architectural problem.

---

# 1. The Problem

Imagine we use Redis as the source of truth for:

```text
cart:user:42
    ↓
{
    productA: 2,
    productB: 1
}
```

Everything is working beautifully.

Then the Redis server crashes.

```text
Redis
 └── RAM
      └── Cart data
           ↓
        SERVER DIES
           ↓
        RAM LOST
```

If Redis were **only a cache**, this might be fine:

```text
Cache lost
   ↓
Read from database
   ↓
Repopulate cache
```

But if Redis contained the only copy:

```text
Redis lost
   ↓
Data lost
```

So we need to answer:

> **How can an in-memory system preserve data beyond the lifetime of its process or machine?**

That's where Redis persistence comes in.

---

# 2. The Big Idea

> **Redis can periodically or continuously record its in-memory state to persistent storage so that the dataset can be reconstructed after a restart.**

Conceptually:

```text
             Redis
              │
             RAM
              │
       ┌──────┴──────┐
       │             │
       ▼             ▼
   Persistence    Persistence
    mechanism      mechanism
       │             │
       ▼             ▼
   Snapshot       Append log
       │             │
       └──────┬──────┘
              ▼
       Persistent storage
```

Redis gives us different persistence approaches because there's an inherent tradeoff:

> **How much recent data are we willing to lose in exchange for performance and simplicity?**

---

# 3. Two Fundamental Approaches

At a conceptual level, there are two important ways to persist an in-memory dataset.

### Approach A — Periodically save the entire state

Think:

```text
10:00 → Save everything
10:01 → Changes happen
10:02 → Save everything
10:03 → Changes happen
10:04 → Save everything
```

This is a **snapshot**.

If Redis crashes at:

```text
10:03:30
```

the latest persisted state might be:

```text
10:02
```

So changes after that snapshot could be lost.

---

### Approach B — Record changes as they happen

Instead of repeatedly saving the entire dataset:

```text
SET user:42 Alice
INCR views
DEL session:abc
SET cart:42 ...
```

we record the operations.

Conceptually:

```text
Operation 1
Operation 2
Operation 3
Operation 4
...
```

After a crash, Redis can reconstruct its state by replaying those changes.

This is an **append-only log** approach.

---

# 4. Snapshotting

Let's make the first approach concrete.

Suppose Redis contains:

```text
Redis RAM

user:1 → Alice
user:2 → Bob
user:3 → Carol
```

At some point Redis creates a snapshot:

```text
┌─────────────────────┐
│ Persistent Snapshot │
├─────────────────────┤
│ user:1 → Alice      │
│ user:2 → Bob        │
│ user:3 → Carol      │
└─────────────────────┘
```

Now suppose:

```text
user:4 → David
user:5 → Eve
```

Those changes exist in memory, but the previous snapshot doesn't contain them.

If Redis crashes before another snapshot:

```text
RAM
 ├── user:1
 ├── user:2
 ├── user:3
 ├── user:4  ← potentially lost
 └── user:5  ← potentially lost

Persistent snapshot
 ├── user:1
 ├── user:2
 └── user:3
```

After restart, Redis can recover the snapshot.

Therefore:

> **Snapshot persistence gives us a recovery point, not necessarily every recent change.**

---

# 5. Why Use Snapshots?

You might ask:

> Why not save every change immediately?

Because saving the entire dataset periodically can be much simpler than recording every operation forever.

Imagine:

```text
1 billion keys
```

A snapshot represents the state at a particular point in time.

It's conceptually:

```text
Current state
     ↓
Save state
     ↓
Done
```

This can also be useful for backups.

For example:

```text
                    Redis
                      │
                      ▼
                  Snapshot
                      │
             ┌────────┴────────┐
             ▼                 ▼
          Backup             Recovery
```

The tradeoff is the recovery point.

---

# 6. Append-Only Logging

Now let's take the second approach.

Instead of:

```text
Save entire state
```

we record changes:

```text
SET user:1 Alice
SET user:2 Bob
SET user:3 Carol
SET user:4 David
SET user:5 Eve
```

The persistent representation is essentially:

```text
┌──────────────────────────────┐
│ Operation Log                │
├──────────────────────────────┤
│ SET user:1 Alice             │
│ SET user:2 Bob               │
│ SET user:3 Carol             │
│ SET user:4 David             │
│ SET user:5 Eve               │
└──────────────────────────────┘
```

After a crash:

```text
Operation log
      ↓
Replay operations
      ↓
Reconstruct Redis state
```

So instead of recovering:

> "What was the entire dataset at 10:00?"

we can reconstruct:

> "What sequence of changes happened?"

---

# 7. Why Is This Better for Durability?

Imagine:

```text
10:00 → snapshot
10:01 → change
10:02 → change
10:03 → change
10:04 → crash
```

With snapshotting:

```text
Recovered state ≈ 10:00
```

With sufficiently frequent persistence of the change log:

```text
Recovered state ≈ much closer to 10:04
```

So the fundamental difference is:

```text
Snapshot
   ↓
Periodic state

Append-only log
   ↓
Sequence of changes
```

---

# 8. But There's No Free Lunch

Recording every change sounds strictly better.

It isn't.

Suppose your application performs:

```text
5 million writes/second
```

You now have an enormous stream of persistence activity.

The system has to deal with:

- additional disk I/O
- growing log size
- log management
- recovery/replay time
- persistence latency
- storage overhead

So again we're making a tradeoff.

```text
More persistence
      ↓
Better durability
      ↓
Potentially more overhead
```

---

# 9. Recovery Is Also a Cost

Here's a subtle point that's important for HLD.

Suppose we have:

```text
100 GB dataset
+
10 billion logged operations
```

Redis crashes.

To reconstruct everything purely from the beginning:

```text
10 billion operations
        ↓
Replay
        ↓
Current state
```

That could take significant time.

This introduces another concept:

> **Recovery time matters.**

A system isn't highly reliable simply because it technically has a copy of its data.

We also care about:

> **How quickly can it become usable again?**

This is the distinction between durability and availability/recovery.

---

# 10. Durability vs Availability

These concepts are easy to mix up.

### Durability

> If the system acknowledges data, how confident are we that the data survives a failure?

### Availability

> Can the system continue serving requests when something fails?

For example:

```text
Redis A crashes
     ↓
Data persisted safely
     ↓
Data can be recovered
```

That's good durability.

But if:

```text
Redis A crashes
     ↓
No other Redis available
     ↓
Application cannot access Redis
```

we have a temporary availability problem.

Later we'll solve this with **replication and high availability**.

---

# 11. The Most Important Question: What Are We Protecting Against?

When designing persistence, don't simply say:

> "We'll enable persistence."

Ask:

### What failure are we protecting against?

For example:

#### Process restart

```text
Redis process crashes
       ↓
same machine restarts Redis
```

Persistence can help.

#### Machine failure

```text
Machine dies
       ↓
RAM disappears
```

Local persistence may not be enough if the persistent storage is also unavailable.

#### Disk failure

```text
RAM
 ↓
Disk
 ↓
Disk dies
```

Now your persistence mechanism itself needs protection.

#### Entire machine / availability-zone failure

```text
Redis A
   ↓
Entire infrastructure fails
```

Now you need redundancy somewhere else.

This is why persistence alone isn't a high-availability strategy.

---

# 12. Redis Architecture Is Starting to Evolve

Our original system was:

```text
Application
     │
     ▼
   Redis
     │
     ▼
    RAM
```

Persistence adds:

```text
Application
     │
     ▼
   Redis
     │
    RAM
     │
     ▼
Persistent Storage
```

But now we ask:

> What if the Redis machine itself disappears?

Persistence doesn't completely solve that.

So our next architectural evolution becomes:

```text
                 ┌──────────┐
Application ────►│ Redis A  │
                 └────┬─────┘
                      │
                  replication
                      │
                 ┌────▼─────┐
                 │ Redis B  │
                 └──────────┘
```

Now if Redis A fails, Redis B may be able to continue serving the system.

And **that is the next major Redis problem: replication and high availability.**

---

# 13. One Important HLD Trap

Suppose someone says in an interview:

> "Redis has persistence, so we're safe if Redis crashes."

That's an incomplete answer.

I'd push back:

> **"What kind of crash?"**

Because:

```text
Persistence
    ≠
High Availability
```

Persistence answers:

> **Can we recover the data?**

Replication/HA answers:

> **Can another node continue serving while this node is unavailable?**

And these are different architectural concerns.

---

# 14. A Useful Mental Model

Think of Redis as having **three different safety questions**:

```text
                 Redis
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
    Persistence  Replication  Scaling
        │          │          │
   "Can I recover?" "Can I survive?" "Can I handle more?"
```

We'll eventually answer all three.

---

# Key Takeaways

- Redis is primarily memory-based, so persistence is needed when we care about surviving restarts/failures.
- **Snapshots** periodically capture the dataset's state.
- **Append-only logging** records changes so state can be reconstructed.
- Snapshots generally give simpler recovery points but may lose recent changes.
- More aggressive persistence can reduce potential data loss but introduces overhead.
- **Durability and availability are different.**
- Persistence does **not** automatically mean high availability.
- To survive the failure of an entire Redis node while continuing to serve requests, we need **replication and failover**.

### Your turn

Before we move to replication, explain this in your own words:

> **Suppose Redis is being used as the source of truth rather than merely as a cache. Why might we want persistence, and why isn't persistence alone enough to make Redis highly available?**

Don't worry about using the exact terminology. I want your reasoning.
