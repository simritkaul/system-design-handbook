Up until now, we've been pretending that the cache is just one machine.

But imagine Twitter.

Do you think one Redis server can cache the timelines for 500 million users?

Of course not.

So the question becomes:

> **"How do we scale the cache itself?"**

---

# Module 3 — Chapter 5 — Distributed Caching

> **Goal:** Understand how caches scale across multiple machines, how data is distributed, how failures are handled, and the architectural concepts behind distributed caches.

---

# 1. The Problem

Imagine your application becomes successful.

Initially, your architecture looks like this.

```text
                Users
                  │
                  ▼
            Application
                  │
                  ▼
               Redis Cache
                  │
                  ▼
              MySQL Database
```

Everything works.

---

Months later,

you now have

```text
50 Million Users
```

The Redis server now receives

```text
2 Million Requests/sec
```

Problems begin appearing.

- Memory becomes full.
- CPU reaches 100%.
- Network bandwidth is exhausted.

Buying a larger server helps temporarily.

Eventually,

one machine isn't enough.

---

# 2. The Big Idea

Instead of

one cache,

build

many caches.

```text
          Application

        /      |      \

     Cache1 Cache2 Cache3

         \      |      /

            Database
```

Now

the workload is shared.

This is

**Distributed Caching.**

---

# 3. How Do We Decide Where Data Goes?

Suppose

we have

```text
Cache A

Cache B

Cache C
```

Where should

```text
User123
```

be stored?

There must be a rule.

Usually

the cache key

is hashed.

```text
hash(UserID)

↓

Cache
```

Exactly like

database sharding.

---

# 4. Wait...

Doesn't This Sound Familiar?

It should.

Distributed caches use many of the same ideas we learned earlier.

- Sharding
- Partitioning
- Consistent Hashing
- Replication

Nothing new.

Just applied to cache instead of databases.

---

# 5. Cache Sharding

Instead of

every cache storing everything,

each cache stores

only part of the data.

Example

```text
Cache A

User1

User2

User3

----------------

Cache B

User4

User5

User6

----------------

Cache C

User7

User8

User9
```

Each server owns a subset of the keys.

---

# 6. Why Not Replicate Everything?

Suppose

every cache stored

every key.

```text
Cache A

All Data

------------

Cache B

All Data

------------

Cache C

All Data
```

Memory usage triples.

Adding more cache servers

doesn't increase capacity.

It only increases redundancy.

That's usually too expensive.

---

Instead,

most distributed caches

**partition first**.

Replication is added only when needed for availability.

---

# 7. Cache Replication

Suppose

Cache A crashes.

Without replication

```text
Cache A ❌

↓

All cached data lost.
```

The database suddenly receives thousands of requests.

---

Instead,

replicate the cache.

```text
Primary Cache

↓

Replica Cache
```

If one fails,

another continues serving requests.

---

Notice

the concepts are identical

to database replication.

---

# 8. Cache Failures

Suppose

Cache B crashes.

What happens?

Does the system stop?

No.

Remember

the cache

is

**not the source of truth.**

The application simply

falls back

to the database.

```text
Cache Miss

↓

Database

↓

Repopulate Cache
```

The system becomes slower,

but it still works.

This is a key difference between caches and databases.

---

# 9. Consistent Hashing Returns

Imagine

we add

a fourth cache.

Without

consistent hashing,

almost every key

would move.

Exactly the problem

we solved earlier.

Distributed caches therefore often use **consistent hashing or similar partitioning schemes** to minimize key movement when nodes are added or removed.

---

# 10. Cache Coherency

Suppose

User123

is cached

on

three servers.

The database updates.

Now

all three caches

must eventually

reflect

the new value.

Keeping multiple cached copies synchronized

is called

**Cache Coherency.**

This becomes increasingly important

when caches are replicated.

---

# 11. Local Cache vs Distributed Cache

Imagine

10 application servers.

Each server has

its own memory cache.

```text
App1

↓

Local Cache

----------------

App2

↓

Local Cache

----------------

App3

↓

Local Cache
```

These caches are

