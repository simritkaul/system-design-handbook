# Module 9 — Chapter 5: DynamoDB

We've now reached an interesting point in the database journey.

So far:

```text
MySQL
   → relational model

PostgreSQL
   → relational model + richer capabilities

MongoDB
   → document model + flexible aggregates

Cassandra
   → distributed, partition-oriented database
```

DynamoDB enters with a deceptively simple question:

> **What if I want a massively scalable distributed database, but I don't want my engineering team managing the database cluster itself?**

That's the problem we need to understand first.

---

# 1. The Problem

Imagine you're building a global application on AWS.

You need:

```text
Millions of requests/sec
Billions of records
High availability
Automatic scaling
Low predictable latency
Multiple geographic regions
```

You could operate a distributed database yourself.

But now your team has to worry about:

```text
Nodes
Replication
Partitioning
Capacity
Scaling
Failures
Replacement
Rebalancing
Patching
Monitoring
```

That's a lot of operational responsibility.

So there are really two problems:

```text
Problem 1:
How do we build a massively distributed database?

Problem 2:
Who operates it?
```

DynamoDB's answer is essentially:

> **AWS operates the distributed database infrastructure for you, while you interact with a managed key-value/document database.**

That operational model is a huge part of why DynamoDB exists.

---

# 2. The Big Idea

> **DynamoDB is a fully managed, distributed NoSQL database designed for predictable, low-latency access at very large scale, where the application defines its access patterns through keys and indexes.**

There are several important words here:

```text
Fully managed
Distributed
Key-value/document
Predictable access
Massive scale
Low latency
```

The last two chapters now connect nicely:

```text
Cassandra
    ↓
Distributed database
    ↓
Partitioning + replication
    ↓
High availability + scale

DynamoDB
    ↓
Similar distributed goals
    +
Managed infrastructure
    +
AWS-specific model
```

But don't think:

> "DynamoDB = hosted Cassandra."

It isn't.

---

# 3. The Fundamental Model

At the simplest level:

```text
DynamoDB
   │
   └── Table
         │
         ├── Item
         ├── Item
         └── Item
```

An item is conceptually similar to a document:

```text
{
    userId: "42",
    name: "Simrit",
    age: 27,
    preferences: {
        theme: "dark"
    }
}
```

But DynamoDB's mental model is more strongly centered around:

> **How do I identify and retrieve this item efficiently?**

That makes keys extremely important.

---

# 4. The Most Important DynamoDB Concept: Keys

Suppose we have:

```text
Users
```

and want:

```text
Get user 42
```

We can have:

```text
Partition key:
userId
```

Conceptually:

```text
userId = 42
      ↓
partitioning
      ↓
appropriate storage location
      ↓
User 42
```

This should feel familiar from Cassandra.

The database needs a way to distribute data across its infrastructure.

The key participates in that distribution.

---

# 5. Partition Key

Imagine:

```text
Orders
------------------
orderId
customerId
status
total
createdAt
```

We might choose:

```text
partition key = orderId
```

Then:

```text
Order 1001
Order 1002
Order 1003
...
```

can be distributed across the underlying infrastructure.

The important thing is that DynamoDB handles the physical infrastructure.

You don't manually decide:

```text
Node A
Node B
Node C
```

That's part of the managed service.

---

# 6. Partition Key vs Sort Key

DynamoDB also supports a **composite primary key**:

```text
Partition key
+
Sort key
```

For example:

```text
customerId
orderDate
```

Imagine:

```text
customerId = 42
```

Then we could have:

```text
42 | 2026-10-01
42 | 2026-10-03
42 | 2026-10-05
42 | 2026-10-08
```

Conceptually:

```text
Partition:
customerId = 42

       Sort key
           ↓
      ┌──────────────┐
      │ Oct 1        │
      │ Oct 3        │
      │ Oct 5        │
      │ Oct 8        │
      └──────────────┘
```

This is extremely useful for queries like:

> "Give me this customer's orders between these dates."

---

# 7. The Cassandra Connection

Compare this with what we just learned:

### Cassandra

```text
Partition key
+
Clustering columns
```

### DynamoDB

```text
Partition key
+
Sort key
```

The terminology differs, but the architectural idea is familiar:

```text
Partition key
    ↓
Where the data belongs

Secondary ordering/key
    ↓
How related items are organized/queryable
```

This is why your Module 2 and Cassandra knowledge matters here.

---

# 8. Query-Driven Modeling

DynamoDB strongly reinforces the principle:

> **Design your data model around how the application will access the data.**

Suppose your application needs:

```text
Get all orders for customer
Get customer's recent orders
Get order by orderId
```

We design keys and indexes to support those access patterns.

But suppose six months later the product team asks:

> "Give me all orders across the entire system where status = DELIVERED and total > ₹10,000, sorted by customer region."

If we didn't design for that access pattern, DynamoDB isn't going to magically behave like PostgreSQL.

That's the tradeoff.

---

# 9. This Is the Key Difference From SQL

With MySQL, you can often say:

> "I didn't think of this query when I designed the schema."

You can potentially add an index later.

And SQL gives you substantial flexibility in querying relationships.

With DynamoDB, your thinking should be more like:

```text
Required query
      ↓
What key/index supports it?
      ↓
Design table accordingly
```

So DynamoDB rewards:

> **Predictable access patterns.**

---

# 10. Secondary Indexes

But what if you need another access pattern?

Suppose your primary key is:

```text
customerId
```

but you also need:

```text
Find orders by status
```

DynamoDB provides secondary indexes.

Conceptually:

```text
Main table
--------------------
customerId
orderDate
status
total


Index
--------------------
status
orderDate
```

Now the same underlying data can support another access pattern.

Think:

```text
Primary key
     ↓
Primary access pattern

Secondary index
     ↓
Additional access pattern
```

This is extremely useful.

But again:

> **Indexes aren't free.**

They consume storage and add work when underlying data changes.

---

# 11. Why DynamoDB Can Scale So Far

The important architectural idea is:

```text
Application
     │
     ▼
DynamoDB
     │
     ├── partitioning
     ├── replication
     ├── distributed infrastructure
     └── automatic capacity management
```

You aren't manually running:

```text
Node 1
Node 2
Node 3
...
Node 100
```

AWS manages the underlying infrastructure.

This dramatically reduces the operational burden.

So the tradeoff is:

```text
Self-managed distributed database
        ↓
More control
        +
More operational responsibility

DynamoDB
        ↓
Less operational responsibility
        +
More platform-specific constraints
```

---

# 12. Availability

DynamoDB is designed for high availability through distributed infrastructure and replication.

Conceptually:

```text
                 DynamoDB
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
       Replica   Replica   Replica
```

If underlying infrastructure fails, the service handles the recovery.

From the application's perspective, you don't normally need to perform the equivalent of:

```text
Replace failed Cassandra node
Rebalance cluster
Repair replicas
```

AWS handles that layer.

This is one of the biggest differences between learning Cassandra and using DynamoDB.

---

# 13. Consistency

Now things get interesting.

DynamoDB offers different read consistency choices.

At a high level:

```text
Eventually consistent read
```

versus:

```text
Strongly consistent read
```

An eventually consistent read may temporarily return an older value after a write.

A strongly consistent read provides a stronger guarantee that you're seeing the latest committed value within the supported scope.

So again:

```text
Consistency
     ↕
Latency / throughput / cost
```

becomes an architectural decision.

---

# 14. Don't Confuse "Eventually Consistent" With "Incorrect"

Suppose:

```text
Write:
balance = 100
```

Immediately afterward, another read could potentially see:

```text
balance = 90
```

under eventual consistency if replication hasn't converged.

That doesn't mean:

> "The database lost the write."

It means:

> **Different replicas may temporarily expose different versions.**

Eventually they converge.

This is the same distributed-systems principle you already learned.

---

# 15. Transactions

DynamoDB does support transactions.

You can perform coordinated operations across multiple items.

But here's the important architectural question:

> **Should your data model require transactions across dozens of unrelated items all the time?**

If yes, you should reconsider the data model.

As with Cassandra and MongoDB, the existence of transactions doesn't mean:

> "It is equivalent to a relational database."

The underlying design philosophy remains optimized for distributed key-based access.

---

# 16. DynamoDB's Scaling Trap: Hot Partitions

Now we hit one of the most important HLD issues.

Suppose:

```text
partition key = celebrityId
```

and one celebrity has:

```text
100 million requests/sec
```

