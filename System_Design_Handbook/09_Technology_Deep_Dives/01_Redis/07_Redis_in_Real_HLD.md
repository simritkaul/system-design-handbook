# Redis — Part 7: Choosing Redis in Real HLD Scenarios

We've now learned enough Redis internals to stop asking:

> "What features does Redis have?"

and start asking the question that actually matters in an SDE II interview:

> **"Given this requirement, why would I choose Redis — and why not something else?"**

This is the point of Module 9.

---

# 1. The Technology-Selection Mindset

Suppose an interviewer says:

> "We're building a food-delivery application. We need to track the current location of millions of users."

Don't jump to:

> "Use Redis."

Instead:

```text
Requirement
    ↓
What data?
    ↓
What access pattern?
    ↓
What latency?
    ↓
What consistency?
    ↓
What scale?
    ↓
What durability?
    ↓
What alternatives?
    ↓
Technology choice
```

Redis should be the **conclusion of reasoning**, not the starting point.

Let's practice that reasoning.

---

# 2. Use Case #1 — Caching

This is the obvious one.

Suppose:

```text
Product Service
      ↓
Database
```

We have:

```text
10,000 requests/sec
```

but most requests repeatedly ask for the same products.

Without caching:

```text
10,000 requests/sec
       ↓
   Database
```

With Redis:

```text
                    ┌── Redis HIT ──→ Response
                    │
Request → Application
                    │
                    └── MISS ──→ Database
                                  ↓
                                Redis
```

Why Redis?

Because the workload is:

```text
Key-based lookup
+
Very frequent reads
+
Low latency
+
Data can be reconstructed
```

That's a strong Redis workload.

---

## But why Redis instead of an in-process cache?

Good interview question.

Suppose we have:

```text
Server A
Server B
Server C
```

Each application instance could have its own memory cache:

```text
A → local cache
B → local cache
C → local cache
```

That's extremely fast.

But now:

```text
A has product = X

B has product = Y
```

The caches can diverge.

A shared Redis cache gives:

```text
A ─┐
B ─┼──→ Redis
C ─┘
```

Now all instances can access the same cache.

So:

> **Redis becomes attractive when we need a shared, low-latency cache across application instances.**

---

# 3. But Why Not Use the Database Directly?

Because we have to ask what the database is doing.

Suppose:

```text
Database
   ↓
Complex query
   ↓
10 ms
```

and we're performing it:

```text
100,000 times/sec
```

Even if the database _can_ handle it, we're making it perform work that doesn't need to happen repeatedly.

Redis allows:

```text
First request
   ↓
Database
   ↓
Redis

Next 99,999 requests
   ↓
Redis
```

So Redis isn't necessarily replacing the database.

It is often **protecting the database from unnecessary repeated work**.

---

# 4. Use Case #2 — Session Store

Imagine an application with:

```text
10 application servers
```

A user's session could live in server memory:

```text
User
  ↓
Server A
  ↓
session
```

But what happens if the next request reaches:

```text
Server B
```

Server B doesn't have the session.

We could use sticky sessions, but now we're coupling users to particular servers.

Instead:

```text
Server A ─┐
Server B ─┼──→ Redis
Server C ─┘
```

Session:

```text
session:user123
```

Redis gives every application server shared access.

Why is Redis a good candidate?

```text
Session
   ↓
small object
   ↓
key-based lookup
   ↓
high frequency
   ↓
TTL
```

That combination fits Redis extremely well.

---

# 5. Why TTL Is Particularly Useful Here

Sessions naturally expire.

For example:

```text
session:user123
TTL = 30 minutes
```

If the user doesn't interact:

```text
30 minutes
    ↓
session automatically disappears
```

We don't need a background process constantly cleaning old sessions.

This is an example of something we've learned repeatedly:

> **A technology becomes especially valuable when one of its primitives naturally matches the workload.**

Redis's TTL isn't merely a feature.

It's a reason Redis fits session storage well.

---

# 6. Use Case #3 — Distributed Rate Limiter

Suppose an API has:

```text
100 requests / minute / user
```

and we have:

```text
20 application servers
```

A local counter doesn't work well:

```text
Server A → 80 requests
Server B → 80 requests
```

Each server thinks:

```text
80 < 100
```

but globally:

```text
160 > 100
```

We need shared state.

Redis is a strong candidate:

```text
               ┌── Server A
               │
Requests ──────┼── Server B
               │
               └── Server C
                      │
                      ▼
                    Redis
                      │
                request:user123
```

The counter can be updated atomically.

This gives us:

```text
Shared state
+
Atomic increment
+
TTL
+
Low latency
```

Again, notice the pattern:

> We're not choosing Redis because "Redis is fast."

We're choosing it because **its primitives match the problem**.

---

# 7. Use Case #4 — Leaderboard

Suppose we're building a gaming platform.

We need:

> "Show the top 100 players."

Our data looks conceptually like:

```text
Player A → 5000
Player B → 4200
Player C → 7000
```

We need:

```text
7000
5000
4200
...
```

This is where Redis's **sorted set** becomes interesting.

Conceptually:

```text
Leaderboard
────────────────
Player C   7000
Player A   5000
Player B   4200
```

Now operations such as:

```text
Update player's score
Find top players
Find player's rank
```

map naturally to the data structure.

This is a beautiful technology-selection example.

We don't say:

> "Redis has sorted sets."

We say:

> **"The workload is dominated by ordered ranking operations, and Redis provides an in-memory ordered data structure optimized for this access pattern."**

That's an HLD answer.

---

# 8. Why Not MySQL for the Leaderboard?

You absolutely _could_ use MySQL.

For example:

```sql
SELECT *
FROM players
ORDER BY score DESC
LIMIT 100;
```

The question isn't:

> "Can MySQL do it?"

Almost every technology can do something.

The question is:

> **"Which technology makes the workload cheap at our scale?"**

If leaderboard updates and reads are extremely frequent:

```text
Millions of updates
+
Millions of rank queries
+
Very low latency
```

Redis may be attractive.

If the leaderboard is:

```text
Small
+
Infrequently updated
+
Part of a larger relational workload
```

MySQL might be perfectly reasonable.

That's the nuance interviewers want.

---

# 9. Use Case #5 — Distributed Lock

Suppose we have:

```text
10 application servers
```

and exactly one should process:

```text
monthly-billing-job
```

We can use a shared coordination mechanism backed by Redis.

Conceptually:

```text
Server A ─┐
Server B ─┼──→ Redis
Server C ─┘
```

One acquires:

```text
billing-lock
```

Others don't.

Redis is attractive because:

```text
Fast
+
Shared
+
Atomic primitives
+
Expiration/leases
```

But this is where we need to show maturity.

Don't say:

> "Redis is perfect for distributed locks."

Say:

> **"Redis can provide primitives for distributed coordination, but correctness requirements around expiration, failover, ownership, and network failures need careful consideration."**

That's a much stronger answer.

---

# 10. Use Case #6 — Counters

Suppose we're building a video platform.

We need:

```text
video:123:view_count
```

and millions of users are watching simultaneously.

A naïve approach:

```text
GET count
+
SET count + 1
```

creates a race condition.

Redis gives us atomic increment semantics:

```text
INCR video:123:view_count
```

So:

```text
Request
   ↓
Redis INCR
   ↓
counter
```

is extremely attractive.

But there's another question:

> **Should Redis be the permanent source of truth for views?**

Not necessarily.

We might instead use Redis for fast aggregation and eventually persist/flush counts into durable storage.

This is a crucial architectural distinction:

```text
Redis
   ↓
fast temporary aggregation

Database
   ↓
durable business record
```

---

# 11. Use Case #7 — Real-Time State

Consider a multiplayer game.

We might need state such as:

```text
player:123
    position = (120, 450)
    health = 80
    weapon = sword
```

This state may change extremely frequently.

We care about:

```text
very low latency
+
frequent updates
+
fast reads
```

Redis can be attractive.

But now ask:

> "Do we need permanent durability for every intermediate position?"

Probably not.

If the player moves:

```text
100 → 101 → 102 → 103
```

we may not need to persist every intermediate state forever.

This is a workload where an in-memory datastore can make sense because:

> **The current state matters more than the historical intermediate states.**

Again, technology follows requirements.

---

# 12. When Redis Is a Bad Choice

This is just as important.

Let's say someone proposes:

> "Let's put our entire application's database into Redis because it's fast."

Red flag.

Why?

Because the workload may require:

```text
Complex relationships
+
Flexible queries
+
Strong transactional semantics
+
Durable storage
+
Large datasets
+
Rich indexing
```

Redis may not be the best fit.

A relational database may be much more appropriate.

---

# 13. Large Durable Dataset

