# Module 9 — Technology Deep Dive

## Chapter 1: Redis

We’ll treat this as **one technology deep dive**, not as a collection of unrelated Redis features.

The goal is to reach the point where, in an SDE II HLD interview, someone says:

> “We need Redis here.”

and you can respond with:

> **“Why Redis? What exactly are we using it for? What happens at scale? What happens if it fails? And why is it better than the alternatives?”**

---

# 1. Why Redis Exists

Let's start with a system we already understand.

Imagine Instagram has an endpoint:

```text
GET /user/123/profile
```

The application needs to retrieve:

```text
User
 ├── id
 ├── name
 ├── profile_picture
 ├── followers
 └── bio
```

Initially:

```text
User
  ↓
Application Server
  ↓
Database
  ↓
Return User
```

This works.

But suppose the same user profile is requested **100,000 times per minute**.

The database now has to repeatedly:

1. receive the request
2. locate the data
3. execute the query
4. return the result

Even if the database is perfectly capable of handling the workload, we're repeatedly doing work for data that may not change very often.

We learned the conceptual solution in Module 3:

> **Cache data that is expensive or unnecessary to retrieve repeatedly.**

So we introduce an in-memory data store:

```text
                    ┌──────────────┐
                    │    Redis     │
                    │   (memory)   │
                    └──────▲───────┘
                           │
Request → Application ─────┤
                           │ miss
                           ▼
                    ┌──────────────┐
                    │   Database   │
                    └──────────────┘
```

Now most requests can avoid the database entirely.

But this immediately raises an interesting question:

> **Why Redis?**

Why not simply use the database?

Why not put the data in the application server's memory?

Why not use some other caching mechanism?

That's the problem Redis is designed to address.

---

# 2. The Big Idea

> **Redis is a fast, network-accessible, in-memory data store that provides useful data structures and operations for workloads where extremely low latency and high throughput matter.**

Notice that I deliberately did **not** define Redis as:

> "a cache."

That's too narrow.

Redis can be used as:

- a cache
- a session store
- a counter store
- a rate limiter
- a distributed coordination mechanism
- a leaderboard
- a queue-like structure
- a temporary data store
- a source of truth for some workloads

The important distinction is:

```text
Redis
  │
  ├── Cache
  ├── Session Store
  ├── Counter
  ├── Rate Limiter
  ├── Leaderboard
  ├── Distributed Lock
  └── ...
```

**Cache is one use case, not Redis's definition.**

---

# 3. What Makes Redis Different?

There are three characteristics we should keep in our head initially.

### 1. Data is primarily held in memory

RAM is dramatically faster to access than persistent storage.

So instead of:

```text
Application
    ↓
Disk-backed database
    ↓
Data
```

Redis can often do:

```text
Application
    ↓
Network
    ↓
RAM
    ↓
Data
```

This makes very low latency possible.

---

### 2. Redis understands data structures

A traditional key-value model might look like:

```text
"user:123" → "Simrit"
```

Redis can instead represent richer structures:

```text
"user:123"
    ↓
Hash
 ├── name → Simrit
 ├── age → 25
 └── city → Delhi
```

Or:

```text
"leaderboard"
    ↓
Sorted Set
 ├── Alice → 5000
 ├── Bob   → 4200
 └── Carol → 3900
```

Or:

```text
"notifications:user:123"
    ↓
List
 ├── Notification A
 ├── Notification B
 └── Notification C
```

The data structure itself provides useful operations.

---

### 3. Redis performs operations close to the data

Suppose we want:

> Increment Alice's score.

We don't necessarily need:

```text
READ Alice's score
        ↓
Application calculates score + 1
        ↓
WRITE new score
```

Redis can perform the operation itself:

```text
Redis
  ↓
INCR / ZINCRBY
  ↓
New value
```

That matters because moving data between the application and the data store unnecessarily adds work and creates concurrency problems.

---

# 4. Redis's Fundamental Model

At the simplest level, Redis has a **keyspace**.

Think:

```text
Key → Value
```

For example:

```text
"user:123" → ...
"session:abc" → ...
"rate:user:123" → ...
"leaderboard" → ...
```

The key identifies the data.

But the value isn't restricted to a simple string.

