# Module 9 — Technology Deep Dive #2: MySQL

We've finished Redis. Now we move to **MySQL**.

The important shift is that Redis and MySQL solve very different problems.

With Redis, we started from:

> "We need extremely fast access to hot/shared state."

With MySQL, we're starting from a much more fundamental problem:

> **"We need a durable system of record for structured business data, with relationships, constraints, and transactions."**

And because you've already learned SQL databases in Module 1, we're not going to relearn relational databases from scratch.

We're going to ask:

> **Why does MySQL make the architectural choices it does, how do those choices affect its behavior, and when would I choose it in an HLD?**

---

# 1. Why Does MySQL Exist?

Imagine we're building Amazon.

We need to store:

```text
Customers
Orders
Products
Payments
Addresses
Inventory
```

And these aren't independent blobs of data.

There are relationships:

```text
Customer
   │
   ├── places ──→ Orders
   │                 │
   │                 ├── contains → Products
   │                 │
   │                 └── has → Payment
   │
   └── has → Addresses
```

We need guarantees such as:

- An order shouldn't reference a customer that doesn't exist.
- A payment should correspond to a valid order.
- Two requests shouldn't accidentally overwrite each other's updates.
- A transaction involving several records should either succeed together or fail together.
- Data should survive application crashes.
- We should be able to ask flexible questions about our data.

This is the environment where a relational database shines.

---

# 2. The Big Idea

> **MySQL organizes durable structured data into related tables and provides mechanisms such as indexes, transactions, constraints, and concurrency control to safely read and modify that data.**

Think of MySQL less as:

> "A place where I run SQL."

and more as:

```text
                MySQL
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
   Structured   Durable   Transactional
     data        data        updates
       │
       ▼
    Indexes
       │
       ▼
 Efficient access
```

Those properties are what make MySQL useful in architecture.

---

# 3. Why Not Just Store Everything in Redis?

We just spent a lot of time learning Redis.

So let's make the comparison immediately.

Suppose we have:

```text
Customer
 ├── id
 ├── name
 └── email

Order
 ├── id
 ├── customer_id
 ├── created_at
 └── total
```

With a relational database, we can explicitly model:

```text
Customer 1 ──────── N Orders
```

and enforce that relationship.

Redis could store:

```text
customer:123
order:456
```

but now **the application has to enforce much more of the structure itself**.

For example:

> Does `order:456` reference a valid customer?

MySQL can enforce relational constraints.

Redis isn't designed primarily around this kind of relational integrity.

So:

```text
Redis
→ fast access to known data

MySQL
→ durable structured business data + relationships + transactions
```

That's the first major distinction.

---

# 4. The Relational Model

At the heart of MySQL is the relational model.

Suppose we have:

```text
users
+----+--------+------------------+
| id | name   | email            |
+----+--------+------------------+
| 1  | Alice  | alice@example... |
| 2  | Bob    | bob@example...   |
+----+--------+------------------+
```

And:

```text
orders
+----+---------+--------+
| id | user_id | total  |
+----+---------+--------+
| 10 | 1       | 500    |
| 11 | 1       | 200    |
| 12 | 2       | 900    |
+----+---------+--------+
```

The important thing isn't merely that these are tables.

It's that we have a **relationship**:

```text
users.id
    ↑
    │
orders.user_id
```

This structure gives us powerful querying capabilities.

For example:

> "Give me all orders placed by Alice."

Conceptually:

```text
users
   │
   │ join
   ▼
orders
```

---

# 5. Why Relationships Matter in HLD

Consider an e-commerce system.

You might need:

> Find all orders for customers in Bangalore that contain products from category X and were placed in the last 30 days.

That's a query spanning multiple concepts:

```text
Customers
    ↓
Orders
    ↓
Order Items
    ↓
Products
    ↓
Categories
```

A relational database is designed around this kind of structured relationship.

This is fundamentally different from a key-value model where your primary question is:

```text
"What is the value for key X?"
```

The relational model says:

> **"Let me describe my data and relationships, and then ask questions about that model."**

---

# 6. But Relationships Create a Cost

Nothing comes for free.

Relational queries can become expensive.

Suppose we have:

```text
10 million users
500 million orders
```

and run:

```text
Find all orders for user X
```

A naïve database might have to inspect huge amounts of data.

This is where **indexes** become crucial.

---

# 7. Indexes — The First Major MySQL HLD Concept

Imagine a book with:

```text
1000 pages
```

You want to find every occurrence of:

```text
"distributed systems"
```

You could read every page.

Or you could use the index at the back.

A database index serves a similar purpose.

Without an appropriate index:

```text
Query
 ↓
Inspect many rows
 ↓
Find matches
```

With an index:

```text
Query
 ↓
Index
 ↓
Locate relevant rows
```

The fundamental tradeoff:

> **Indexes make reads faster by maintaining additional data structures, but they consume storage and make writes more expensive.**

---

# 8. Why Do Indexes Make Writes More Expensive?

Suppose we have:

```text
users
```

and indexes on:

```text
id
email
created_at
```

Now we insert:

```text
new user
```

The database doesn't just add one row.

It also has to update the relevant index structures.

Conceptually:

```text
INSERT
  │
  ├── table
  ├── id index
  ├── email index
  └── created_at index
```

So:

```text
More indexes
    ↓
Faster certain reads
    ↓
More write work
+
More storage
```

This is a very important HLD tradeoff.

---

# 9. What Does an Index Actually Look Like?

For many common MySQL workloads, the important mental model is a **B-tree-style ordered index**.

You don't need to memorize the implementation details.

Think:

```text
                Index
                  │
            ┌─────┴─────┐
            ▼           ▼
          A-M           N-Z
         /   \         /   \
        ...  ...      ...  ...
```

The structure is ordered and balanced so that the database can narrow down where the desired values are rather than scanning every row.

For example:

```sql
WHERE email = 'alice@example.com'
```

can use an index on:

```text
email
```

rather than scanning the entire users table.

---

# 10. The Important Interview Question

Suppose someone says:

> "Let's put indexes on every column."

Sounds good, right?

Not necessarily.

Imagine:

```text
20 columns
```

and you index all 20.

Now every write potentially has to maintain many index structures.

You also consume more storage and memory.

And some indexes may never actually be useful.

So the correct principle is:

> **Index according to the application's access patterns, not according to the schema alone.**

Ask:

```text
What queries are frequent?
What filters are common?
What ordering is common?
What joins are common?
Which queries are latency-sensitive?
```

Then design indexes around those queries.

---

# 11. Composite Indexes

Now suppose we frequently query:

```sql
WHERE user_id = ?
AND created_at > ?
ORDER BY created_at DESC
```

A single-column index on `user_id` may help.

But a composite index can be designed around the access pattern:

```text
(user_id, created_at)
```

Conceptually:

```text
user_id
   ↓
created_at
```

This lets the database narrow down the relevant user's records and then efficiently navigate by time.

The deeper lesson:

> **Database schema and index design should follow the queries your system actually needs to execute.**

This becomes extremely important at scale.

---

# 12. Query Execution

Now we have an index.

But how does MySQL actually answer:

```sql
SELECT *
FROM orders
WHERE user_id = 123
AND created_at > '2026-01-01';
```

At a high level:

```text
SQL query
   ↓
Parse
   ↓
Understand query structure
   ↓
Choose an execution strategy
   ↓
Use indexes / scan data
   ↓
Retrieve rows
   ↓
Return result
```

The database has to decide:

> "What's the cheapest way to execute this?"

This is the role of the **query optimizer**.

---

# 13. The Query Optimizer

Imagine:

```text
Table A = 1 million rows
Table B = 10 rows
```

and we're joining them.

There are multiple possible execution strategies.

The optimizer tries to estimate:

```text
Which indexes?
Which table first?
Which join strategy?
How many rows will each step produce?
```

and chooses a plan.

So when a query is slow, the problem isn't necessarily:

> "The database is slow."

It might be:

```text
Bad query
    ↓
Poor execution plan
    ↓
Too much data scanned
    ↓
High latency
```

This is why HLD engineers should understand the concept of **query execution**, even if they're not database administrators.

---

# 14. Why Indexes Don't Automatically Make Every Query Fast

Suppose we have:

```text
10 million users
```

and an indexed column:

```text
country
```

But:

```text
9 million users = India
1 million = other countries
```

Now query:

```sql
WHERE country = 'India'
```

The index may still identify a huge portion of the table.

An index isn't magic.

Its usefulness depends on:

```text
Selectivity
+
Query shape
+
Data distribution
+
Available indexes
```

This is a subtle but important interview point.

---

# 15. Transactions

Now we reach one of MySQL's most important architectural capabilities.

Suppose we're transferring money:

```text
Alice = ₹1000
Bob   = ₹500
```

Transfer ₹200:

```text
Alice = 800
Bob   = 700
```

There are two changes.

What if:

```text
Alice updated
      ↓
Application crashes
      ↓
Bob not updated
```

Now:

```text
Alice = 800
Bob = 500
```

₹200 disappeared.

That's unacceptable.

We need the two operations to behave as one logical unit.

```text
BEGIN

Alice -= 200
Bob += 200

COMMIT
```

If something goes wrong:

```text
ROLLBACK
```

The transaction provides the abstraction:

> **Either the logical operation succeeds as a unit, or its changes are not committed.**

