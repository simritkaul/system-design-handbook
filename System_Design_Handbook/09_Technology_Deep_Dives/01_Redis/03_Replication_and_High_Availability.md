# Redis — Part 3: Replication & High Availability

We've now established an important distinction:

```text
Persistence
    ↓
"Can I recover my data?"
```

But imagine we're running a production system and our Redis server suddenly dies.

Even if we can recover the data from disk, we may have a problem:

```text
Redis dies
   ↓
Application requests
   ↓
?????
```

Recovery might take seconds or minutes.

For some systems, that's unacceptable.

So the next question is:

> **Can we have another Redis server already holding the data and ready to take over?**

That's the problem **replication** addresses.

---

# 1. The Problem

Suppose we have:

```text
              Application
                   │
                   ▼
              ┌─────────┐
              │ Redis A │
              └─────────┘
```

Everything goes through Redis A.

Now:

```text
Redis A
   ↓
machine failure
   ↓
Redis unavailable
```

Even if we have persistence:

```text
Redis A
   ↓
     💥
   ↓
Persistent data exists somewhere
```

we still need to:

1. start another Redis instance
2. load/recover the data
3. make the application connect to it
4. resume serving traffic

That is **recovery**, not immediate availability.

For many systems, we'd rather have:

```text
              Application
                   │
                   ▼
              ┌─────────┐
              │ Redis A │
              └────┬────┘
                   │
               replication
                   │
              ┌────▼────┐
              │ Redis B │
              └─────────┘
```

Now Redis B already has a copy.

If A fails:

```text
Redis A 💥
    │
    X
    │
Redis B
    │
    ▼
Continue serving
```

That's the fundamental idea.

---

# 2. The Big Idea

> **Replication keeps copies of Redis's data on multiple nodes so that the system doesn't depend on a single Redis server.**

The most common basic model is:

```text
Primary
   │
   │ changes
   ▼
Replica
```

The primary accepts writes.

The replica receives those changes and maintains a copy.

For example:

```text
Application
    │
    │ SET user:42 Alice
    ▼
Primary
    │
    │ replicate
    ▼
Replica
```

Both eventually contain:

```text
user:42 → Alice
```

---

# 3. Why Not Just Have Two Independent Redis Servers?

We need to be careful here.

This:

```text
Redis A

Redis B
```

doesn't automatically mean replication.

If the application writes:

```text
SET user:42 Alice
```

to A but not B:

```text
A → user:42 = Alice

B → doesn't know
```

we don't have redundancy.

Replication means there is a mechanism that keeps the datasets related:

```text
Primary
   │
   ├── change 1 ──→ Replica
   ├── change 2 ──→ Replica
   └── change 3 ──→ Replica
```

---

# 4. Primary and Replica

Let's use terminology carefully.

We have:

```text
        Primary
           │
      writes happen
           │
           ▼
        Replica
```

The **primary** is responsible for accepting writes.

The **replica** maintains a copy.

This gives us:

```text
                  ┌──────────┐
                  │ Primary  │
                  └────┬─────┘
                       │
                replication
                       │
                ┌──────▼──────┐
                │   Replica   │
                └─────────────┘
```

We can even have multiple replicas:

```text
                    Primary
                   /   |    \
                  /    |     \
                 ▼     ▼      ▼
             Replica Replica Replica
```

Now several machines have copies of the data.

---

# 5. What Does Replication Actually Buy Us?

There are several benefits, but let's separate them.

## A. Failure Recovery

If the primary disappears:

```text
Primary 💥
```

we have another node with the data:

```text
Replica
   ↓
has copy
```

This is the foundation for failover.

---

## B. Read Scaling

Suppose our workload is:

```text
99% reads
1% writes
```

Sending every request to one Redis server may become unnecessary.

We could potentially distribute reads:

```text
                 ┌── Replica A
                 │
Application ─────┼── Replica B
                 │
                 └── Replica C

                 Primary
                    ↑
                 Writes
```

Now multiple nodes can serve read traffic.

This gives us a second benefit:

