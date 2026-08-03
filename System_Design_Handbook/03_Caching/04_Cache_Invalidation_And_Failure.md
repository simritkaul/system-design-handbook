There's an old saying in computer science:

> **"There are only two hard things in Computer Science: cache invalidation and naming things."**

It's meant as a joke—but only partly. Cache invalidation is difficult because it forces you to balance **freshness**, **performance**, and **complexity**.

This chapter is also one of the most practical in HLD interviews.

---

# Module 3 — Chapter 4 — Cache Invalidation & Cache Failure Scenarios

> **Goal:** Understand why caches become stale, the major invalidation strategies, and common failure scenarios like cache stampedes, hot keys, and cache penetration.

---

# 1. The Problem

Suppose we cache

```text id="rt2t4v"
Product 123

↓

₹999
```

A user requests it.

The cache returns

```text id="okl86w"
₹999
```

Everything works.

---

Now the seller updates the price.

```text id="zpwjfe"
₹999

↓

₹899
```

The database updates successfully.

But...

The cache still contains

```text id="yy9z9h"
₹999
```

Users continue seeing

the old price.

The cache is now

**stale**.

---

# 2. The Big Idea

Caching introduces a second copy of your data.

Whenever the source of truth changes,

you must decide:

> **How do I keep the cached copy correct?**

This process is called

**Cache Invalidation.**

---

# 3. Strategy 1 — TTL (Time-To-Live)

This is the simplest approach.

Every cached item gets an expiration time.

Example

```text id="5xwws3"
Product

↓

10 minutes
```

After

10 minutes

the cache automatically removes it.

The next request fetches fresh data.

---

## Advantages

- Extremely simple
- No manual invalidation
- Works well for slowly changing data

---

## Disadvantages

Imagine

the price changes

after

30 seconds.

Users still see

stale data

for

9 minutes and 30 seconds.

---

# 4. Strategy 2 — Write Invalidate

Instead of waiting,

remove the cached item immediately after updating the database.

Flow

```text id="3r1jvz"
Update Database

↓

Delete Cache Entry

↓

Next Read

↓

Database

↓

Fresh Cache
```

Notice something.

We don't update the cache.

We simply remove it.

The next read repopulates it.

This works naturally with **Cache Aside**.

---

## Advantages

- Simple
- Fresh data appears on the next read
- Avoids updating cache unnecessarily

---

## Disadvantages

The first read after invalidation experiences a cache miss.

---

# 5. Strategy 3 — Write Update

Instead of deleting,

update both

the database

and

the cache.

```text id="p6gk6n"
Database

↓

Cache
```

Now future reads

see the latest value immediately.

---

## Advantages

- No stale cache
- No cache miss after update

---

## Disadvantages

You may update cache entries that nobody reads again.

More write work.

---

# 6. Which Strategy Should I Choose?

Generally:

**Frequently changing but heavily read data**

→ Update the cache.

---

**Most web applications**

→ Invalidate the cache.

---

**Slowly changing data**

→ TTL may be sufficient.

---

# 7. Cache Stampede (Thundering Herd)

This is a famous interview question.

Suppose

a popular product

expires.

```text id="e76u6o"
iPhone
```

One million users request it

at exactly the same moment.

Every request experiences

a cache miss.

All of them hit

the database simultaneously.

```text id="2x9bka"
1,000,000 Requests

↓

Database
```

The cache intended to protect the database.

Instead,

its expiration caused the database to be overwhelmed.

This is called

**Cache Stampede**.

---

# 8. How Do We Prevent Cache Stampede?

Several approaches exist.

### A. Request Coalescing (Single Flight)

The first request

fetches the data.

Everyone else waits.

```text id="srn0zw"
1000 Requests

↓

One Database Query

↓

Shared Result
```

Only one request regenerates the cache.

---

### B. Background Refresh

Refresh popular cache entries

before

they expire.

Users never see a cache miss.

---

### C. Randomized TTL

Instead of

everything expiring at

exactly

10:00,

spread expirations out.

Example

```text id="7i2bff"
10 minutes ± random offset
```

This prevents many popular keys from expiring simultaneously.

---

# 9. Cache Penetration

Suppose someone repeatedly requests

```text id="4u9vzu"
User ID

999999999
```

That user