Conceptually:

```text
Key
 │
 └── Value
      │
      ├── String
      ├── Hash
      ├── List
      ├── Set
      ├── Sorted Set
      └── Stream
```

This is one of the biggest reasons Redis is more interesting than a generic cache.

---

# 5. Why In-Memory Matters

Let's compare two broad approaches.

### Traditional persistent database

```text
Application
     ↓
Database
     ↓
Persistent storage
```

The database is designed around a much broader set of requirements:

- durability
- complex queries
- transactions
- indexes
- large datasets
- persistence
- recovery

That's extremely useful.

But it also means the database has more work to do.

Redis focuses heavily on:

```text
Fast access
+
Simple operations
+
In-memory data
```

So a request might look like:

```text
Application
      │
      │ request
      ▼
   Redis
      │
      │ memory lookup
      ▼
    Value
```

This is particularly useful when the system needs **very frequent, low-latency operations**.

---

# 6. But Here's the Catch

If all of this sounds amazing, there's an obvious question:

> **Why don't we just put our entire database in Redis?**

Because memory is expensive.

And more importantly:

> **Memory is not automatically durable.**

Consider:

```text
Redis
 └── RAM
      └── User data
```

Now the machine crashes.

What happens?

Potentially:

```text
        Redis crashes
             ↓
           RAM lost
             ↓
        Data disappears
```

This is the first major tradeoff.

Redis gives us extremely fast access partly because it keeps data in memory.

But our system now has to answer:

> **Do we care if this data disappears?**

That question determines how we should use Redis.

---

# 7. Redis as a Cache

The easiest case.

Suppose:

```text
Database
   │
   │ source of truth
   ▼
Redis
   │
   │ cached copy
   ▼
Application
```

If Redis loses everything:

```text
Redis crashes
     ↓
Cache empty
     ↓
Application queries DB
     ↓
Data gets cached again
```

This is acceptable.

So:

```text
Database = source of truth

Redis = performance optimization
```

This is an important architectural distinction.

If Redis disappears, **the system becomes slower**, but the data isn't fundamentally lost.

---

# 8. Redis as the Actual Store

Now consider something different.

Suppose we're storing:

```text
online:user:123 → true
```

Maybe losing this information isn't catastrophic because it can be reconstructed.

Fine.

But imagine Redis is the **only place** where we store some important business state.

Now:

```text
Redis
  ↓
Only copy of data
```

If Redis fails and that data isn't recoverable, we've lost actual state.

Therefore Redis's persistence and availability mechanisms suddenly become much more important.

This gives us an important HLD question:

> **Is Redis holding a copy of data, or is Redis holding the data itself?**

The answer changes the architecture.

---

# 9. TTL — Data That Expires Automatically

A particularly useful Redis capability is **expiration**.

Suppose we store:

```text
session:abc123 → user_id=42
```

We may only want the session to exist for 30 minutes.

Instead of having the application constantly clean it up:

```text
Create session
      ↓
30 minutes pass
      ↓
Application deletes session
```

Redis can associate an expiration time with the key:

```text
session:abc123
      │
      ├── user_id = 42
      └── expires in 30 min
```

After the TTL expires:

```text
session:abc123 → gone
```

This is useful for things such as:

- sessions
- temporary tokens
- OTP-related state
- cached responses
- temporary locks
- rate-limit windows

TTL is therefore not merely a caching feature.

It is a **data-lifecycle mechanism**.

---

# 10. Redis Eviction

Now suppose Redis has limited memory.

Eventually:

```text
Redis memory
████████████████████████████
            FULL
```

What happens when we try to insert more data?

Redis can use **eviction policies** to decide which data should be removed.

For example, conceptually:

```text
Memory full
    ↓
Choose a key
    ↓
Evict it
    ↓
Store new key
```

Different policies make different assumptions about which data is least valuable.

For example:

- least recently used
- least frequently used
- random
- TTL-oriented policies

This is another architectural consideration.

If Redis is merely a cache, eviction may be perfectly acceptable.

If Redis contains irreplaceable business data, eviction could be disastrous.

So again:

> **How Redis is being used determines whether a particular behavior is acceptable.**

---

# 11. Why Not Application Memory?

