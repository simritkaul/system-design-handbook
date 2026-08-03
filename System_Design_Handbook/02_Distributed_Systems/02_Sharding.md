# Module 2 — Chapter 2 — Sharding

> **Goal:** Understand why sharding exists, what problems it solves, how it works, the different sharding strategies, and its tradeoffs.

---

# 1. The Problem

Suppose you're running Instagram.

Your database now receives

```text
10 million reads/sec
500,000 writes/sec
```

Earlier, we solved the read problem using replication.

```
                Primary
               /   |   \
          Replica Replica Replica
```

Now reads are distributed.

Great.

But...

Where do the writes go?

Still here.

```
Primary
```

The primary eventually becomes overloaded.

---

# 2. Why Replication Fails

Suppose

100,000 users like posts simultaneously.

Every write goes here.

```
Primary

↑
↑
↑
↑
↑
```

Adding more replicas doesn't help.

Because replicas usually don't accept writes.

Replication scales reads.

It doesn't scale writes.

---

# 3. The Big Idea

Instead of

copying the database,

split it.

Imagine one huge database.

```
Users

1

2

3

...

100 Million
```

Instead of storing everything together,

divide it.

```
Database A

Users

1-25M

----------------

Database B

26-50M

----------------

Database C

51-75M

----------------

Database D

76-100M
```

Now

writes are distributed.

This is called

**Sharding.**

---

# 4. What Is a Shard?

A shard is simply

a **piece of the data.**

Instead of

```
Entire Database
```

we have

```
Shard 1

Shard 2

Shard 3

Shard 4
```

Each shard stores only part of the dataset.

Together,

they behave like one logical database.

---

# 5. How Does the Application Know?

Suppose a user logs in.

```
User ID = 38,291
```

Which shard owns that user?

The application needs a rule.

That rule is called

the **Shard Key**.

---

# 6. What Is a Shard Key?

The shard key decides

where data lives.

Example

```
User ID

↓

Shard
```

Choosing a shard key is one of the most important decisions in system design.

A poor shard key can create bottlenecks.

A good one distributes load evenly.

---

# 7. Sharding Strategies

There isn't just one way to shard.

Let's look at the common strategies.

---

## A. Range-Based Sharding

Example

```
Users

1-1M

↓

Shard A

---------------

1M-2M

↓

Shard B

---------------

2M-3M

↓

Shard C
```

Very simple.

Advantages

- Easy to understand.
- Range queries are efficient.

Problems

Suppose new users always have increasing IDs.

Eventually,

every new write goes to

the last shard.

One shard becomes overloaded.

This is called a **hot shard**.

---

## B. Hash-Based Sharding

Instead of ranges,

calculate

```
hash(UserID)

↓

Shard
```

Now

User 1

may go to

Shard C.

User 2

may go to

Shard A.

User 3

may go to

Shard D.

The goal is to distribute data more evenly.

Advantages

- Better load balancing.

Problems

- Range queries become difficult because nearby IDs may live on different shards.

---

## C. Geographic Sharding

Sometimes,

data is divided by region.

Example

```
India

↓

Shard A

USA

↓

Shard B

Europe

↓

Shard C
```

Advantages

Lower latency.

Regional compliance.

Problems

Users moving between regions or interacting across regions can complicate the design.

---

# 8. Why Is Sharding Powerful?

Suppose one database handles

```
100,000 writes/sec
```

Now create

4 shards.

```
Shard A

100k

Shard B

100k

Shard C

100k

Shard D

100k
```

Now the system can potentially handle around

```
400,000 writes/sec
```

The work is distributed.

---

# 9. Cross-Shard Queries

Suppose your boss asks

```
Count all users.
```

Where are they?

Not on one machine.

They're on

```
Shard A

Shard B

Shard C

Shard D
```

Now every shard must participate.

Some queries become significantly more expensive after sharding.

---

# 10. Transactions Become Hard

Suppose

Alice

lives on

Shard A.

Bob

lives on

Shard C.

Alice transfers money to Bob.

One transaction now touches

two databases.

Distributed transactions are much more complicated than local ones.

---

# 11. Re-Sharding

Suppose

you started with

```
4 shards
```

Now you need

```
8 shards.
```

Can you simply add four servers?

No.

Existing data must be redistributed.

Moving large amounts of data while keeping the system online is a challenging operation.

This is one reason adding shards isn't trivial.

---

# 12. Where Sharding Shines

### Social Media

Billions of users.

---

### E-commerce

Millions of products.

---

### Messaging Systems

Huge write volumes.

---

### Gaming

Millions of player records.

---

### SaaS Platforms

Large customer bases.

---

# 13. Where Sharding Isn't Needed

Suppose your HR application has

```
10,000 employees.
```

One database is enough.

Sharding introduces complexity.

Don't do it unless you actually need it.

---

# 14. Sharding vs Replication

This is one of the most important comparisons.

| Replication          | Sharding                             |
| -------------------- | ------------------------------------ |
| Copies data          | Splits data                          |
| Improves reads       | Improves writes and storage capacity |
| Same data everywhere | Different data on each shard         |
| High availability    | Horizontal scalability               |

In practice,

many systems use both.

Each shard often has its own replicas.

---

# 15. Mental Model

Imagine a library.

Replication

↓

Buying

10 copies

of the same book.

More people can read simultaneously.

---

Sharding

↓

Splitting

the library

into

Science

History

Fiction

Mathematics

Each building stores different books.

Now no single building holds everything.

---

# 16. Tradeoffs

### Advantages

- Scales write throughput.
- Increases total storage capacity.
- Enables horizontal growth.
- Removes the single write bottleneck.

---

### Disadvantages

- Choosing a shard key is difficult.
- Cross-shard queries are expensive.
- Distributed transactions become complex.
- Re-sharding is operationally challenging.
- More application and operational complexity.

---

# 17. Real-World Examples

### Instagram

User data distributed across shards.

---

### Facebook

Massive user data partitioning.

---

### Discord

Users partitioned across infrastructure.

---

### Large SaaS Platforms

Customer data often partitioned by tenant or region.

---

# 18. Common Interview Questions

- Why do we shard databases?
- Replication vs sharding?
- What is a shard key?
- What is a hot shard?
- Hash vs range sharding?
- Why are distributed transactions difficult?
- What challenges arise when adding new shards?

---

# 19. Connections

Replication solved

```
Read Bottleneck
```

Sharding solved

```
Write Bottleneck
```

But...

There's still one problem.

Suppose your shard key is

```
User ID % 4
```

Now you add another shard.

Every mapping changes.

Almost every record needs to move.

That's incredibly expensive.

The industry needed a better way.

That solution is our next chapter:

> **Consistent Hashing**.

---

# Key Takeaways

- **Replication copies data** to improve reads and availability.
- **Sharding splits data** to improve write throughput and storage capacity.
- The **shard key** determines where data is stored and is one of the most critical design choices.
- Sharding introduces challenges such as cross-shard queries, distributed transactions, and re-sharding.
- Large-scale systems often combine **sharding and replication**, with each shard maintaining its own replicas.

---