doesn't exist.

Cache

↓

Miss.

Database

↓

No user.

Next request.

Same thing.

Again.

Again.

The cache never stores anything because the data doesn't exist.

Attackers can exploit this to overload your database.

This is called

**Cache Penetration**.

---

## Solutions

### Cache Negative Results

Store

```text id="l6bho2"
Not Found
```

for a short TTL.

Future requests hit the cache instead of the database.

---

### Bloom Filters

A probabilistic data structure that quickly answers:

> "Could this key possibly exist?"

If the Bloom filter says **definitely not**,

skip the database entirely.

(We'll study Bloom Filters later as a separate concept.)

---

# 10. Cache Breakdown (Hot Key Expiration)

Imagine one key.

```text id="hvlqyo"
Trending Video
```

Millions of requests.

Then...

that single key expires.

Every request now misses the cache.

One hot item overwhelms the database.

Unlike a stampede involving many keys,

this problem centers on **one extremely popular key**.

---

## Solutions

- Never let extremely hot keys expire unexpectedly.
- Refresh them proactively.
- Use request coalescing.
- Extend TTL while traffic remains high.

---

# 11. Hot Keys

Suppose

one celebrity posts.

One billion users request

the same profile.

One cache server now receives almost all traffic.

The bottleneck shifts

from the database

to the cache itself.

---

## Solutions

- Replicate hot cache entries across multiple cache nodes.
- Load balance requests.
- Detect hot keys and treat them specially.

---

# 12. Visual Summary

| Problem           | Cause                            | Solution                               |
| ----------------- | -------------------------------- | -------------------------------------- |
| Stale Cache       | Database changed                 | TTL, invalidate, or update             |
| Cache Stampede    | Many requests after expiration   | Single flight, refresh, randomized TTL |
| Cache Penetration | Requests for missing data        | Negative caching, Bloom Filter         |
| Cache Breakdown   | One hot key expires              | Refresh early, extend TTL              |
| Hot Key           | One key receives extreme traffic | Replication, load balancing            |

---

# 13. Mental Model

Imagine a restaurant.

Everyone orders

the special dish.

The kitchen prepares it in advance.

Now imagine

the prepared dish runs out.

Every waiter rushes to the chef

at the same time.

The chef becomes overwhelmed.

That's a cache stampede.

A better restaurant starts preparing another batch

before

the first one runs out.

---

# 14. Tradeoffs

There is no perfect invalidation strategy.

- **TTL** is simple but can serve stale data.
- **Invalidate-on-write** minimizes stale data but causes cache misses.
- **Update-on-write** keeps data fresh but increases write cost.
- Preventing cache failures often requires additional coordination and complexity.

---

# 15. Real-World Examples

### Amazon

Product price updates invalidate or refresh cached product data.

---

### YouTube

Trending videos are proactively refreshed because they're accessed constantly.

---

### Google

Frequently searched results are cached with carefully managed refresh strategies.

---

### Social Media

Celebrity profiles and viral posts are classic hot-key scenarios.

---

# 16. Common Interview Questions

- What is cache invalidation?
- TTL vs invalidate-on-write?
- What is a cache stampede?
- How would you prevent a thundering herd?
- What is cache penetration?
- What is negative caching?
- What is a hot key?
- How is cache breakdown different from cache stampede?

---

# 17. Connections

We've now answered

- Why caches exist.
- How applications use them.
- How entries are removed.
- How to keep them fresh.
- What can go wrong.

The next step is to scale the cache itself.

Questions like:

- What if one Redis server isn't enough?
- How do we distribute cached data?
- Should caches be replicated?
- How does Redis Cluster work?
- Where does consistent hashing fit?

That takes us to:

> **Distributed Caching**

---

# Key Takeaways

- Cache invalidation keeps cached data synchronized with the source of truth.
- Common strategies are **TTL**, **invalidate-on-write**, and **update-on-write**.
- **Cache stampede** occurs when many requests regenerate the same expired data simultaneously.
- **Cache penetration** targets missing data and can be mitigated with negative caching or Bloom filters.
- **Cache breakdown** focuses on one expired hot key, while **hot keys** are heavily accessed entries that can overload cache servers.
- Designing a cache involves not just speed, but also freshness, resilience, and failure handling.

---