while everyone else has:

```text
100 requests/sec
```

You have:

```text
                DynamoDB
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
      Normal      Normal     HOT
                            partition
```

The service can automatically scale the overall system.

But:

> **Scaling the total database doesn't magically make a poorly distributed access pattern healthy.**

This is the same lesson as Cassandra.

---

# 17. Write Sharding / Key Sharding

One solution can be to introduce controlled randomness or buckets into the key.

Instead of:

```text
celebrityId = 42
```

you might conceptually use:

```text
celebrityId = 42#0
celebrityId = 42#1
celebrityId = 42#2
...
```

Then requests can distribute across multiple partitions.

But now reading all data for celebrity 42 may require querying multiple buckets.

So:

```text
Avoid hot partition
       ↓
More distributed writes
       ↓
But
       ↓
More complicated reads
```

Again:

> **There is no free scalability.**

---

# 18. DynamoDB's Pricing Model Matters Architecturally

This is something you should understand for HLD interviews.

With a traditional self-managed database, you might think:

```text
Buy server
       ↓
Server capacity
```

DynamoDB pushes you toward:

```text
Requests
+
Storage
+
Throughput/capacity
       ↓
Cost
```

There are different capacity modes and pricing mechanisms, but the architectural lesson is:

> **Your access pattern directly influences your DynamoDB cost.**

A badly designed workload can therefore be:

```text
Technically scalable
        +
Financially terrible
```

That's a legitimate HLD consideration.

---

# 19. DynamoDB vs Cassandra

This is probably the most important comparison in this chapter.

|                        | Cassandra                  | DynamoDB                               |
| ---------------------- | -------------------------- | -------------------------------------- |
| Distributed database   | Yes                        | Yes                                    |
| Horizontal scale       | Core strength              | Core strength                          |
| High availability      | Yes                        | Yes                                    |
| Query-driven model     | Strongly                   | Strongly                               |
| Partition keys matter  | Extremely                  | Extremely                              |
| Managed service        | No, typically self-managed | Yes                                    |
| Operational burden     | Higher                     | Much lower                             |
| Infrastructure control | Higher                     | Lower                                  |
| AWS integration        | External                   | Excellent                              |
| Cost model             | Infrastructure-based       | Managed-service/request-capacity based |
| Portability            | Higher                     | Lower / AWS-specific                   |

The biggest difference isn't:

> "Cassandra is faster."

or:

> "DynamoDB is better."

It's:

```text
Cassandra
   ↓
You operate the distributed database

DynamoDB
   ↓
AWS operates the distributed database
```

That's the architectural decision.

---

# 20. When Would I Choose Cassandra?

Suppose:

```text
Huge scale
+
High availability
+
Predictable queries
+
Need control over deployment
+
Need more infrastructure/database flexibility
```

Cassandra becomes interesting.

Especially if:

```text
Cloud portability
or
self-managed infrastructure
```

matters.

---

# 21. When Would I Choose DynamoDB?

Suppose:

```text
AWS-native application
+
Massive scale
+
Predictable key-based access
+
Low operational overhead
+
Automatic scaling
+
High availability
```

DynamoDB becomes extremely attractive.

You essentially say:

> "I don't want my team spending engineering time operating the distributed database."

That's a legitimate architecture decision.

---

# 22. When Would I Choose Neither?

Suppose requirements are:

```text
Complex joins
+
Ad-hoc queries
+
Rich relational constraints
+
Analytical queries
+
Strong transactional workflows
```

Then:

```text
DynamoDB
   ↓
probably wrong abstraction
```

Likewise Cassandra.

We'd probably start looking toward:

```text
PostgreSQL
MySQL
```

or perhaps a specialized analytical/search system depending on the workload.

---

# 23. DynamoDB vs MongoDB

This is another useful comparison.

### MongoDB

```text
Document-oriented
+
Flexible document queries
+
Rich aggregation
+
Flexible schema
```

### DynamoDB

```text
Key-oriented
+
Predictable access patterns
+
Massive scale
+
Managed infrastructure
```

So if someone says:

> "I have JSON documents, therefore DynamoDB."

That's insufficient.

Ask:

> **How do you query them?**

If the application needs flexible document queries and aggregations:

```text
MongoDB
```

may be more natural.

