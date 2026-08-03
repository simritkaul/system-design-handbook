This chapter is much easier once you understand one fact:

> **A cache is always smaller than the data it could hold.**

That's why eviction exists.

You only have room for a small subset of the data.

The question is:

> **When the cache becomes full, what should we remove?**

That's what cache eviction policies answer.

---

# Module 3 — Chapter 3 — Cache Eviction Policies

> **Goal:** Understand why cache eviction is necessary, the major eviction algorithms, their tradeoffs, and when to use each one.

---

# 1. The Problem

Imagine your cache can store only

```text
100 products
```

Initially,

everything is fine.

```
Cache

Product 1

Product 2

...

Product 100
```

Then another request arrives.

```
Product 101
```

The cache is already full.

You can't keep everything.

Something has to leave.

But...

**Which item should be removed?**

---

# 2. The Big Idea

A cache eviction policy is simply:

> **A rule for deciding which cached item should be removed when the cache is full.**

Different policies make different assumptions about future access patterns.

---

# 3. Least Recently Used (LRU)

This is probably the most popular policy.

### The Idea

If something hasn't been used recently,

it's less likely to be needed again soon.

Remove it.

---

Example

Suppose your cache contains

```text
A

B

C

D
```

Access pattern

```
A

C

D
```

Notice

B hasn't been used in a while.

A new item arrives.

```
E
```

LRU removes

```
B
```

because it was used least recently.

---

## Why It Works

Human behavior often shows locality.

If someone visited

Amazon Product A

five seconds ago,

they're more likely to visit it again

than a product they viewed yesterday.

LRU takes advantage of this.

---

## Advantages

- Excellent for most web applications
- Adapts naturally to user behavior
- Widely supported

---

## Disadvantages

Imagine

a monthly report.

Millions of people access it

once.

LRU fills the cache with report data,

pushing out frequently used items.

This is called **cache pollution**.

---

# 4. Least Frequently Used (LFU)

Now we think differently.

Instead of asking

"When was it used?"

we ask

"How often is it used?"

---

Example

```
A

Used

1000 times

-----------

B

Used

2 times
```

Cache becomes full.

LFU removes

```
B
```

because it has been accessed much less often.

---

## Why It Works

Frequently accessed items

are likely to remain popular.

Example

Google logo

Amazon homepage

Popular products

Trending videos

---

## Advantages

- Great for stable popularity
- Keeps frequently requested items longer
- Useful for CDN-style workloads

---

## Disadvantages

Suppose

a product

was popular

last year.

It accumulated

10 million accesses.

Today

nobody uses it.

LFU still keeps it

because its historical count is huge.

Old popularity can become a problem.

---

# 5. FIFO (First In, First Out)

This one is simple.

Remove

the oldest cached item.

Regardless of

usage.

---

Example

```
A

↓

B

↓

C

↓

D
```

New item

```
E
```

FIFO removes

```
A
```

even if

A

was requested

one second ago.

---

## Advantages

- Extremely simple
- Easy implementation

---

## Disadvantages

Ignores actual access patterns.

Rarely the best choice.

---

# 6. Random Eviction

When full,

pick

a random item.

---

Surprisingly,

this works reasonably well

for some workloads.

It's extremely cheap.

---

## Advantages

- Simple
- Fast
- No bookkeeping

---

## Disadvantages

May remove

the hottest item

by chance.

Unpredictable.

---

# 7. TTL-Based Eviction

Sometimes

data naturally expires.

Example

```
OTP

↓

5 Minutes
```

After

5 minutes

the cache removes it.

Not because

the cache is full,

but because

the data

is no longer useful.

---

Examples

Sessions

Authentication Tokens

Temporary URLs

---

# 8. Which One Is Best?

There isn't one answer.

It depends

on the workload.

---

### Amazon

Products

User Profiles

LRU works well.

---

### YouTube

Trending Videos

LFU can be useful.

---

### OTP Service

TTL.

---

### Queue

FIFO.

---

# 9. LRU vs LFU

This is a favorite interview question.

Imagine

News Article A

```
Yesterday

1 million views
```

Today

Nobody reads it.

---

Article B

```
Published

5 minutes ago

Everyone is reading it.
```

Which algorithm adapts faster?

LRU.

Because

recent activity

matters more.

---

Now imagine

Google Logo.

It's requested

millions of times

every day.

LFU keeps it forever.

Perfect.

---

# 10. Modern Systems

Many production caches

don't use

pure

LRU

or

pure

LFU.

Instead,

they use

approximations

or

hybrid algorithms

that balance recency and frequency while remaining efficient.

The exact implementation depends on the system.

---

# 11. Mental Model

Imagine your wardrobe.

### LRU

Throw away

the clothes

you haven't worn recently.

---

### LFU

Throw away

the clothes

you almost never wear.

---

### FIFO

Throw away

the oldest clothes,

whether you still wear them or not.

---

### Random

Close your eyes.

Pick one.

---

### TTL

Throw away

expired food

from the refrigerator.

---

# 12. Tradeoffs

| Policy | Good For                  | Weakness                               |
| ------ | ------------------------- | -------------------------------------- |
| LRU    | Recent access patterns    | Doesn't recognize long-term popularity |
| LFU    | Frequently accessed items | Slow to adapt when popularity changes  |
| FIFO   | Simplicity                | Ignores usage                          |
| Random | Low overhead              | Unpredictable                          |
| TTL    | Naturally expiring data   | Doesn't account for popularity         |

---

# 13. Real-World Examples

### Redis

Supports configurable eviction policies, including LRU, LFU, random, and TTL-based approaches.

---

### Browser Cache

Often combines expiration times with recency heuristics.

---

### CDN

Frequently accessed content is often retained longer using policies influenced by recency and frequency.

---

### Session Storage

TTL-based expiration.

---

# 14. Common Interview Questions

- Why do caches need eviction?
- LRU vs LFU?
- Which policy is most common?
- When is LFU better than LRU?
- Why isn't FIFO ideal?
- Why do systems combine multiple strategies?
- What role does TTL play?

---

# 15. Connections

We've now answered

```
How do we remove data?
```

But another problem appears.

Suppose

the cache

contains

```
Product

↓

₹999
```

The database updates

the price

to

```
₹899
```

The cache

still returns

```
₹999
```

Now users receive

incorrect information.

This is one of the hardest problems in distributed systems.

It's called

> **Cache Invalidation.**

That will be our next chapter.

---

# Key Takeaways

- Cache eviction is necessary because cache memory is limited.
- **LRU** assumes recently accessed data is more likely to be accessed again.
- **LFU** assumes frequently accessed data will remain popular.
- **FIFO** removes the oldest entry regardless of usage.
- **Random** trades optimality for simplicity.
- **TTL** removes data based on expiration rather than popularity.
- Modern cache systems often use hybrid or approximate policies rather than pure textbook algorithms.

---

## One thing I'd tweak in the handbook

I'd make a small distinction between **eviction** and **expiration**, because they're easy to confuse:

- **Eviction** happens because the cache **needs space**.
- **Expiration (TTL)** happens because the data has **become too old**, even if the cache still has plenty of free space.

Some systems can use TTL as part of their eviction strategy, but conceptually they're solving different problems. Keeping that distinction clear will help later when we discuss Redis configuration and cache invalidation.