> Replication can sometimes increase read capacity.

But there's an important catch.

---

# 6. Replicas Can Be Behind

This is one of the most important Redis HLD concepts.

Suppose:

```text
Primary
   │
   │
   ▼
user:42 → balance = 500
```

The application updates it:

```text
SET balance = 700
```

The primary now has:

```text
balance = 700
```

But the replica might temporarily still have:

```text
balance = 500
```

because replication takes time.

So:

```text
Primary
balance = 700

Replica
balance = 500
```

This is called **replication lag**.

---

# 7. Why Replication Lag Matters

Imagine:

```text
User updates profile
        ↓
Write → Primary
        ↓
Immediately read → Replica
```

The user might see:

```text
Old profile
```

even though they just updated it.

This is not necessarily a Redis bug.

It's a consequence of distributing copies of data.

We have encountered this concept before in Module 2:

> **Replication can introduce consistency tradeoffs.**

Redis gives us a concrete example of that concept.

---

# 8. Primary vs Replica: A Simple Timeline

Imagine:

```text
T1
Primary: 100
Replica: 100

T2
Primary: 200
Replica: 100

T3
Primary: 200
Replica: 200
```

During T2:

```text
Primary ≠ Replica
```

Eventually:

```text
Primary = Replica
```

assuming replication catches up.

That's why this is commonly described as **eventual consistency between replicas** in the replication path.

The important interview insight isn't memorizing the phrase.

It's understanding:

> **If I read from a replica, I may not necessarily see the latest write.**

---

# 9. Now We Hit a Bigger Problem

Suppose:

```text
              Application
                   │
                   ▼
                Primary
                /     \
               ▼       ▼
           Replica A  Replica B
```

Primary suddenly dies.

What happens?

We have:

```text
Application
     │
     ▼
Primary 💥
```

We have replicas, but who tells the application:

> "Stop using Primary. Use Replica A instead."

Someone or something has to coordinate that.

This brings us to **failover**.

---

# 10. Failover

Failover means:

> **When the current primary becomes unavailable, another node is promoted to become the new primary.**

Conceptually:

```text
Before:

Primary A
   │
   ├── Replica B
   └── Replica C
```

Then:

```text
A 💥
```

The system decides:

```text
B → New Primary
```

Now:

```text
Primary B
   │
   └── Replica C
```

The application needs to be directed toward B.

---

# 11. But Who Makes That Decision?

This is where high availability becomes much more interesting.

Imagine:

```text
          Primary A
             │
        ┌────┴────┐
        ▼         ▼
    Replica B  Replica C
```

A network problem occurs.

The replicas can't communicate with A.

But perhaps A is actually still alive.

Now we have:

```text
B thinks:
"A is dead."

A thinks:
"I'm alive."
```

If B simply declares itself primary, we could potentially end up with:

```text
        Primary A
           │
           │
           X
           │
        Primary B
```

Two nodes both accepting writes.

This is the classic **split-brain** problem.

And this is why high availability isn't simply:

> "Put two Redis servers next to each other."

We need coordination.

---

# 12. High Availability Is More Than Replication

This is a very important distinction:

```text
Replication
    ↓
Keep copies of data

High Availability
    ↓
Keep the service operational
despite failures
```

Replication is one building block of HA.

But HA also needs mechanisms for things like:

- detecting failures
- deciding which node should become primary
- coordinating failover
- preventing conflicting primaries
- redirecting clients

Conceptually:

```text
                 ┌──────────────┐
                 │ Coordination │
                 └──────┬───────┘
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
          Primary              Replica
              │                   │
              └──── replication ──┘
```

Redis has mechanisms for this, which we'll get to shortly.

---

# 13. What Happens to Writes During Failover?

Here's another subtle problem.

Suppose:

```text
Primary
   │
   ├── write A
   ├── write B
   ├── write C
   │
   └── crashes
```

But the replica received:

```text
write A
write B
```

and never received:

```text
write C
```

Now we promote the replica.

The new primary has:

```text
A
B
```

but not:

```text
C
```

