# Module 9 — Chapter 4: Cassandra

We've reached an important shift.

So far:

```text
MySQL
   → relational model

PostgreSQL
   → relational model + richer capabilities

MongoDB
   → document model + flexible aggregates
```

Cassandra asks a different question:

> **What if our dominant problem isn't complex data relationships, but distributing enormous amounts of data and traffic across many machines?**

This chapter is especially important for you because Cassandra is where several Module 2 concepts become concrete.

---

# 1. The Problem

Imagine we're building a global activity/event platform.

Users generate events:

```text
User 42
  → viewed product
  → searched laptop
  → added item to cart
  → viewed product
  → purchased
```

At a global scale:

```text
10 million events/sec
```

And the data keeps growing:

```text
1 TB
10 TB
100 TB
1 PB
...
```

Our requirements might be:

- enormous write throughput
- enormous storage capacity
- high availability
- horizontal scaling
- geographically distributed users
- predictable queries
- no dependence on complex joins

Notice what's _not_ on the list:

```text
Complex JOINs
Rich relational constraints
Complex transactions
```

That's the key.

---

# 2. Why a Traditional Relational Database Becomes Difficult

We could start with:

```text
MySQL
```

and scale vertically:

```text
Bigger machine
     ↓
Bigger machine
     ↓
Bigger machine
```

Eventually we hit practical limits.

So we introduce replication:

```text
             Primary
            /       \
           ▼         ▼
       Replica     Replica
```

Great for read scaling.

But our problem is primarily:

> **10 million writes/sec and petabytes of data.**

All writes still need to be distributed.

So we eventually need:

```text
                 Application
                     │
                 Sharding
              /      |      \
             ▼       ▼       ▼
          Node A   Node B   Node C
```

Now we have to deal with:

- partitioning
- rebalancing
- replication
- failures
- consistency
- routing
- availability

And this is exactly the class of problem Cassandra was designed around.

---

# 3. The Big Idea

> **Cassandra is a distributed database designed to provide high availability and massive horizontal scalability by distributing data across many nodes, while allowing the application to choose consistency appropriate to the operation.**

The important words are:

```text
Distributed
+
Highly available
+
Horizontally scalable
+
Partition-oriented
```

Don't think:

> "Cassandra is a SQL database but faster."

That's the wrong mental model.

Think:

> **Cassandra starts with distribution as a fundamental requirement.**

---

# 4. Cassandra's Fundamental Model

At a high level:

```text
                    Cassandra Cluster

          ┌────────────┬────────────┐
          │            │            │
          ▼            ▼            ▼
       Node A       Node B       Node C
          │            │            │
          └────────────┼────────────┘
                       │
                  Distributed
                     data
```

There isn't a single primary database node through which every write must pass.

This is one of the biggest architectural differences from the traditional primary-replica model we've been discussing.

Cassandra uses a **peer-to-peer architecture**.

Conceptually:

```text
Node A ↔ Node B ↔ Node C ↔ Node D
```

Nodes cooperate as equals.

That has enormous consequences for scaling and availability.

---

# 5. Why No Single Primary?

Imagine:

```text
            Primary
           /       \
          ▼         ▼
       Replica   Replica
```

If the primary disappears:

```text
Primary
   X
   ↓
Failover
   ↓
New primary
```

That's workable.

But Cassandra asks:

> Why make one node the central write authority at all?

Instead:

```text
      Node A
      ↕   ↕
Node B ↔ Node C
      ↕   ↕
      Node D
```

Data is distributed across the cluster.

Any appropriate node can receive a request and coordinate the operation.

This reduces dependence on a single leader.

---

# 6. Data Distribution — Partitioning

Here's where Module 2 comes back.

Suppose we have:

```text
Event
{
    userId,
    timestamp,
    eventType
}
```

We need to determine:

> **Which node stores this data?**

Cassandra uses a **partition key**.

For example:

```text
partition key = userId
```

Conceptually:

```text
hash(userId)
      ↓
partition
      ↓
nodes
```

So:

```text
User 100 → Partition X
User 101 → Partition Y
User 102 → Partition Z
```

The exact routing mechanism involves Cassandra's partitioning/ring architecture, but for HLD purposes the important idea is:

> **The partition key determines where a row belongs in the distributed system.**

---

# 7. Partition Key vs Clustering Key

This distinction is extremely important.

Suppose our access pattern is:

> Get all events for a user ordered by time.

We might conceptually model:

```text
Partition key:
    userId

Clustering key:
    timestamp
```

So:

```text
Partition: userId = 42

┌────────────────────────────┐
│ timestamp | eventType      │
├────────────────────────────┤
│ 10:00     | login          │
│ 10:05     | search         │
│ 10:12     | product_view   │
│ 10:20     | purchase       │
└────────────────────────────┘
```

The partition determines:

> **Which group of data belongs together.**

The clustering key determines:

> **How rows within that partition are organized.**

This is a very different way of thinking from designing normalized relational tables.

---

# 8. Cassandra Data Modeling Starts With Queries

This is perhaps Cassandra's most important concept.

In MySQL, you might think:

```text
What entities do I have?
How should I normalize them?
```

In MongoDB:

```text
What aggregates do I retrieve together?
```

In Cassandra:

> **What exact queries must this system support?**

Suppose the requirement is:

```text
Get events for user X
between time A and B
ordered by time
```

We design the table around that query.

Conceptually:

```text
user_events
--------------------------------
user_id       ← partition key
timestamp     ← clustering key
event_type
metadata
```

Now the query maps naturally to the storage layout.

---

# 9. This Leads to a Crucial Cassandra Principle

> **In Cassandra, data modeling is query-driven.**

You may intentionally duplicate data.

Suppose we need:

```text
Query 1:
Get events by user

Query 2:
Get events by event type
```

We might create separate tables:

```text
events_by_user
events_by_type
```

The same underlying event may therefore exist in both.

In a relational database, duplication might make you uncomfortable.

In Cassandra:

> **Denormalization is often intentional.**

Why?

Because Cassandra optimizes for predictable, scalable reads rather than arbitrary relational querying.

---

# 10. Why Not Just Use MongoDB?

Excellent question.

Both are distributed NoSQL databases.

But their design priorities differ.

### MongoDB

Think:

```text
Document
   ↓
Flexible aggregate
   ↓
Rich document queries
```

### Cassandra

Think:

```text
Query
   ↓
Partition key
   ↓
Distributed storage
   ↓
Predictable access
```

MongoDB gives you a richer document/query model.

Cassandra gives you a more constrained model optimized around massive distributed workloads and availability.

---

# 11. Replication

Now let's bring back another Module 2 concept.

Suppose:

```text
Replication Factor = 3
```

A piece of data might conceptually be stored on:

```text
Node A
Node B
Node C
```

So:

```text
                 Data
              /    |    \
             ▼     ▼     ▼
          Node A Node B Node C
```

If Node B dies:

```text
Node A ✓
Node B ✗
Node C ✓
```

The data still has replicas.

This gives us fault tolerance.

---

# 12. But Here's Where Cassandra Gets Interesting

Suppose we have three replicas:

```text
R = 3
```

and we perform a write.

Do we need all three replicas to acknowledge it before returning success?

Not necessarily.

Cassandra lets us choose a **consistency level** for operations.

For example:

```text
ONE
QUORUM
ALL
```

Conceptually:

```text
R = 3

ONE
→ one replica acknowledgement

QUORUM
→ majority acknowledgement

ALL
→ all replicas
```

For three replicas:

```text
QUORUM = 2
```

This gives us a tunable tradeoff between:

```text
Latency
Availability
Consistency
```

---

# 13. Why Is This Powerful?

Imagine a read-heavy system where slightly stale data is acceptable.

You might choose a weaker consistency level:

```text
Read → ONE
```

This can potentially provide:

```text
lower latency
+
higher availability
```

Now imagine a critical operation where stronger consistency is required.

You could choose:

```text
Read → QUORUM
Write → QUORUM
```

The system can coordinate enough replicas to provide stronger guarantees.

The important mental model:

> **Cassandra does not force one global consistency level for every operation.**

Consistency can be part of the application decision.

---

# 14. But Don't Misunderstand This

It does **not** mean:

> "Choose QUORUM and Cassandra becomes a relational database."

You still have a fundamentally distributed, partition-oriented system.

The consistency level controls replica coordination.

It doesn't magically give you:

```text
JOINs
Foreign keys
Arbitrary transactions
Relational constraints
```

The underlying model remains different.

---

# 15. Eventual Consistency

Suppose:

```text
R = 3
```

and a write reaches:

```text
Node A ✓
Node B ✓
Node C ... delayed
```

The system can return success depending on the consistency level.

For a short period:

```text
A → new value
B → new value
C → old value
```

The replicas can converge.

That's the basic intuition behind eventual consistency.

This connects directly to what you learned earlier:

```text
Replication
    ↓
Network delay
    ↓
Replicas temporarily disagree
    ↓
Consistency model
```

---

# 16. Cassandra and CAP

This is one of the places where your Module 2 knowledge becomes concrete.

Suppose there's a network partition:

```text
Node A     Node B
   │          │
   │   X      │
   │          │
Node C     Node D
```

The cluster is divided.

Cassandra is designed with a strong emphasis on:

> **Availability under partition.**

Rather than requiring the entire distributed system to stop accepting requests whenever communication between nodes is disrupted.

That means you must accept weaker consistency characteristics in some situations.

So your mental model can be:

```text
Network partition
       ↓
Cassandra prioritizes
availability
       ↓
Consistency may be weaker
depending on configuration/workload
```

This is not "Cassandra doesn't care about consistency."

It provides **tunable consistency**.

But its architectural design strongly favors availability and partition tolerance.

---

# 17. What Happens During a Node Failure?

Suppose:

```text
Node A ✗
```

but replicas exist elsewhere.

Other nodes can continue serving data.

The system can later repair/reconcile replicas.

Conceptually:

```text
Failure
   ↓
Other replicas continue
   ↓
Requests continue
   ↓
Failed node returns
   ↓
Data repaired/reconciled
```

This is fundamentally different from architectures where a primary failure immediately requires a leader failover before writes can continue.

---

# 18. The Huge Benefit: Horizontal Scaling

Suppose:

```text
10 nodes
```

isn't enough.

Add:

```text
Node 11
Node 12
Node 13
```

Data can be redistributed.

Conceptually:

```text
             Cassandra Cluster

Before:
[A] [B] [C] [D]

After:
[A] [B] [C] [D] [E] [F] [G]
```

This is the fundamental scalability story.

Instead of:

```text
One gigantic database
```

we build:

```text
Many cooperating nodes
```

---

# 19. But There's a Catch: Partitioning

Remember:

> **Adding nodes only helps if data and traffic distribute well.**

Suppose:

```text
userId
```

is the partition key.

Most users generate similar amounts of traffic.

Great.

But suppose one user is Taylor Swift and generates:

```text
10 million requests/sec
```

while everyone else generates:

```text
100 requests/sec
```

Then:

```text
Taylor's partition
       ↓
hot partition
       ↓
one/few nodes overloaded
```

Your cluster might have:

```text
100 nodes
```

but the hot partition still limits you.

This is the same principle we saw earlier:

> **Horizontal scale is only useful when workload distribution is healthy.**

---

# 20. The Hot Partition Problem

Imagine:

```text
events_by_user
```

with:

```text
partition key = userId
```

Most users:

```text
100 events/day
```

One customer:

```text
1 billion events/day
```

Now that customer's data is concentrated around one partition key.

Potential solution:

```text
partition key =
(userId, timeBucket)
```

For example:

```text
userId = 42
day = 2026-10-04
```

Then:

```text
42 + Oct 4
42 + Oct 5
42 + Oct 6
...
```

This distributes the workload over time.

But now queries must understand the bucket.

Again:

```text
Scale
  ↕
Query complexity
```

Tradeoff.

---

# 21. Cassandra Has a Different Definition of "Flexible"

MongoDB says:

> "Your documents can have flexible shapes."

Cassandra's flexibility is different.

Cassandra is much more structured around:

```text
partition key
clustering columns
query patterns
```

So you shouldn't think:

```text
Cassandra = schema-less
```

Instead:

> **Cassandra gives you flexibility in distributing and modeling data around access patterns, but it intentionally restricts arbitrary querying.**

That restriction is a feature.

---

# 22. Why Restrict Queries?

Imagine allowing arbitrary queries across:

```text
1 PB
100 nodes
```

You ask:

> Find all events where `eventType = purchase` and `country = India`.

If neither is a good partitioning dimension, the system might need to inspect enormous amounts of distributed data.

That's expensive.

Cassandra instead wants you to say:

```text
I need this query.
```

Then design:

```text
partition key
+
clustering columns
```

around it.

So Cassandra trades:

```text
Query flexibility
```

for:

```text
Predictable distributed performance
```

That's one of its defining tradeoffs.

---

# 23. Cassandra vs MongoDB

Now the comparison becomes clearer.

|                              | MongoDB              | Cassandra                                 |
| ---------------------------- | -------------------- | ----------------------------------------- |
| Core model                   | Documents            | Distributed wide-column/partitioned model |
| Main design focus            | Aggregates/documents | Query-driven partitions                   |
| Schema flexibility           | High                 | More structured                           |
| Rich document querying       | Strong               | Limited                                   |
| Horizontal scale             | Strong               | Core strength                             |
| Availability                 | High                 | Very high                                 |
| Arbitrary queries            | More flexible        | Intentionally limited                     |
| Denormalization              | Common               | Fundamental                               |
| Huge write workloads         | Good                 | Particularly strong                       |
| Access-pattern-driven design | Important            | Absolutely central                        |