extremely fast,

but

they don't share data.

One server updating its local cache doesn't update the others automatically.

---

Distributed Cache

```text
App1

↓

Redis Cluster

↑

App2

↑

App3
```

Now

everyone

shares

the same cache.

---

# 12. Multi-Level Caching

Many production systems

use

both.

```text
Browser Cache

↓

CDN

↓

Application Local Cache

↓

Redis

↓

Database
```

Each level

removes

more load

from the next.

We'll explore browser caches and CDNs in the next chapter.

---

# 13. Where Distributed Caching Shines

### Twitter

User timelines.

---

### Instagram

Feeds.

---

### YouTube

Video metadata.

---

### Amazon

Product pages.

---

### Netflix

Movie information.

---

Any application

serving millions of users.

---

# 14. Tradeoffs

### Advantages

- Scales horizontally
- More total memory
- Higher throughput
- Better fault tolerance (with replication)
- Reduced database load

---

### Disadvantages

- More operational complexity
- Cache coherency challenges
- Network latency between application and cache
- More difficult debugging
- Rebalancing required when nodes change

---

# 15. Real-World Examples

### Redis Cluster

Partitions keys across multiple Redis nodes.

---

### Memcached Cluster

Distributes cached objects across many servers.

---

### Facebook

Massive distributed cache infrastructure for feeds and social graphs.

---

### Twitter

Distributed caches for timelines and user metadata.

---

# 16. Common Interview Questions

- Why can't one Redis server handle everything?
- How do distributed caches scale?
- Why shard a cache?
- Why replicate a cache?
- What happens if a cache node fails?
- Why is consistent hashing useful?
- Local cache vs distributed cache?

---

# 17. Mental Model

Imagine a city library.

One branch

can't hold

every book.

So

the city builds

many branches.

Each branch

stores

part of the collection.

Popular books

may have

multiple copies

at different branches.

If one branch closes,

people visit another branch

or the central archive.

That's exactly how a distributed cache works.

---

# 18. Before vs After

### Single Cache

```text
Users
   │
   ▼
Application
   │
   ▼
Redis
   │
   ▼
Database
```

Simple,

but limited by one machine.

---

### Distributed Cache

```text
                Users
                  │
                  ▼
             Application
                  │
      ┌───────────┼───────────┐
      ▼           ▼           ▼
   Cache A     Cache B     Cache C
      │           │           │
      └───────────┼───────────┘
                  ▼
               Database
```

The load is shared,

capacity increases,

and failures become easier to tolerate.

---

# 19. Connections

We've now learned:

- Why caches exist.
- How applications interact with them.
- How entries are evicted.
- How stale data is handled.
- How caches scale.

The final caching chapter broadens the scope.

Instead of caching inside your backend,

we move the cache closer to the user.

That means studying:

- Browser Cache
- Reverse Proxy Cache
- Edge Cache
- **Content Delivery Networks (CDNs)**

---

# Key Takeaways

- Distributed caching scales cache capacity and throughput by using multiple cache nodes.
- Cache data is typically **partitioned** across nodes, often using **consistent hashing** or a similar distribution mechanism.
- Replication improves availability but increases memory usage and synchronization complexity.
- Unlike databases, caches are **not the source of truth**, so cache failures generally degrade performance rather than correctness.
- Real-world systems often combine **local caches**, **distributed caches**, and **CDNs** into a multi-level caching hierarchy.

---

## One refinement I'd make

This chapter reveals something interesting:

We're no longer learning isolated topics. We're seeing the same core ideas reappear in different contexts.

| Database Concept     | Cache Equivalent        |
| -------------------- | ----------------------- |
| Sharding             | Cache Partitioning      |
| Replication          | Cache Replication       |
| Consistent Hashing   | Cache Node Distribution |
| Failover             | Cache Failover          |
| Eventual Consistency | Cache Coherency         |

That's a pattern you'll keep seeing throughout system design. Once you understand a distributed systems concept deeply, you'll recognize it being reused in databases, caches, message brokers, storage systems, and even load balancers. That's why learning the concepts first pays off so well.
