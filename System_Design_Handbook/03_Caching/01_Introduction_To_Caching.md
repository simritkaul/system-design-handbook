I think this is going to be one of the most important chapters in the handbook.

If someone asked me:

> **"What one concept appears in almost every HLD interview?"**

I'd probably say **Caching**.

Almost every large-scale system eventually asks the same question:

> **"Why are we doing the same expensive work over and over again?"**

Caching is the answer.

---

# Module 3 — Caching

---

# Chapter 1 — Introduction to Caching

> **Goal:** Understand what caching is, why it exists, how it works conceptually, and why it is one of the most powerful techniques for improving system performance.

---

# 1. The Problem

Imagine you're building Amazon.

A customer opens the product page for

```text
iPhone 17 Pro
```

Your application does

```text
Application

↓

Database

↓

Product Information

↓

Return Response
```

Everything works.

---

One second later,

another customer opens

the exact same product.

Your application again does

```text
Application

↓

Database

↓

Product Information

↓

Return Response
```

Then another.

Then another.

Then another.

Eventually

100,000 users

request

the same product.

Your database keeps doing

the same work

100,000 times.

---

# 2. What's Wrong Here?

Notice something.

The product information hasn't changed.

We're repeatedly asking the database

for identical data.

The database isn't overloaded because the queries are difficult.

It's overloaded because we're asking

the same question

again

and again.

---

# 3. The Big Idea

Instead of asking the database every time,

save the answer somewhere much faster.

```text
User

↓

Application

↓

Cache

↓

Found?

↓

Yes

↓

Return Immediately

↓

No

↓

Database

↓

Store in Cache

↓

Return
```

The next request never reaches the database.

---

# 4. What Is a Cache?

A cache is simply

> **A temporary storage layer that keeps frequently accessed data closer to the application so future requests can be served faster.**

Notice something.

A cache is

not

the source of truth.

The database still owns the data.

The cache merely stores

copies.

---

# 5. Why Does Caching Work?

Think about YouTube.

How many people watch

the newest Marvel trailer?

Millions.

Should YouTube ask

its primary database

for the same video metadata

millions of times?

No.

Store the frequently accessed information

in the cache.

Now the database only handles requests

when necessary.

---

# 6. Memory Hierarchy

Why is a cache fast?

Because data is kept in faster storage.

Think of a computer.

```text
CPU Registers

↓

CPU Cache

↓

RAM

↓

SSD

↓

HDD
```

The closer data is to the processor,

the faster it is.

Applications use the same philosophy.

Instead of repeatedly reading from slower storage,

keep frequently accessed data

in memory.

---

# 7. Cache Hit

Suppose

the requested data

already exists in the cache.

```text
User

↓

Application

↓

Cache ✔

↓

Response
```

The database is never contacted.

This is called

a

**Cache Hit.**

---

# 8. Cache Miss

Suppose

the requested data

isn't in the cache.

```text
User

↓

Application

↓

Cache ❌

↓

Database

↓

Store in Cache

↓

Return
```

This is called

a

**Cache Miss.**

---

# 9. Cache Hit Ratio

Suppose

100 requests arrive.

```text
90

↓

Cache Hit
```

```text
10

↓

Cache Miss
```

Your

Cache Hit Ratio

is

```text
90%
```

Generally,

higher hit ratios mean

better performance,

lower database load,

and lower latency.

---

# 10. Why Not Cache Everything?

Good question.

Memory is expensive.

A database might store

```text
100 TB
```

Your cache might only have

```text
200 GB
```

So the cache stores

the most valuable

or

most frequently accessed

data.

---

# 11. What Makes Good Cache Data?

Usually

data that is

- Read frequently
- Changes infrequently
- Expensive to compute
- Expensive to fetch
- Shared across many users

Examples

- Product details
- User profiles
- Trending videos
- Popular tweets
- Leaderboards
- Configuration

---

# 12. What Shouldn't Be Cached?

Data that

