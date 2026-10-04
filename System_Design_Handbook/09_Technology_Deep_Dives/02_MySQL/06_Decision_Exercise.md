Good. Let's close out MySQL properly with the scenario rather than adding more theory.

## MySQL Decision Exercise — Walkthrough

The important thing here isn't that the answer is **MySQL**. It's _why_.

### 1. What characteristics matter?

From the requirements:

```text
Orders
  ├── Users
  ├── Products
  ├── Payments
  └── Shipping

Strong correctness
        +
Relationships
        +
Transactions
        +
Query by user/date/status
        +
20× more reads than writes
```

The strongest signals are:

- **Relational data**
- **Transactions**
- **Strong correctness**
- **Multiple related entities**
- **Range/filter queries**
- **Read-heavy workload**

That immediately makes a relational database a strong candidate.

---

# 2. Why MySQL fits

The order workflow could be represented as:

```text
BEGIN

Create Order
Create Order Items
Update payment/order state
Reserve inventory

COMMIT
```

If something goes wrong:

```text
ROLLBACK
```

We also have natural relationships:

```text
User
 │
 └── Orders
       │
       └── Order Items
              │
              └── Products
```

And queries such as:

```sql
SELECT *
FROM orders
WHERE user_id = ?
  AND created_at >= ?
ORDER BY created_at DESC;
```

are exactly the sort of workload relational databases handle well.

---

# 3. Why not Cassandra?

Cassandra is excellent when we have:

```text
Huge distributed scale
+
Predictable access patterns
+
High write throughput
+
Availability across many nodes
```

But our order system has an important requirement:

> **Strong transactional correctness across related business operations.**

We don't want to make order creation logic unnecessarily difficult just because we selected a database optimized around a different model.

So Cassandra isn't impossible.

It's simply not the most natural first choice.

---

# 4. Why not DynamoDB?

DynamoDB could handle:

```text
Massive scale
High availability
Predictable key-based access
```

But again, we need to ask:

> What are our access patterns?

We have:

```text
Orders by user
Orders by date
Orders by status
Relationships between entities
Transactional order creation
```

DynamoDB can model many of these requirements, but you have to design your access patterns and keys very deliberately.

A relational database gives us a more natural model for this domain.

---

# 5. Why not MongoDB?

MongoDB becomes attractive when the data naturally looks like:

```text
Order
{
    user: {...},
    items: [...],
    shipping: {...}
}
```

and retrieving the aggregate as a document is the dominant operation.

But our domain has multiple strongly related entities and transactional business logic.

Again, MongoDB can absolutely support transactions.

The point isn't:

> "MongoDB can't do this."

It's:

> **MySQL gives us these requirements naturally without forcing the application to work around the database model.**

That's the distinction you should make in interviews.

---

# 6. Now the Interesting Part: 20× More Reads

Suppose:

```text
Writes = 5,000/sec
Reads  = 100,000/sec
```

We don't immediately shard.

First:

```text
                Application
                    │
             ┌──────┴──────┐
             │             │
           Writes         Reads
             │             │
             ▼             ▼
          Primary      Read Replicas
                           │
                    ┌──────┼──────┐
                    ▼      ▼      ▼
                   R1     R2     R3
```

Now the primary concentrates on writes while replicas absorb read traffic.

---

# 7. Add Indexes

Order history is queried by:

```text
user_id
created_at
status
```

So we'd investigate indexes appropriate to the actual query patterns.

For example, a query like:

```sql
WHERE user_id = ?
ORDER BY created_at DESC
```

could benefit from an index designed around:

```text
(user_id, created_at)
```

This is important:

> **We don't add indexes because "indexes are good." We add them because they support actual access patterns.**

And remember the tradeoff:

```text
More indexes
     ↓
Faster reads
     ↓
But
     ↓
More storage
+
More work during writes
```

---

# 8. What About Caching?

Suppose users repeatedly open:

```text
GET /orders/{orderId}
```

We could use Redis:

```text
                Application
                    │
                    ▼
                  Redis
                 /     \
             HIT       MISS
              │          │
              ▼          ▼
           Response     MySQL
```

But we shouldn't blindly cache everything.