This is an important HLD comparison.

Suppose we have:

```text
Application Server A
    └── Memory

Application Server B
    └── Memory

Application Server C
    └── Memory
```

Each server could maintain its own cache.

That's extremely fast because there's no network hop.

But now:

```text
User → Server A
         │
         └── cache says:
             balance = ₹500

User → Server B
         │
         └── cache says:
             balance = ₹700
```

We have a consistency problem.

Also, if Server A dies:

```text
Server A
   ↓
dies
   ↓
its cache disappears
```

A shared Redis instance gives us:

```text
Server A ──┐
Server B ──┼──→ Redis
Server C ──┘
```

Now application instances can share the same state.

That's one reason a **distributed in-memory store** is useful.

---

# 12. Why Redis Instead of the Database?

Let's make the comparison explicit.

| Requirement                           | Traditional DB | Redis                           |
| ------------------------------------- | -------------- | ------------------------------- |
| Complex relational queries            | Excellent      | Poor fit                        |
| Durable primary storage               | Excellent      | Possible, but depends on design |
| Very low latency                      | Good           | Excellent                       |
| Rich in-memory structures             | Limited        | Excellent                       |
| Simple key-based access               | Good           | Excellent                       |
| TTL / temporary state                 | Possible       | Excellent                       |
| Huge persistent datasets              | Excellent      | Expensive in RAM                |
| Cache workloads                       | Possible       | Excellent                       |
| Transactions / relational constraints | Strong         | Different model                 |

The point isn't:

> Redis is better than a database.

It's:

> **They optimize for different problems.**

---

# 13. The First Important HLD Decision

Imagine you're designing a food-delivery application.

You need to maintain:

```text
restaurant:123 → currently accepting orders
```

Would you put this in Redis?

Probably yes, potentially.

Why?

Because:

- reads may be extremely frequent
- value is simple
- low latency matters
- state may change frequently
- the data can potentially be reconstructed from a durable source

Now consider:

```text
order:984732
    ├── customer
    ├── restaurant
    ├── items
    ├── payment
    └── final amount
```

Would Redis automatically be your first choice?

Probably not.

That's core HLD reasoning:

> **Don't ask "Can Redis store this?"**

Redis can store a huge variety of things.

Ask:

> **"What properties does this workload require?"**

---

# 14. Redis Is a Tool for a Certain Shape of Workload

A useful mental model is:

```text
                  Workload
                     │
          ┌──────────┴──────────┐
          │                     │
     Very low latency       Complex queries
     Key-oriented access    Relationships
     Temporary state        Strong durability
     Counters               Large persistent data
          │                     │
          ▼                     ▼
       Redis                 Database
```

Neither side wins universally.

The architecture depends on the workload.

---

# 15. How Redis Fits Into the Systems We've Already Learned

This is where Module 9 should connect back to the handbook.

### Module 3 — Caching

We learned:

```text
Cache Aside
Write Through
Write Around
Write Back
TTL
Eviction
Cache Stampede
Hot Keys
```

Redis is one technology that can implement these patterns.

For example:

```text
Application
    │
    ├── GET Redis
    │      │
    │      ├── HIT → return
    │      │
    │      └── MISS
    │           ↓
    │        Database
    │           ↓
    │        Redis
    │
    └── return
```

That's the **Cache-Aside pattern** we learned earlier.

Redis didn't invent the concept.

It gives us a concrete system in which we can implement it.

---

### Module 8 — Rate Limiting

We learned:

```text
Rate limit:
User → 100 requests / minute
```

Redis is well suited to maintaining counters:

```text
rate:user:123 → 73
```

with an expiration window.

So:

```text
Module 8 concept
      ↓
Rate Limiting
      ↓
Redis
      ↓
Concrete implementation
```

---

### Module 8 — Distributed Locking

We learned the concept of coordinating multiple distributed processes.

Redis provides primitives that can be used to implement distributed locking.

Again:

```text
Concept
   ↓
Distributed Lock
   ↓
Redis
```

The technology doesn't replace the concept.

It gives us concrete mechanisms with their own tradeoffs.

---

# 16. What Redis Is Particularly Good At

At this point, we can identify a pattern.

Redis is particularly attractive when we have:

