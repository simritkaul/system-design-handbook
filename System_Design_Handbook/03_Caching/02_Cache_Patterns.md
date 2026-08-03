# Module 3 — Chapter 2 — Cache Patterns

> **Goal:** Understand the major caching strategies, how reads and writes flow through the system, and the tradeoffs of each pattern.

---

# 1. The Problem

We now have

```text
Application

↓

Cache

↓

Database
```

Great.

But...

Suppose a user asks

```text
Product 123
```

What happens?

Should the application

look in the cache first?

Or the database?

When should the cache be updated?

Who updates it?

Immediately?

Later?

These questions define the caching pattern.

---

# 2. The Five Major Patterns

There are five patterns you'll encounter most often.

```text
Cache Aside

Read Through

Write Through

Write Around

Write Back
```

Each optimizes a different problem.

Let's study them one by one.

---

# 3. Cache Aside (Lazy Loading)

This is the most common pattern.

Probably used in

80%

of HLD interviews.

---

## The Idea

The application manages the cache.

The cache knows nothing about the database.

---

### Read Flow

Suppose a user asks for

```text
Product 123
```

Application

↓

Cache

↓

Miss

↓

Database

↓

Store in Cache

↓

Return

---

Next request

Application

↓

Cache

↓

Hit

↓

Return

Database isn't touched.

---

### Write Flow

Suppose the product price changes.

Application

↓

Database

↓

Update Cache

(or invalidate it)

The application is responsible for keeping the cache correct.

---

## Advantages

- Simple
- Easy to implement
- Cache stores only frequently accessed data
- Very popular

---

## Disadvantages

- First request is always slow (cache miss)
- Developers must handle cache invalidation correctly
- Stale data is possible if cache isn't updated properly

---

## Best For

Almost every web application.

Twitter.

Instagram.

Netflix.

Amazon.

---

# 4. Read Through

Now,

the application doesn't talk

to the database.

Instead

Application

↓

Cache

↓

Database

The cache becomes smarter.

---

### Read Flow

Application

↓

Cache

↓

Miss

↓

Cache fetches Database

↓

Stores Result

↓

Returns Data

The application only knows about

the cache.

---

### Advantages

- Cleaner application code
- Cache automatically populates itself
- Centralized caching logic

---

### Disadvantages

- More complex cache layer
- Tightly coupled with the backing data store
- Less common unless your caching infrastructure supports it

---

## Best For

Enterprise platforms.

Managed cache services.

---

# 5. Write Through

Suppose

a user changes

their profile.

Instead of

Application

↓

Database

↓

Cache

We do

Application

↓

Cache

↓

Database

The cache writes to

the database

immediately.

---

### Read

Reads always hit the cache.

---

### Write

Application

↓

Cache

↓

Database

↓

Success

Only after both succeed.

---

## Advantages

- Cache always stays fresh
- Reads almost always hit the cache
- No stale data caused by forgetting to update the cache

---

## Disadvantages

- Every write is slower because both cache and database are updated
- Even rarely read data ends up in the cache, which may waste memory

---

## Best For

Frequently read data

that must remain synchronized.

---

# 6. Write Around

Suppose

writes

completely skip the cache.

Application

↓

Database

Later

if someone reads

Application

↓

Cache

↓

Miss

↓

Database

↓

Cache

Only reads populate the cache.

---

## Advantages

- Cache doesn't fill with data that nobody reads
- Better cache utilization

---

## Disadvantages

- Immediately after a write,
  the next read is usually a cache miss
- Slightly higher read latency after updates

---

## Best For

Write-heavy systems

where only some data becomes popular.

---

# 7. Write Back (Write Behind)

This one is interesting.

Application

↓

Cache

↓

Success

The user gets a response immediately.

The database

is updated

later.

---

### Flow

Application

↓

Cache

↓

Immediate Response

↓

Background Worker

↓

Database

---

## Advantages

- Extremely fast writes
- Database handles fewer immediate write operations
- Can batch multiple writes together

---

## Disadvantages

- Risk of data loss if the cache fails before flushing to the database
- More complex implementation
- Eventual consistency

---

## Best For

Analytics

Logging

Counters

Telemetry

Situations where losing a few recent writes is acceptable.

---

# 8. Visual Comparison

| Pattern       | Reads                  | Writes                          | Most Common Use          |
| ------------- | ---------------------- | ------------------------------- | ------------------------ |
| Cache Aside   | Cache → DB on miss     | DB then cache update/invalidate | General web apps         |
| Read Through  | Cache handles misses   | Depends on implementation       | Managed cache systems    |
| Write Through | Cache                  | Cache then DB                   | Strong cache consistency |
| Write Around  | Cache after first read | DB only                         | Write-heavy workloads    |
| Write Back    | Cache                  | Cache immediately, DB later     | High write throughput    |

---

# 9. Which Pattern Should I Choose?

Let's think about the problem.

### Amazon Product Page

Millions of reads.

Few updates.

Cache Aside.

---

### Banking

Need consistent reads after writes.

Write Through.

---

### Logging System

Millions of writes.

Can tolerate delayed persistence.

Write Back.

---

### News Website

Articles change occasionally.

Huge read traffic.

Cache Aside.

---

### Analytics Pipeline

Continuous writes.

Database shouldn't become the bottleneck.

Write Back.

---

# 10. Mental Model

Imagine a school library.

### Cache Aside

The librarian checks the nearby shelf.

If the book isn't there,

they fetch it from storage

and place it on the shelf.

---

### Read Through

Students always ask the librarian.

The librarian decides

where to fetch the book from.

---

### Write Through

Every new book

goes to

the shelf

and

the archive

before it's considered available.

---

### Write Around

New books go directly

to the archive.

Only popular books

make it to

the shelf.

---

### Write Back

The librarian places

the returned books

on the nearby shelf first.

Later,

someone organizes them

back into the archive.

---

# 11. Tradeoffs

There is no universally best pattern.

The choice depends on:

- Read vs write ratio
- Freshness requirements
- Memory constraints
- Latency goals
- Risk tolerance

System design is about selecting the pattern whose tradeoffs match your application's needs.

---

# 12. Common Interview Questions

- What is Cache Aside?
- Cache Aside vs Read Through?
- Write Through vs Write Back?
- Why would you use Write Around?
- Which caching strategy would you choose for Twitter? For a bank?
- Which strategy provides the fastest writes?
- Which strategy minimizes stale data?

---

# 13. Connections

Now we know **how data gets into and out of the cache**.

The next question is:

> **"What happens when the cache becomes full?"**

We obviously can't keep every item forever.

Some data must be removed.

But...

**Which data?**

That leads us to **Cache Eviction Policies**:

- LRU
- LFU
- FIFO
- Random
- TTL-based eviction

---

# Key Takeaways

- A **caching pattern** defines how the application, cache, and database interact.
- **Cache Aside** is the most common and usually the default choice for web applications.
- **Read Through** hides database access behind the cache.
- **Write Through** prioritizes cache freshness by updating cache and database together.
- **Write Around** avoids filling the cache with rarely read data.
- **Write Back** prioritizes write performance by delaying database updates.

---

## One suggestion for improving the handbook

I think every pattern chapter should end with a **"Default Recommendation"** because interviewers often ask, "What would you use?"

Mine would be:

- **No special constraints?** → **Cache Aside**
- **Need very fresh cached reads?** → **Write Through**
- **Write-heavy analytics or logging?** → **Write Back**
- **Many writes but only some data is ever read?** → **Write Around**
- **Infrastructure manages the cache for you?** → **Read Through**

That gives you a practical decision tree instead of just five definitions to memorize.