Suppose:

```text
10 TB of customer records
```

and the requirements are:

```text
Long-term durability
+
Complex queries
+
Transactions
+
Indexes
+
Reporting
```

Putting everything in RAM is likely a poor architectural decision.

The question becomes:

> **Why am I paying RAM prices for data that doesn't need Redis's low-latency characteristics?**

This is a major Redis tradeoff:

> **Memory gives us speed, but memory is expensive.**

---

# 14. Complex Querying

Suppose we need:

> "Find customers in Bangalore who bought a laptop in the last six months, have spent more than ₹2 lakh, and belong to a specific segment."

That's not naturally a Redis workload.

We'd rather use a system designed around:

```text
Filtering
+
Indexing
+
Joins / query execution
+
Complex predicates
```

Redis is primarily attractive when the access pattern is already well understood.

Think:

```text
Redis:
"What value corresponds to this key?"

Database/Search system:
"Find me all records satisfying these conditions."
```

That's a useful mental distinction.

---

# 15. Redis vs MySQL

Let's make the comparison concrete.

| Requirement                 | Redis                                | MySQL                     |
| --------------------------- | ------------------------------------ | ------------------------- |
| Very low latency key lookup | Excellent fit                        | Good, but usually heavier |
| Complex queries             | Limited                              | Strong                    |
| Joins                       | Not the core model                   | Strong                    |
| Transactions                | Limited compared with relational DBs | Strong                    |
| Large durable dataset       | Expensive in RAM                     | Strong fit                |
| TTL-based ephemeral data    | Excellent                            | Possible, but awkward     |
| Counters                    | Excellent                            | Possible                  |
| Shared cache                | Excellent                            | Poor fit                  |
| Leaderboards                | Excellent with suitable structures   | Possible                  |
| Primary business database   | Sometimes                            | Common choice             |

The point isn't:

> "Redis beats MySQL."

It's:

> **They optimize for different workloads.**

---

# 16. Redis vs Local In-Memory Cache

This is another useful interview comparison.

```text
Local cache
    ↓
Fastest possible access
```

because:

```text
Application
   ↓
same process memory
```

But:

```text
Server A cache ≠ Server B cache
```

Redis:

```text
Server A ─┐
Server B ─┼──→ Redis
Server C ─┘
```

slightly increases access cost compared with local memory but gives:

```text
Shared state
+
Centralized management
+
Cross-instance consistency
```

So a strong architecture may even use both:

```text
Application
    ↓
Local cache
    ↓ miss
Redis
    ↓ miss
Database
```

Now we've built a multi-level cache.

---

# 17. Redis vs Kafka

This is a particularly important distinction because both can appear in real-time systems.

Imagine:

> "We need to process user activity."

Someone might say:

> "Use Redis."

But first ask:

> Do we need **current state** or an **event history**?

Redis:

```text
Current state
    ↓
user:123 → online
```

Kafka:

```text
Event history
    ↓
UserLoggedIn
UserViewedVideo
UserAddedToCart
...
```

Redis answers:

> **"What is the state right now?"**

Kafka answers more like:

> **"What happened?"**

This distinction will become very useful when we reach Kafka.

---

# 18. Redis vs S3

Suppose we're storing:

```text
10 million images
```

Redis?

No.

Why?

Because:

```text
Large blobs
+
Long-term storage
+
Cheap capacity
+
Durability
```

are fundamentally different requirements.

Object storage is designed for this.

Redis is optimized for:

```text
small/structured hot data
+
very low latency
```

Again:

> **Technology choice follows workload characteristics.**

---

# 19. A Real HLD Example

Let's design a simplified Instagram-like feed.

Requirements:

```text
100M users
10M daily active users
Feed must load in <100 ms
Posts are persistent
Feed is requested frequently
```

Would we use Redis?

Likely yes — but not as the primary database.

Architecture might conceptually look like:

```text
                 ┌──────────────┐
                 │   Client     │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Application  │
                 └──────┬───────┘
                        │
                 ┌──────┴───────┐
                 ▼              ▼
             Redis          Database
              Feed          Posts
               │
               │ miss
               ▼
           Feed builder
```

Redis might hold:

```text
feed:user123
```

containing IDs of recently relevant posts.

The durable post itself lives elsewhere.

This is a key pattern:

> **Redis often stores the result of expensive computation rather than replacing the durable system that owns the underlying data.**

---