Order state is business-critical.

If:

```text
Order = PAID
```

and then:

```text
Order = REFUNDED
```

we need to think carefully about invalidation and consistency.

This is where your Module 3 knowledge becomes useful.

---

# 9. What If One Primary Can't Handle the Writes?

Suppose eventually:

```text
100k writes/sec
```

and we've already optimized:

```text
Queries
Indexes
Application
Caching
Database hardware
```

Replication won't solve the fundamental write bottleneck:

```text
               Primary
                  ↑
        all writes converge here
```

We may then consider sharding:

```text
                    Application
                         │
                   Shard Router
                /        |        \
               ▼         ▼         ▼
           MySQL A    MySQL B    MySQL C
```

Perhaps by:

```text
user_id
```

so:

```text
hash(user_id)
      ↓
Shard
```

Then:

```text
User A → Shard 1
User B → Shard 3
User C → Shard 2
```

Now write responsibility is distributed.

---

# 10. But What Did We Lose?

This is the important part.

Before sharding:

```text
One database
     ↓
Easy transactions
Easy joins
Easy queries
```

After:

```text
Multiple databases
     ↓
Cross-shard joins become difficult
Cross-shard transactions become difficult
Routing becomes necessary
Rebalancing becomes difficult
Hot shards become possible
```

So our architecture became:

```text
More scale
     ↑
     │
     │
More complexity
```

That's the tradeoff.

---

# 11. The Final Architecture

At a reasonably large scale, we could arrive at something like:

```text
                         Clients
                            │
                            ▼
                       Application
                            │
              ┌─────────────┴─────────────┐
              │                           │
            Reads                       Writes
              │                           │
              ▼                           ▼
         Redis Cache                   MySQL
              │                       Primary
              │                           │
              │                    ┌──────┴──────┐
              │                    ▼             ▼
              │                Replica        Replica
              │
              └──── Cache Miss ────────────────►
```

And if write/data scale eventually demands it:

```text
                       Application
                            │
                       Shard Router
                    /       |       \
                   ▼        ▼        ▼
               Shard A   Shard B   Shard C
                  │         │         │
                P + R     P + R     P + R
```

That's a **MySQL-based distributed architecture**.

Notice how we didn't arrive here by memorizing "MySQL architecture."

We arrived here by following:

```text
Requirement
    ↓
Workload
    ↓
Database model
    ↓
MySQL
    ↓
Scaling constraint
    ↓
Replication
    ↓
Caching
    ↓
Sharding
```

That is exactly the reasoning skill Module 9 is supposed to build.

---

# MySQL: Final Mental Model

If you remember only one thing:

> **MySQL is a strong choice when the system's core data is structured, relational, and transaction-heavy, and correctness of multi-step business operations matters.**

Then think:

```text
MySQL
 │
 ├── Relationships
 │      ↓
 │   SQL + JOINs
 │
 ├── Correctness
 │      ↓
 │   Transactions
 │   Isolation
 │   Locking
 │   MVCC
 │
 ├── Read scale
 │      ↓
 │   Replicas
 │   Cache
 │
 ├── Availability
 │      ↓
 │   Replication + Failover
 │
 └── Extreme scale
        ↓
     Sharding
        ↓
     Complexity
```

And the key tradeoff:

> **MySQL gives you powerful relational and transactional guarantees, but preserving those guarantees while horizontally scaling introduces coordination and architectural complexity.**

That is enough MySQL knowledge for our **SDE II HLD objective**. We don't need to go into optimizer internals, storage-page layouts, every isolation edge case, or DBA configuration.

---

# Chapter 2 — PostgreSQL

Now the natural next question is:

> **If MySQL and PostgreSQL solve almost the same broad problem, why does PostgreSQL exist as a separate technology choice?**

We'll approach PostgreSQL differently from MySQL.

We **won't repeat**:

```text
What is a relational database?
What is an index?
What is a transaction?
What is replication?
What is sharding?
```

You've already learned those concepts.

Instead, we'll ask:

> **What does PostgreSQL bring to the relational-database problem that makes someone deliberately choose it over MySQL?**

That will make the MySQL → PostgreSQL transition useful rather than repetitive.