The important distinction:

```text
MongoDB:
"What document represents my object?"

Cassandra:
"What partition represents my query?"
```

That's an excellent interview mental model.

---

# 24. Cassandra vs MySQL

Now compare the extremes:

```text
MySQL
 ↓
Rich relational model
 ↓
Transactions
 ↓
Joins
 ↓
Constraints
```

versus:

```text
Cassandra
 ↓
Distributed partition model
 ↓
Massive horizontal scale
 ↓
High availability
 ↓
Predictable queries
```

Neither is "better."

They're optimizing for different problems.

---

# 25. When Would You Choose Cassandra?

Strong candidates include workloads with:

```text
Huge datasets
+
Very high write throughput
+
Horizontal scaling
+
High availability
+
Predictable access patterns
+
Limited need for joins
```

Examples can include:

- massive event/activity data
- telemetry
- time-oriented workloads
- large-scale user activity
- distributed workloads where availability is extremely important

But remember our accuracy rule: a specific company's architecture should be verified before presenting it as fact.

---

# 26. When Would You NOT Choose Cassandra?

Be suspicious if the requirements say:

```text
Complex joins
+
Ad-hoc queries
+
Strong relational constraints
+
Multi-row transactional workflows
+
Highly flexible querying
```

For example:

```text
Banking ledger
```

might naturally lead you toward a relational database.

Or:

```text
Business intelligence
```

might lead toward an analytical system rather than Cassandra.

The warning sign is:

> **You are trying to force Cassandra to behave like a relational database.**

---

# 27. The Most Important Cassandra Tradeoff

If I asked you:

> "What do you give up by choosing Cassandra?"

A strong answer is:

> **We gain massive horizontal scalability and high availability, but we sacrifice query flexibility and accept a more restrictive, query-driven data model. We also take on the complexity of designing good partition keys, avoiding hot partitions, and reasoning about consistency across replicas.**

That's a very good SDE II answer.

---

# 28. Cassandra Mental Model

Keep this:

```text
                 Cassandra
                     │
             Distributed first
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   Partitioning  Replication  Availability
        │            │            │
        └────────────┼────────────┘
                     ▼
              Query-driven
               data model
                     │
              ┌──────┴──────┐
              ▼             ▼
        Partition Key   Clustering Key
              │
              ▼
       Predictable access
```

And the central tradeoff:

> **Cassandra trades relational/query flexibility for massive horizontal scalability, high availability, and predictable distributed performance.**

---

# 29. The Connection Across Our First Four Technologies

This is worth pausing on because this is where Module 9 starts becoming useful.

We now have:

```text
                    Data problem
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
   Relational         Document        Distributed
       │                 │             partitions
       ▼                 ▼                 ▼
     MySQL           MongoDB         Cassandra
       │                 │                 │
 Transactions      Aggregates        Scale + HA
 Relationships     Flexibility       Predictable queries
 Joins             Nested data       Denormalization
```

And PostgreSQL sits beside MySQL:

```text
MySQL
   │
Conventional relational workloads

PostgreSQL
   │
Relational + richer/extensible capabilities
```

This is exactly the progression we want from Module 9.

We're not memorizing databases.

We're building a **technology selection map**.

---

# 30. One Final Interview Scenario

Suppose an interviewer says:

> "Design a system that stores billions of user activity events. We expect hundreds of thousands of writes per second. We need to retrieve events for a user over a time range. The system must remain available even if several machines fail. We don't need joins or complex transactions."

Your reasoning should now start:

```text
Billions of records
        ↓
High write throughput
        ↓
Horizontal scaling
        ↓
High availability
        ↓
Predictable query
        ↓
No joins
        ↓
Cassandra becomes a strong candidate
```

Then immediately ask:

> **"What's my partition key?"**

Potentially:

```text
(userId, timeBucket)
```

Then:

> **"What's my replication factor?"**

Then:

> **"What consistency level does this workload require?"**

Then:

> **"Can a single user become a hot partition?"**

Then:

> **"How does the system behave when a node fails?"**

That is the level of Cassandra reasoning I want you to develop.

---

## Next: DynamoDB

Now we have an interesting comparison coming.

Cassandra says:

> "Let's build a highly available, horizontally scalable distributed database."

AWS's DynamoDB takes a related set of ideas and packages them into a **fully managed cloud database**, with a very different operational model and some important differences in data modeling and consistency.

So the next question is:

> **If I want Cassandra-like distributed scale without managing a Cassandra cluster myself, what changes when I use DynamoDB?**

That's **Chapter 5 — DynamoDB**.