If the application mostly does:

```text
Get item by key
Query items within a partition
```

at enormous scale:

```text
DynamoDB
```

can be a very strong choice.

---

# 24. A Real HLD Scenario

Imagine we're building a **shopping cart service**.

Requirements:

```text
100 million users
Millions of active carts
Very high request volume
Cart lookup by userId
Cart updates by userId
Low latency
High availability
```

We don't need:

```text
Complex joins
Ad-hoc SQL
Relational constraints
```

The dominant access pattern is:

```text
userId
   ↓
cart
```

DynamoDB becomes a strong candidate.

Conceptually:

```text
Cart
------------------------
userId       ← partition key
updatedAt    ← sort key / metadata
items
total
...
```

Application:

```text
GET cart(userId)
       ↓
DynamoDB
       ↓
Cart
```

Very straightforward.

---

# 25. Now Change One Requirement

Suppose product management suddenly says:

> "We need to run arbitrary queries over all carts:
>
> - carts containing product X
> - carts abandoned for more than 7 days
> - carts above ₹10,000
> - carts grouped by geography
> - arbitrary combinations of these filters."

Now DynamoDB becomes much less attractive as the primary query engine.

Why?

Because we've moved from:

```text
Predictable key access
```

toward:

```text
Flexible querying
```

And that's precisely the tradeoff DynamoDB makes.

---

# 26. The Most Important DynamoDB Tradeoff

If the interviewer asks:

> **"What's the biggest drawback of DynamoDB?"**

Don't say:

> "It's NoSQL."

That's meaningless.

A much better answer:

> **"DynamoDB works extremely well when access patterns are known and can be expressed through keys and indexes, but that predictability is also its constraint. If requirements evolve toward arbitrary queries, complex relationships, or workloads that don't map cleanly to the key model, DynamoDB can become difficult or expensive to use."**

That's the answer I want you to internalize.

---

# 27. DynamoDB Mental Model

Keep this:

```text
                     DynamoDB
                         │
                  Fully managed
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
        Distributed              Key-driven
        infrastructure           access
              │                     │
              ▼                     ▼
        High availability     Partition key
        Massive scale         Sort key
                              Indexes
                                  │
                                  ▼
                         Predictable queries
```

And the tradeoff:

> **DynamoDB trades query flexibility and infrastructure control for managed operations, predictable low-latency access, and massive horizontal scalability.**

---

# 28. Cassandra → DynamoDB: The Bigger Lesson

This is actually more important than memorizing DynamoDB features.

We started with:

```text
Cassandra
```

and asked:

> How do we build a database that can survive enormous scale and failures?

Then:

```text
DynamoDB
```

asks:

> How can we provide similar distributed-database capabilities without requiring the customer to operate the infrastructure?

So the technology choice isn't just about database features.

It's also about:

```text
Who owns the complexity?
```

### Cassandra

```text
Your team
    ↓
Cluster management
Replication
Capacity
Nodes
Repairs
Operations
```

### DynamoDB

```text
AWS
    ↓
Underlying distributed infrastructure
```

Your team instead focuses primarily on:

```text
Data model
Keys
Indexes
Access patterns
Consistency
Capacity/cost
```

That is a **major architectural tradeoff**.

---

# 29. Our Database Map So Far

We're starting to get a pretty useful map:

```text
                         Database choice
                               │
       ┌───────────────────────┼────────────────────────┐
       │                       │                        │
       ▼                       ▼                        ▼
   Relational              Documents             Distributed
       │                       │                   key-oriented
   ┌───┴───┐                   │                   ┌────┴────┐
   ▼       ▼                   ▼                   ▼         ▼
 MySQL PostgreSQL          MongoDB             Cassandra DynamoDB
   │       │                   │                   │         │
 OLTP   Rich/extensible    Aggregates          Scale     Managed scale
 joins  relational data    flexibility         + HA        + HA
```

And now we're ready for a completely different problem.

So far, we've mostly asked:

> **Where should operational data live?**

Next:

> **What if the main problem isn't storing the data, but finding information inside a huge amount of text quickly?**

That's where we move from databases toward:

# **Chapter 6 — Elasticsearch**

And this one will be particularly important because we'll finally see why:

> **"Just put an index on the database"**

isn't enough for serious search workloads.