# 20. But What If Redis Loses the Feed?

That's okay if:

```text
Feed can be reconstructed
```

We can do:

```text
Redis lost
   ↓
Rebuild feed
   ↓
Populate Redis
```

This is very different from:

```text
Customer balance lost
```

which cannot simply be reconstructed.

This gives us another powerful design principle:

> **The easier something is to reconstruct, the more comfortable we can be using Redis as an ephemeral layer.**

---

# 21. A Simple Redis Decision Tree

When considering Redis, ask:

```text
                 Do we need Redis?
                       │
                       ▼
             Is latency very important?
                  /          \
                No            Yes
                │              │
             Maybe not         ▼
                         Is access pattern
                           well defined?
                           /        \
                         No          Yes
                         │            │
                      Maybe not       ▼
                              Is data primarily
                               hot/shared state?
                              /          \
                            No            Yes
                            │              │
                         Other tech        ▼
                                  Can data be
                                  reconstructed?
                                  /       \
                                Yes         No
                                │            │
                         Redis is easier   Need stronger
                         to justify        durability/HA
```

Then evaluate:

```text
Scale
Consistency
Durability
Cost
Hot keys
Failure behavior
Alternatives
```

---

# 22. The Redis Interview Answer

If an interviewer asks:

> **"When would you use Redis?"**

A strong answer isn't:

> "Redis is an in-memory database and it's fast."

Instead:

> **"I'd consider Redis when I have a latency-sensitive workload with well-defined access patterns, particularly shared state, key-based lookups, counters, TTL-based data, or specialized in-memory structures. I'd then evaluate whether the data can be reconstructed or whether I need Redis to provide stronger durability, and I'd consider dataset size, scaling, hot keys, consistency, failure behavior, and the alternatives."**

That demonstrates actual architectural reasoning.

---

# 23. Redis — What You Should Now Know

At this point, your Redis mental model should look like:

```text
                         REDIS
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
   Data Model          Execution           Scaling
        │                  │                  │
 Strings/Hashes       Atomic commands      Sharding
 Lists/Sets           Transactions         Replication
 Sorted Sets          Coordination         Cluster
 Streams              Pipelining
        │
        ▼
    Use Cases
        │
 ┌──────┼──────┬────────┬─────────┐
 ▼      ▼      ▼        ▼         ▼
Cache Sessions Counters Leaderboard Locks
```

And above all of that:

```text
                    HLD DECISION
                         │
                         ▼
                  Does Redis fit?
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
           Workload   Tradeoffs   Failure
```

---

# The Redis Chapter Is Now Complete

We've gone far enough.

You don't need to learn every Redis command or become a Redis administrator.

For your SDE II HLD goal, you should now be able to reason about:

- What Redis fundamentally is
- Why in-memory storage matters
- Redis data structures
- Atomic operations
- Transactions and coordination
- Persistence
- Replication
- Failover
- Sharding and Redis Cluster
- Hot keys
- Cache stampede
- Memory constraints
- Replication lag
- Network partitions
- Redis as cache vs source of truth
- Common HLD use cases
- Redis vs relational databases
- Redis vs local cache
- Redis vs event streaming
- Redis vs object storage
- Most importantly: **when Redis is actually the right choice**

And that last point is the reason we spent all this time on Redis.

---

## One Final Exercise Before MySQL

I want you to make the technology decision yourself.

### Scenario

We're building an **e-commerce flash-sale system**.

A popular product has:

```text
Stock: 10,000 units

Traffic:
500,000 requests/sec

Most requests are:
"Is this product available?"
"How many units are left?"
"Try to purchase one."
```

Requirements:

- Very low latency
- Thousands of concurrent purchase attempts
- Stock must never become negative
- The product database should not receive 500k requests/sec
- The system must survive an application-server failure
- Losing a few milliseconds of performance is preferable to selling more items than we actually have

### Question

**Would you use Redis here?**

Don't just answer _yes/no_.

Walk me through:

1. **What would Redis be responsible for?**
2. **What data would you keep in Redis?**
3. **Why is Redis a good fit?**
4. **What would remain in the durable database?**
5. **How would you handle the atomic stock decrement?**
6. **What happens if Redis fails?**
7. **What happens if 500k requests all target the same product?**
8. **What is the biggest Redis-specific risk in this design?**

Give me your architecture and reasoning. I'll challenge it like an HLD interviewer before we move on to **MySQL**.