- Changes every millisecond
- Must always be perfectly accurate
- Is rarely accessed
- Is extremely user-specific and short-lived (unless the use case justifies it)

Examples

- Bank account balances (often)
- Live stock trading prices (depending on latency requirements)
- Rapidly changing counters

The key idea is:

If stale data causes serious problems,

be cautious.

---

# 13. Time To Live (TTL)

Suppose

we cache

```text
Product Price

↓

₹999
```

Tomorrow

the seller changes it

to

```text
₹899
```

If we never remove

the cached copy,

users keep seeing

the old price.

Caches often assign

a

**TTL (Time To Live)**

Example

```text
Cache

↓

10 Minutes

↓

Automatically Remove
```

After expiration,

the next request fetches

fresh data.

---

# 14. Benefits of Caching

### Faster Response Time

Reading from memory

is much faster than querying a database.

---

### Lower Database Load

Thousands of identical requests

become

one database query

followed by many cache hits.

---

### Better Scalability

The database handles fewer requests,

allowing the application to serve more users.

---

### Lower Cost

Reducing expensive database operations

can lower infrastructure costs.

---

# 15. Where Caching Is Used

Almost everywhere.

### Amazon

Product pages.

---

### Netflix

Movie metadata.

---

### YouTube

Video information.

---

### Twitter

User timelines.

---

### Instagram

Feeds.

---

### Google Maps

Frequently accessed map tiles.

---

# 16. Where Caching Doesn't Help

Suppose

every request

is completely unique.

Example

```text
Random Number Generator
```

or

highly personalized,

never-repeated computations.

The chance of a future cache hit is low.

Caching provides little benefit.

---

# 17. Mental Model

Imagine a chef.

The pantry

contains every ingredient.

Walking there

takes time.

Instead,

the chef keeps

salt,

pepper,

oil,

and spices

on the countertop.

Frequently used ingredients

stay close.

Rare ingredients

remain in storage.

The countertop

is the cache.

The pantry

is the database.

---

# 18. Tradeoffs

### Advantages

- Lower latency
- Reduced database load
- Better scalability
- Lower infrastructure costs
- Improved user experience

---

### Disadvantages

- Extra complexity
- Data can become stale
- Memory is expensive
- Cache management is difficult
- Choosing what to cache is an architectural decision

---

# 19. Common Interview Questions

- What is caching?
- Why is caching effective?
- What is a cache hit?
- What is a cache miss?
- What is cache hit ratio?
- Why don't we cache everything?
- What is TTL?
- What kinds of data are good candidates for caching?

---

# 20. Connections

Now we've answered

**why**

caches exist.

The next question is

> **"How should my application actually use the cache?"**

There isn't just one answer.

There are several patterns:

- Cache Aside
- Read Through
- Write Through
- Write Around
- Write Back

Each solves a different problem.

Those patterns form the heart of caching architecture and are among the most frequently discussed topics in High-Level Design interviews.

---

# Key Takeaways

- A cache stores **temporary copies of frequently accessed data** to reduce latency and backend load.
- The database remains the **source of truth**; the cache stores copies.
- A **cache hit** serves data directly from the cache, while a **cache miss** fetches data from the database and typically populates the cache.
- Good cache candidates are frequently read, infrequently changing, and expensive to compute or retrieve.
- **TTL** helps prevent stale data from remaining in the cache indefinitely.

---

## One thing I'd add to the handbook

From this chapter onward, I think every architecture-related concept should include a **"Before vs After"** section.

For caching, for example:

### Before Caching

```text
Every Request
      │
      ▼
Application
      │
      ▼
Database
```

Every read hits the database.

---

### After Caching

```text
Every Request
      │
      ▼
Application
      │
      ▼
Cache
   │       │
 Hit     Miss
   │       │
   ▼       ▼
Response Database
             │
             ▼
       Populate Cache
```

One diagram immediately shows _why_ the architecture changed. I think adding this "evolution" view to future chapters (load balancers, CDNs, message queues, etc.) will make the handbook much easier to internalize because you'll always see the problem first and then the architectural improvement.