So a failover can potentially result in the loss of writes that hadn't reached the replica.

This is why replication introduces another architectural question:

> **How much replication lag are we willing to tolerate?**

---

# 14. The Durability + Replication Picture

Now we can see that several independent mechanisms exist:

```text
                   Redis
                     │
          ┌──────────┴──────────┐
          │                     │
     Persistence            Replication
          │                     │
          ▼                     ▼
   Recover after crash     Survive node failure
```

And they solve different problems.

### Persistence

```text
"Redis restarted.
Can I reconstruct my state?"
```

### Replication

```text
"Redis A died.
Does another node already have my data?"
```

### Failover

```text
"Redis A died.
Can Redis B become the new primary?"
```

This separation is extremely useful in HLD interviews.

---

# 15. Does Replication Mean We Can Throw Away Persistence?

Not necessarily.

Consider:

```text
Primary A
    │
    ▼
Replica B
```

Suppose both are in the same machine or infrastructure failure domain.

A catastrophic failure could affect both.

Or:

```text
Primary A
   ↓
corruption / bad operation
   ↓
Replica B
```

Replication copies changes.

That means replication is not automatically the same thing as backup.

If bad data is replicated:

```text
Bad write
   ↓
Primary
   ↓
Replica
```

both can contain the bad state.

So:

> **Replication protects against some failures; backups/persistence protect against others.**

Again, different mechanisms solve different failure modes.

---

# 16. Where Replication Helps

Replication is particularly useful when:

### Availability matters

```text
One node dies
     ↓
Another node takes over
```

### Read traffic is large

```text
Many reads
    ↓
Distribute across replicas
```

### We need redundancy

```text
Multiple copies
    ↓
Less dependence on one machine
```

---

# 17. Where Replication Doesn't Magically Help

Replication isn't a free scaling button.

If you have:

```text
100 million writes/sec
```

and one primary handles all writes:

```text
                Primary
              /    |    \
             ▼     ▼     ▼
          Replica Replica Replica
```

Adding replicas doesn't automatically distribute those writes.

The primary may remain the bottleneck.

This is an important distinction:

```text
Replication
     ↓
Primarily redundancy / read scaling

Sharding
     ↓
Distributing the dataset and write workload
```

And that leads directly into our next major Redis problem.

---

# 18. Replication vs Sharding

We've already learned sharding in Module 2.

Now we can connect it to Redis.

### Replication

Creates copies:

```text
             Data
              │
        ┌─────┼─────┐
        ▼     ▼     ▼
        A     B     C
```

All nodes broadly contain the same dataset.

### Sharding

Splits the dataset:

```text
          Entire Dataset
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
    Shard A  Shard B  Shard C
    users     users     users
    1-1000   1001-2000 2001-3000
```

Different nodes own different portions.

And in a production architecture, we often combine them:

```text
              Dataset
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
      Shard A  Shard B  Shard C
        │        │        │
      Replica  Replica  Replica
```

Now we're getting toward a genuinely scalable Redis architecture.

---

# Key Takeaways

The important ideas from this part are:

1. **Replication creates additional copies of Redis data.**

2. A common model is:

   ```text
   Primary → Replica
   ```

3. Replicas can improve **availability** and potentially **read capacity**.

4. Replicas may lag behind the primary, so reading from them can introduce **stale reads**.

5. **Failover** means promoting another node when the primary fails.

6. High availability requires more than replication—it also requires **failure detection and coordinated failover**.

7. Replication does not guarantee that every write survives a primary failure. A write not yet replicated may be lost.

8. **Replication ≠ persistence ≠ backup.**

   They protect against different failure modes.

9. **Replication and sharding solve different problems:**
   - Replication → copies/redundancy/read scaling
   - Sharding → distributing data/workload

10. Real Redis architectures often combine both:

```text
Sharding
   +
Replication
   +
Persistence
   +
Failover
```

Next, we'll tackle **Redis scaling and clustering**—the question that naturally follows:

> **What happens when one Redis machine simply isn't big or powerful enough for our dataset and traffic?**