This is one of the fundamental reasons relational databases remain so important.

---

# 16. ACID

You've encountered this before, but now we're connecting it to actual technology.

### Atomicity

```text
All changes
OR
no changes
```

### Consistency

The database moves from one valid state to another according to its defined constraints and rules.

### Isolation

Concurrent transactions should behave according to the chosen isolation guarantees.

### Durability

Once committed, data should survive failures according to the database's durability guarantees.

The key HLD insight:

> **Transactions aren't merely a database feature. They're a tool for preserving business invariants.**

For example:

```text
Order created
+
Inventory reserved
+
Payment recorded
```

may have relationships that require careful transactional boundaries.

---

# 17. Concurrency

Now imagine:

```text
Stock = 1
```

Two users simultaneously purchase the last item.

```text
User A ──┐
         ├──→ MySQL
User B ──┘
```

Both want:

```text
stock = stock - 1
```

We need concurrency control so we don't end up with:

```text
stock = -1
```

or two successful purchases for one item.

This brings us to:

> **How does MySQL control concurrent access to the same data?**

Through mechanisms including **locking and isolation**.

---

# 18. Locks

A simple mental model:

```text
Transaction A
    ↓
locks row
    ↓
updates row
    ↓
commits
    ↓
releases lock
```

Meanwhile:

```text
Transaction B
    ↓
tries same row
    ↓
waits
```

This prevents certain conflicting operations from happening simultaneously.

This should connect directly to what you learned in Java concurrency:

```text
Shared state
    ↓
Concurrent access
    ↓
Need coordination
```

The difference is that MySQL is coordinating **database state across concurrent transactions**, rather than Java objects inside one process.

---

# 19. But Locks Create Another Problem

Suppose:

```text
Transaction A
    locks Row 1
    waits for Row 2

Transaction B
    locks Row 2
    waits for Row 1
```

We have:

```text
A → waiting for B
B → waiting for A
```

That's a deadlock.

This is the same fundamental problem you encountered when implementing bank transfers with Java locks.

And the same solution principle appears:

> **Impose a consistent ordering on resource acquisition where possible.**

Database engines can detect deadlocks and abort one transaction so the system can make progress.

Again, the broader lesson:

> **Concurrency control protects correctness, but introduces waiting and deadlock risks.**

---

# 20. Isolation

Suppose transaction A is modifying data while transaction B reads it.

What should B see?

Possibilities include:

```text
Uncommitted data?
Old committed data?
A consistent snapshot?
Latest committed value?
```

These are **isolation semantics**.

You don't need to memorize every isolation detail yet.

The important HLD understanding is:

> **Isolation determines how much concurrent transactions are allowed to observe each other's intermediate changes.**

And stronger isolation generally means:

```text
More correctness guarantees
        ↓
More coordination
        ↓
Potentially less concurrency / more overhead
```

Again:

> **Tradeoff.**

---

# 21. MySQL's Core Architecture

Let's build a useful mental model.

At a high level:

```text
                 Application
                      │
                      ▼
                 MySQL Server
                      │
             ┌────────┴────────┐
             ▼                 ▼
       Query Processing     Transactions
             │                 │
             ▼                 ▼
        Query Optimizer   Concurrency Control
             │
             ▼
          Storage Engine
             │
             ▼
           Disk
```

This separation matters.

The SQL layer understands:

```text
"What does the application want?"
```

The storage engine is responsible for much of:

```text
"How do we store and retrieve the data?"
```

For typical MySQL deployments, **InnoDB** is the storage engine you'll care about for HLD.

We don't need to dive into every storage-engine implementation detail.

---

# 22. Why InnoDB Matters

For your HLD purposes, associate InnoDB with the capabilities we care about:

```text
InnoDB
   │
   ├── Transactions
   ├── Row-level locking
   ├── Crash recovery
   ├── Indexes
   └── Durable storage
```

This is the engine underlying the transactional behavior we're discussing.

The important takeaway isn't to memorize:

> "MySQL uses InnoDB."

It's to understand:

> **The storage engine provides much of the machinery that makes MySQL suitable for transactional workloads.**

---

# 23. Where MySQL Is Particularly Strong

MySQL is a particularly strong candidate when we have:

### Structured data

```text
Users
Orders
Payments
Products
```

### Relationships

```text
Customer → Orders → Products
```

### Transactional updates

```text
Payment
+
Order
+
Inventory
```

### Well-defined queries

```text
Find orders for customer X
Find products in category Y
Find unpaid invoices
```

### Strong durability requirements

```text
Business records
Financial records
Orders
```

### Mature operational ecosystem

It's been deployed at enormous scale for decades.

---

# 24. Where MySQL Starts Struggling

Now the other side.

Imagine:

```text
100 TB dataset
+
massive global traffic
+
millions of writes/sec
+
very low latency
+
complex global distribution
```

A single MySQL instance isn't going to handle this.

We need to scale.

And this takes us back to concepts you already learned:

```text
Replication
Sharding
Partitioning
Read scaling
Caching
```

The technology-specific question becomes:

> **How does MySQL actually apply those concepts?**

That's where our next chapters of MySQL will go.

---

# 25. MySQL Scaling — First Step: Read Replicas

Suppose:

```text
100k reads/sec
10k writes/sec
```

A single primary handles everything:

```text
             MySQL
           /       \
        Reads     Writes
```

If reads dominate, we can introduce replicas:

```text
                  Primary
                 /       \
                ▼         ▼
           Replica A   Replica B
```

Writes:

```text
Application
     ↓
Primary
```

Reads:

```text
Application
     ↓
Read replicas
```

Conceptually:

```text
                 ┌── Replica A
                 │
Application ─────┼── Replica B
                 │
                 └── Primary
                      ↑
                    Writes
```

Now read traffic can be distributed.

This is exactly the replication concept we learned earlier.

---

# 26. But Replication Creates a Problem

Remember Redis?

The exact same distributed-systems issue appears:

```text
Primary
   ↓
Replication
   ↓
Replica
```

There can be lag.

Suppose:

```text
Write:
email = new@example.com
```

goes to primary.

Immediately:

```text
Read → Replica
```

The replica might still contain:

```text
old@example.com
```

So read replicas give us:

```text
More read capacity
```

but potentially:

```text
Stale reads
```

This is why we don't simply say:

> "Add replicas."

We ask:

> **"Can this read tolerate replica lag?"**

---

# 27. The Architecture Is Becoming Familiar

This is deliberate.

Redis taught us:

```text
Replication
    ↓
More availability/read capacity
    ↓
Potential lag
```

Now MySQL:

```text
Replication
    ↓
More read capacity
    ↓
Potential lag
```

The underlying concept hasn't changed.

We're now learning:

> **How a real technology implements and exposes that concept.**

That's exactly what Module 9 is supposed to do.

---

# 28. MySQL's Fundamental Tradeoff

If I had to summarize MySQL's architectural identity in one diagram:

```text
                     MySQL
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
     Structured     Durable      Transactional
       data          data          updates
         │             │             │
         └─────────────┼─────────────┘
                       ▼
                 Rich querying
                       │
                       ▼
              More coordination
                       │
                       ▼
            Scaling becomes harder
```

This isn't a weakness.

It's the price of the guarantees MySQL provides.

Compare:

```text
Redis:
Optimize for extremely fast in-memory access
```

versus:

```text
MySQL:
Optimize for durable structured data and transactional correctness
```

Neither is universally "better."

---

# 29. The HLD Question You Should Start Asking

When you see a requirement like:

> "We need to store orders."

Your brain should no longer immediately say:

> "MySQL."

Instead:

```text
Orders
  ↓
Durable?
Yes
  ↓
Structured?
Yes
  ↓
Relationships?
Yes
  ↓
Transactions?
Yes
  ↓
Complex queries?
Likely
  ↓
Relational database is a strong candidate
  ↓
MySQL vs PostgreSQL vs alternatives
```

And _then_ we compare technologies.

That's the skill we're building.

---

# Key Takeaways

For today's first MySQL session, retain these:

1. **MySQL is fundamentally a durable relational database**, not simply "a database that supports SQL."

2. Its strength comes from combining:

   ```text
   Structured data
   +
   Relationships
   +
   Indexes
   +
   Transactions
   +
   Concurrency control
   +
   Durability
   ```

3. **Indexes trade write/storage overhead for faster reads.**

4. Indexes should be designed around **actual query patterns**, not blindly added to every column.

5. The **query optimizer** chooses how a query should be executed; a slow query may be an execution-plan/index problem rather than simply "MySQL being slow."

6. **Transactions protect business invariants**, not merely database rows.

7. Concurrency introduces:

   ```text
   Locks
   Isolation
   Waiting
   Deadlocks
   ```

8. MySQL can scale reads using **replicas**, but replication introduces potential **stale reads**.

9. MySQL's biggest strength is also part of its scaling challenge:
   > **The more correctness, relational querying, and transactional guarantees we demand, the more coordination the system needs.**

---

## Where We Go Next

We've established **what MySQL is and why it exists**.

The next important question is:

> **"A query arrives at MySQL. What actually happens?"**

We'll go deeper into **indexes, B+ trees, clustered vs secondary indexes, composite indexes, query execution, and why some queries become catastrophically slow at scale**.

That is where MySQL starts becoming useful as an HLD technology rather than just something you write SQL against.