### Extremely frequent access

```text
Millions of operations
        ↓
Very low latency requirement
```

### Simple/key-oriented access

```text
key → value
```

### Temporary state

```text
state → expires
```

### Counters

```text
counter++
```

### Ranking

```text
score → sorted leaderboard
```

### Shared distributed state

```text
Server A ──┐
Server B ──┼──→ Redis
Server C ──┘
```

### Data structures that match the problem

```text
Set
Sorted Set
List
Hash
Stream
```

The important thing isn't memorizing this list.

It's recognizing the **shape of the workload**.

---

# 17. Where Redis Doesn't Help

Redis isn't automatically the right answer just because something needs to be fast.

Consider:

> "We need to search employees by salary, department, location, joining date, skills, and perform complex filtering."

That's not naturally Redis's strength.

Or:

> "We need durable storage for 20 TB of historical transactional data."

Keeping all of that in RAM would be expensive.

Or:

> "We need complex joins between customers, orders, products, and payments."

That's a very different workload.

Redis can participate in these systems, but it doesn't mean Redis should replace the primary database.

---

# 18. The Tradeoff

The central Redis tradeoff can be summarized as:

```text
          Redis
            │
            ▼
   Extremely fast access
            │
            │
       But RAM is
       expensive
            │
            ▼
Durability / failure handling
must be designed carefully
```

So we gain:

- very low latency
- high throughput
- useful data structures
- atomic operations
- TTL
- distributed shared state

But we accept considerations around:

- memory cost
- durability
- persistence
- failure
- replication
- eviction
- scaling
- operational complexity

And those are exactly the topics we'll dive into next.

---

# 19. The Redis Architecture Journey

Think of our learning journey as gradually making Redis production-grade.

We start with:

```text
                Application
                    │
                    ▼
                 Redis
                    │
                   RAM
```

Then ask:

> What if Redis crashes?

↓

```text
                Application
                    │
                    ▼
              Redis + Persistence
                    │
                   RAM
                    │
               Persistent data
```

Then:

> What if the Redis machine dies?

↓

```text
              ┌───────────┐
              │  Redis A  │
              └─────┬─────┘
                    │
                 Replica
                    │
              ┌─────▼─────┐
              │  Redis B  │
              └───────────┘
```

Then:

> What if our dataset is too large for one machine?

↓

```text
             Redis Cluster
          /        |        \
       Node A    Node B    Node C
```

Then:

> What if one key becomes extremely hot?

> What if a node fails during a request?

> What consistency do we get?

> What happens during failover?

> When should we choose Redis versus another technology?

That is the actual Redis deep dive.

---

# 20. Where We Are Going

We'll build Redis in roughly this progression:

```text
Redis
 │
 ├── 1. Fundamentals
 │      ├── Data model
 │      ├── Data structures
 │      └── Execution model
 │
 ├── 2. Persistence & Durability
 │
 ├── 3. Replication & High Availability
 │
 ├── 4. Scaling & Redis Cluster
 │
 ├── 5. Atomicity, Transactions & Coordination
 │
 ├── 6. Failure Scenarios
 │
 ├── 7. HLD Use Cases
 │
 ├── 8. Redis vs Alternatives
 │
 └── 9. Technology Selection Scenarios
```

We won't treat this as a checklist where we mechanically cover every Redis feature.

Each topic exists because it answers an architectural question.

---

# Key Takeaways

If you had to walk away from today's chapter with only these ideas:

1. **Redis is not synonymous with cache.** It is an in-memory data store with multiple data structures and use cases.

2. **Redis is fast primarily because its model is optimized around in-memory operations and relatively simple data access.**

3. **Redis can be a cache or a real data store.** The architectural consequences are very different.

4. **TTL and eviction make Redis particularly useful for temporary and bounded-lifetime data.**

5. **Redis is particularly strong for low-latency, high-frequency, key-oriented workloads.**

6. **Redis does not replace a database just because it's faster.** The workload determines the appropriate technology.

7. The important HLD question is not:

   > "Can Redis store this?"

   It's:

   > **"Does this workload benefit from the properties Redis provides enough to justify its tradeoffs?"**

And that's the mindset we'll use for every technology in Module 9.
