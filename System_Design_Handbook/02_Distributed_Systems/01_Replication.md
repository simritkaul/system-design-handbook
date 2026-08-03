# Chapter 1 — Replication

> **Goal:** Understand why replication exists, what problems it solves, the different replication models, their tradeoffs, and when to use them.

---

# 1. The Problem

Imagine you've built an online shopping website.

Everything works perfectly.

Your architecture looks like this:

```text
                Users
                  │
                  ▼
            Application Server
                  │
                  ▼
            MySQL Database
```

Initially,

you have

```text
100 users
```

No issues.

---

A few months later.

```text
100,000 users
```

Still okay.

---

Then your app becomes popular.

```text
5 million users
```

Suddenly,

every page loads product information.

Every profile page loads user information.

Every order history page queries the database.

The database starts receiving

```text
50,000 read requests/sec
```

while only

```text
500 write requests/sec
```

Notice something.

The problem isn't writes.

The problem is **reads**.

---

# 2. The First Thought

You think

> Let's buy a bigger server.

```text
8 CPU

↓

16 CPU

↓

32 CPU

↓

64 CPU
```

This is called

**Vertical Scaling**.

It works.

But only for a while.

Eventually

- hardware becomes expensive
- there's a physical limit
- the server is still a single point of failure

We need another idea.

---

# 3. The Big Idea

Instead of having

one database,

have multiple copies.

```text
            Primary Database

           /      |      \

Replica   Replica   Replica
```

Every copy contains the same data.

Now users can read from different machines.

This is called

**Replication**.

---

# 4. What Is Replication?

Replication simply means

> Maintaining multiple copies of the same data.

Instead of storing

```text
Product

↓

One Machine
```

Store

```text
Product

↓

Machine A

Machine B

Machine C
```

If one machine dies,

another still has the data.

---

# 5. Why Do We Replicate?

Replication solves three major problems.

---

### A. Read Scalability

Instead of

```text
100,000 reads

↓

One Database
```

Use

```text
100,000 reads

↓

4 Replicas

↓

25,000 each
```

The workload is shared.

---

### B. High Availability

Suppose the primary server crashes.

Without replication

```text
Application

↓

Database ❌
```

System is down.

With replication

```text
Primary ❌

↓

Replica becomes Primary
```

The application continues working.

---

### C. Disaster Recovery

Suppose a disk fails.

A machine catches fire.

A data center goes offline.

Another replica still has the data.

---

# 6. Primary–Replica Architecture

This is the most common replication model.

```text
              Writes

                │

                ▼

          Primary Database

           /     |      \

          ▼      ▼       ▼

      Replica  Replica  Replica

          ▲      ▲       ▲

              Reads
```

Notice the rule.

Writes

↓

Primary only.

Reads

↓

Any replica.

---

# 7. Why Can't Clients Write Everywhere?

Imagine two users.

Replica A receives

```text
Balance

₹100

↓

₹50
```

Replica B receives

```text
Balance

₹100

↓

₹200
```

Now both databases disagree.

Which one is correct?

Conflict.

Having a single primary avoids this problem.

---

# 8. Replication Lag

Suppose Alice changes her profile picture.

The write reaches

Primary.

But the replicas haven't updated yet.

Now Bob views Alice's profile.

If Bob reads from a replica,

he might still see the old picture.

This delay is called

**Replication Lag**.

---

# 9. Strong vs Eventual Consistency

Suppose you update your password.

Should every server immediately know?

If yes,

that's **Strong Consistency**.

---

Suppose it takes

```text
200 ms
```

for replicas to update.

For a short time,

different replicas may return different answers.

Eventually,

they all agree.

That's **Eventual Consistency**.

We'll study consistency models in detail later.

For now,

remember that replication often introduces this tradeoff.

---

# 10. Synchronous vs Asynchronous Replication

There are two common approaches.

---

### Synchronous Replication

Primary waits until replicas acknowledge the write.

```text
Write

↓

Primary

↓

Replica A ✔

Replica B ✔

↓

Success
```

Advantages

- Strong consistency
- Minimal replication lag

Disadvantages

- Slower writes
- If a replica is slow, writes slow down

---

### Asynchronous Replication

Primary confirms the write immediately.

Replicas update later.

```text
Write

↓

Primary ✔

↓

Success

↓

Replica updates later
```

Advantages

- Fast writes
- Better performance

Disadvantages

- Replication lag
- A recent write could be lost if the primary fails before replicas receive it

---

# 11. Where Replication Shines

### E-commerce

Many users browse products.

Few users update products.

Read-heavy.

---

### News Websites

Millions read.

Few publish.

---

### Social Media

Reading feeds greatly outnumbers creating posts.

---

### Blogs

Thousands of readers.

One author.

---

# 12. Where Replication Doesn't Solve Everything

Suppose you're Instagram.

You now have

```text
50 million writes/sec
```

Every like,

comment,

follow,

message

is a write.

Replication helps reads.

It does **not** increase write capacity because the primary still handles all writes.

For that,

we'll need the next chapter:

**Sharding**.

---

# 13. Replication vs Backup

This confuses many beginners.

Replication

↓

Keeps another live copy.

Changes are copied almost immediately.

---

Backup

↓

A snapshot taken at a point in time.

Used for recovery.

A backup is **not** a replica.

A replica is **not** a backup.

---

# 14. Mental Model

Imagine a popular library.

Originally,

there is only one copy of a famous book.

Everyone waits in line.

The library buys

10 identical copies.

Now

10 people can read simultaneously.

That's replication.

The book hasn't changed.

Only the number of copies has.

---

# 15. Tradeoffs

### Advantages

- Improves read scalability
- High availability
- Disaster recovery
- Better fault tolerance
- Easier maintenance (one replica can be taken offline while others continue serving reads)

---

### Disadvantages

- Doesn't improve write capacity
- Replication lag
- More infrastructure to manage
- Storage cost increases
- Failover logic adds complexity

---

# 16. Real-World Examples

### Amazon

Product browsing.

---

### Netflix

Movie metadata.

---

### Banking

High availability for account information.

---

### News Websites

Large read traffic.

---

# 17. Common Interview Questions

- Why do we replicate databases?
- Does replication increase write throughput?
- What is replication lag?
- Why do we have a single primary?
- Synchronous vs asynchronous replication?
- Replica vs backup?
- When would replication alone not be enough?

---

# 18. Connections

Replication is the **first scaling technique**.

But eventually you'll encounter another problem.

Even with 100 replicas,

all writes still go to one primary.

When that primary becomes overloaded,

copying it won't help.

You need to **split the data itself** across multiple databases.

That's called **Sharding**, which is our next chapter.

---

# Key Takeaways

- Replication means maintaining **multiple copies of the same data**.
- It primarily improves **read scalability**, **availability**, and **fault tolerance**.
- The **Primary–Replica** model sends writes to one primary and distributes reads across replicas.
- Replication introduces tradeoffs such as **replication lag** and consistency challenges.
- It does **not** solve write bottlenecks—that requires data partitioning or sharding.

---
