# Chapter 4 — Key-Value Databases

> **Goal:** Understand why key-value databases exist, how they work, what problems they solve, why they're so fast, and why Redis, DynamoDB, etc. belong to this family.

---

# 1. The Problem

Imagine you're building Twitter.

A user opens the app.

You need to retrieve:

```text
User 48291
```

Nothing else.

You already know exactly which user you want.

Would you rather do this?

```text
SELECT *
FROM Users
WHERE UserID = 48291
```

Or simply

```text
48291 → User Object
```

That's the key insight.

Many applications don't need SQL's full power.

They just need

> "Give me the value associated with this key."

---

# 2. The Big Idea

Imagine a dictionary.

```text
Apple → Fruit

Dog → Animal

Car → Vehicle
```

You don't search every word.

You jump directly to "Apple."

A key-value database works exactly the same way.

```text
user:48291
↓

{
 Name,
 Email,
 Followers,
 Bio
}
```

The key identifies the data.

The value contains the data.

Nothing more.

Nothing less.

---

# 3. What is a Key?

The key must uniquely identify something.

Examples

```text
user:48291

order:9132

session:a8bc91

product:123

otp:987654
```

Keys are usually strings.

Think of them as addresses.

---

# 4. What is the Value?

The value can be almost anything.

Examples

String

```text
"John"
```

JSON

```json
{
  "name": "John",
  "age": 28
}
```

Image

Video

Binary

Numbers

Some databases support richer data structures than others.

---

# 5. Why Is It So Fast?

Imagine two libraries.

Library A

No organization.

Every time you want a book,

you walk through every shelf.

---

Library B

Every shelf is numbered.

You already know

Shelf 42

Book 17

You walk directly there.

Key-value databases optimize for this exact access pattern.

Most use highly optimized lookup structures (typically hash-based), allowing them to find a value with very little work compared to scanning records.

The result is extremely low-latency lookups.

---

# 6. The Tradeoff

Now imagine someone asks

> "Show every customer who spent more than ₹10,000 last month."

Can a simple key-value lookup answer that efficiently?

No.

Because the database isn't organized around arbitrary queries.

It's organized around **keys**.

You gain speed,

but lose flexibility.

---

# 7. Where Key-Value Databases Shine

## User Sessions

```text
session:abc123

↓

User Data
```

Perfect.

---

## Shopping Carts

```text
cart:user123

↓

Products
```

Simple lookup.

---

## User Profiles

```text
user:48291

↓

Profile
```

---

## Configuration

```text
feature:new-ui

↓

enabled
```

---

## Rate Limiting

```text
user123

↓

37 requests
```

---

## Counters

```text
video:123

↓

1,245,221 views
```

---

## Caching

```text
product:123

↓

Product Details
```

Exactly what a cache needs.

---

# 8. Where They Perform Poorly

Suppose your boss asks:

> Find all users from Delhi.

If your key is

```text
user:48291
```

there's no obvious way to answer that directly.

You'd need to inspect many values or maintain additional indexes.

This is where relational databases excel.

---

# 9. Key-Value vs SQL

| SQL               | Key-Value               |
| ----------------- | ----------------------- |
| Tables            | Keys                    |
| Rows              | Values                  |
| Relationships     | Independent records     |
| Rich queries      | Simple lookups          |
| Joins             | Usually none            |
| Flexible querying | Very fast direct access |

Notice

Neither is better.

They're solving different problems.

---

# 10. Horizontal Scaling

This is one of the biggest strengths.

Imagine

```text
100 million users
```

Instead of storing everything on one machine

```text
Machine A
```

we can distribute keys.

```text
A

user:1-25M

B

26-50M

C

51-75M

D

76-100M
```

Each machine owns a subset of keys.

Because records are mostly independent, this distribution is much simpler than distributing highly relational data.

This is one reason key-value databases scale so well.

---

# 11. Common Technologies

Now we finally meet the implementations.

### Redis

- In-memory
- Extremely fast
- Rich data structures
- Often used for caching, sessions, counters, leaderboards, locks, Pub/Sub, and more

---

### DynamoDB

- Persistent
- Fully managed
- Distributed
- Auto-scaling
- Designed for massive workloads

---

### Riak

An early distributed key-value database emphasizing availability.

---

### Amazon ElastiCache

A managed service that commonly runs Redis or Memcached, rather than being a separate database engine itself.

---

# 12. Mental Model

Imagine lockers in a gym.

Every locker has a number.

```text
Locker 27

↓

Bag
```

You don't ask

> Show me every blue towel.

You ask

> Open locker 27.

That's exactly how a key-value database works.

---

# 13. Common Interview Questions

- Why are key-value databases so fast?
- When should I use a key-value store?
- Why don't they support rich queries like SQL?
- How do they scale horizontally?
- Why are they commonly used for caching?
- What's the difference between Redis and DynamoDB?
- When would you choose a key-value store over a relational database?

---

# 14. Connections

This chapter explains the family.

The next chapters dive into specific members:

- **Redis** → An in-memory key-value database with powerful data structures and many real-time use cases.
- **DynamoDB** → A persistent, distributed key-value database built for massive scale.

Later we'll also compare them directly because although they share the same fundamental model, they're optimized for different goals.

---

# Key Takeaways

- A key-value database stores data as **key → value** pairs.
- It is optimized for **known-key lookups**, not arbitrary querying.
- Simplicity enables extremely fast reads and writes.
- Independent records make horizontal scaling much easier.
- Redis and DynamoDB belong to the same family, but they solve different operational problems.

---
