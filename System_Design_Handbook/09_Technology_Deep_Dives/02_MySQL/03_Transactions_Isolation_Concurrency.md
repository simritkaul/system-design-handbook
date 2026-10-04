Let's proceed. We'll keep the exercise implicit and continue the MySQL deep dive.

# MySQL — Part 3: Transactions, Isolation & Concurrency

We've now understood how MySQL **finds data efficiently**.

But efficient reads are only half the reason we'd choose MySQL.

The harder problem is:

> **What happens when many requests modify the same data at the same time?**

This is where MySQL becomes much more than a query engine.

---

## 1. The Problem: Two Users Buy the Last Item

Suppose:

```text
Product: PS5
Stock: 1
```

Two users click **Buy** almost simultaneously.

```text
User A ─────┐
            │
            ▼
         MySQL
            ▲
            │
User B ─────┘
```

Both requests effectively do:

```text
Read stock
If stock > 0
    decrease stock
    create order
```

Without proper coordination:

```text
A reads stock = 1
B reads stock = 1

A → stock = 0
B → stock = 0

A → order created
B → order created
```

Now we have:

```text
Stock = 0
Orders = 2
```

We sold something twice.

The problem isn't that MySQL can't store the data.

The problem is **concurrent modification of shared state**.

---

# 2. Why Application-Level Protection Isn't Enough

You might think:

```java
synchronized
```

We learned this in Java concurrency.

But imagine:

```text
Application Server 1
        │
Application Server 2
        │
Application Server 3
        │
        ▼
      MySQL
```

A Java `synchronized` block only coordinates threads **inside one JVM**.

It cannot coordinate:

```text
Server 1 ↔ Server 2
```

And it certainly can't coordinate another service written in Python or Go.

The shared resource is actually:

```text
             ┌── Server 1
             ├── Server 2
             ├── Server 3
             │
             ▼
           MySQL
```

Therefore the coordination must happen around the **shared database state**.

That's one reason databases need concurrency-control mechanisms.

---

# 3. Transactions

A transaction lets us group multiple operations into one logical unit.

For example, placing an order might involve:

```text
1. Check stock
2. Decrease stock
3. Create order
4. Create order items
5. Record payment state
```

We don't necessarily want:

```text
1 ✓
2 ✓
3 ✓
4 ✗
```

with the first three changes permanently remaining.

Instead:

```text
BEGIN

1. Check stock
2. Decrease stock
3. Create order
4. Create order items
5. Record payment state

COMMIT
```

If something fails:

```text
ROLLBACK
```

The important abstraction is:

> **A transaction groups related changes so they can be treated as one unit of work.**

---

# 4. ACID in Practical Terms

You've already encountered ACID conceptually.

Now connect each letter to an engineering problem.

### Atomicity

```text
Transfer ₹500

Debit Alice
Credit Bob
```

We don't want:

```text
Debit ✓
Credit ✗
```

Atomicity gives us:

```text
Both
OR
neither
```

---

### Consistency

Suppose:

```text
inventory >= 0
```

is an invariant.

A successful transaction should not leave the database violating its defined rules.

Think:

```text
Valid state
    ↓
transaction
    ↓
Valid state
```

---

### Isolation

Concurrent transactions shouldn't arbitrarily interfere with each other.

This is where things become interesting.

---

### Durability

Once a transaction is successfully committed, its result should survive failures according to the database's durability guarantees.

For example:

```text
COMMIT
  ↓
server crashes
  ↓
restart
  ↓
committed data remains
```

That's fundamentally different from storing important business state only in application memory.

---

# 5. Isolation Is the Interesting One

Imagine:

```text
Transaction A
    updates account balance

Transaction B
    reads account balance
```

Question:

> What exactly is B allowed to see while A is executing?

Suppose A changes:

```text
₹1000 → ₹500
```

but hasn't committed yet.

Should B see:

```text
₹500?
```

Or:

```text
₹1000?
```

This is the purpose of **transaction isolation**.

---

# 6. What Goes Wrong Without Enough Isolation?

There are several classic concurrency anomalies.

You don't need to memorize them as vocabulary first.

Understand the situations.

---

## Dirty Read

Transaction A changes data:

```text
A:
balance = ₹500
```

but hasn't committed.

Transaction B reads:

```text
₹500
```

Then A rolls back.

Actual database state:

```text
₹1000
```

B saw something that never actually became permanent.

That's a **dirty read**.

---

# 7. Non-Repeatable Read

Transaction A reads:

```text
balance = ₹1000
```

Then transaction B changes and commits:

```text
balance = ₹500
```

A reads again:

```text
₹500
```

Same transaction.

Different result.

That's a **non-repeatable read**.

The underlying issue:

> Another transaction changed the data between our reads.

---

# 8. Phantom Read

Now imagine:

```sql
SELECT *
FROM orders
WHERE user_id = 123;
```

A transaction sees:

```text
Order 1
Order 2
```

Another transaction inserts:

```text
Order 3
```

Then the first transaction executes the same query again.

Now:

```text
Order 1
Order 2
Order 3
```

A new row appeared inside the logical result set.

That's the intuition behind a **phantom read**.

---

# 9. Isolation Levels

So databases give us different isolation levels.

Conceptually:

```text
Less isolation
      ↓
More concurrency
      ↓
Fewer guarantees

More isolation
      ↓
More coordination
      ↓
Stronger guarantees
```

The exact levels are:

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

You don't need to memorize them as a ladder.

Understand the engineering tradeoff.

---

# 10. READ UNCOMMITTED

The database provides very weak isolation.

A transaction may potentially see changes that another transaction hasn't committed yet.

So:

```text
A writes
 ↓
B can potentially see it
 ↓
A rolls back
```

That's why dirty reads are possible.

It's rarely appropriate for important transactional business operations.

---

# 11. READ COMMITTED

Now a transaction only sees committed data.

So:

```text
A writes
 ↓
B cannot see uncommitted change
 ↓
A commits
 ↓
B can now see it
```

This removes dirty reads.

But the same transaction can still observe different committed values at different points.

So non-repeatable reads can occur.

---

# 12. REPEATABLE READ

Now imagine transaction A reads:

```text
balance = ₹1000
```

Another transaction changes the balance.

A performs the same logical read again.

The database can provide a consistent view so that A doesn't suddenly see the newer value in the same transaction.

This gives stronger isolation.

And importantly:

> **MySQL/InnoDB's default isolation level is REPEATABLE READ.**

That's a technology-specific detail worth remembering.

---

# 13. SERIALIZABLE

This aims for the strongest isolation semantics.

Conceptually:

```text
Transaction A
      ↓
Transaction B
```

behave much more like they executed one after another rather than freely interleaving.

The benefit:

```text
Strong correctness
```

The cost:

```text
More coordination
More blocking
Potentially lower concurrency
```

So you don't automatically say:

> "Use SERIALIZABLE because it's safest."

You ask:

> **Do we actually need those guarantees badly enough to pay the concurrency cost?**

That's the HLD mindset.

---

# 14. How Does MySQL Actually Enforce This?

Now we get to locking and MVCC.

There are two major ideas you should associate with InnoDB:

```text
Locking
+
MVCC
```

Let's take them separately.

---

# 15. Locking

Suppose:

```text
stock = 1
```

Transaction A wants to modify it.

Conceptually:

```text
A
 ↓
Acquire lock
 ↓
Modify row
 ↓
Commit
 ↓
Release lock
```

Transaction B comes along:

```text
B
 ↓
Wants same resource
 ↓
Can't proceed immediately
 ↓
Waits
```

This prevents conflicting modifications from occurring simultaneously.

---

# 16. Why Not Lock the Entire Table?

We could theoretically do:

```text
Transaction A
      ↓
Lock entire orders table
      ↓
Update one row
```

But that's terrible for concurrency.

Suppose we have:

```text
1 million orders
```

and A only needs:

```text
Order #123
```

Why should another transaction modifying:

```text
Order #999999
```

have to wait?

We want finer-grained coordination.

Hence **row-level locking**.

Conceptually:

```text
orders

Row 1   ← free
Row 2   ← locked by A
Row 3   ← free
Row 4   ← free
```

Another transaction can still work on:

```text
Row 1
Row 3
Row 4
```

This gives much better concurrency.

---

# 17. But Locks Cause Waiting

Now:

```text
A locks Row 1

B wants Row 1
```

B waits.

If we have:

```text
1000 requests
```

all trying to modify the same row:

```text
             Row X
               ↑
     ┌─────────┼─────────┐
     │         │         │
    A          B         C
     │         │         │
     └─────────┴─────────┘
             waiting
```

we've created a **contention hotspot**.

This becomes very important in HLD.

A database can technically support enormous throughput, but if your workload repeatedly modifies the same small set of rows, concurrency becomes constrained.

---

# 18. Hot Rows

Imagine a global counter:

```text
total_likes = 5,000,000
```

Every request does:

```text
total_likes++
```

Now thousands of requests are competing to modify one piece of state.

Even if MySQL is otherwise extremely capable:

```text
Many requests
      ↓
One row
      ↓
Contention
      ↓
Waiting
      ↓
Latency / throughput problem
```

This is a **hotspot**.

This connects directly to the "hot key" concept we learned with distributed caching.

Different technology.

Same underlying problem:

> **Too much traffic concentrated on too little state.**

---

# 19. MVCC

Now we arrive at an important concept.

**MVCC = Multi-Version Concurrency Control.**

The name sounds intimidating.

The basic idea is:

> **Instead of forcing every reader to wait for writers, the database can maintain enough version information to let readers see an appropriate version of the data.**

Imagine:

```text
Account balance

Version 1 → ₹1000
Version 2 → ₹800
```

A transaction may be able to read the appropriate committed version without blocking a concurrent writer in the same way a naïve locking system would.

Conceptually:

```text
Writer
  ↓
creates newer version

Reader
  ↓
reads appropriate visible version
```

This is a huge reason modern transactional databases can support significant concurrent read traffic.

---

# 20. Why MVCC Matters

Imagine:

```text
10,000 readers
+
100 writers
```

If every reader had to wait for every writer:

```text
Writer
  ↓
lock
  ↓
Readers wait
```

we'd get poor concurrency.

MVCC allows much more concurrency between readers and writers.

So:

```text
Locking
→ coordinate conflicting operations

MVCC
→ allow readers to work with appropriate versions
```

They're complementary ideas, not competing ones.

---

# 21. The Subtle Part: MVCC Doesn't Mean "No Locks"

This is an easy interview trap.

Someone might say:

> "MySQL uses MVCC, so reads never lock."

That's too simplistic.

MySQL/InnoDB still uses locks for various operations and isolation semantics.

Think:

```text
MVCC
+
Locking
+
Isolation rules
```

together determine how concurrent transactions behave.

Don't reduce MySQL concurrency to one mechanism.

---

# 22. Deadlocks

Let's revisit something you've already encountered in Java.

Transaction A:

```text
Lock Row 1
↓
needs Row 2
```

Transaction B:

```text
Lock Row 2
↓
needs Row 1
```

Now:

```text
A → waits for B
B → waits for A
```

Deadlock.

MySQL can detect this kind of cycle and abort one transaction.

For example:

```text
A continues
B rolled back
```

The application may then retry B.

This gives us a very important production principle:

> **Deadlocks aren't necessarily evidence that the database is broken. They are a consequence of concurrent transactions acquiring resources in conflicting orders.**

Good application/database design tries to minimize them.

---

# 23. A Better Transfer Design

Suppose we're transferring money:

```text
Account A
Account B
```

Bad ordering:

```text
Thread 1:
lock A
lock B

Thread 2:
lock B
lock A
```

Potential deadlock.

Better:

```text
Always lock lower account ID first.
```

So:

```text
Transfer A → B
    ↓
lock min(A,B)
    ↓
lock max(A,B)
    ↓
perform transfer
```

And:

```text
Transfer B → A
    ↓
lock min(A,B)
    ↓
lock max(A,B)
```

Both transactions acquire locks in the same order.

This is the exact same reasoning you used in your Java concurrency exercise.

That's intentional.

You're now seeing the same distributed-systems principles appearing inside a database.

---

# 24. Transaction Size Matters

Consider:

```text
BEGIN

Update row
Call payment provider
Send email
Call inventory service
Generate PDF
Update another row

COMMIT
```

This is dangerous.

Why?

Because the database transaction remains open while external operations happen.

Potentially:

```text
transaction open
      ↓
lock held
      ↓
network call
      ↓
another network call
      ↓
slow service
      ↓
lock held for seconds
```

Now other requests wait.

A good principle is:

> **Keep database transactions as short as reasonably possible.**

Don't hold database locks while waiting on unrelated external systems unless you have a very deliberate reason.

---

# 25. This Creates an HLD Boundary

Suppose your architecture is:

```text
Order Service
     │
     ├── MySQL
     │
     ├── Payment Service
     │
     └── Notification Service
```

You might want:

```text
Create order
+
Charge payment
+
Send notification
```

to be one atomic transaction.

But MySQL's transaction cannot magically provide atomicity across:

```text
MySQL
+
Payment Service
+
Notification Service
```

This is exactly where our Module 8 concepts become relevant:

```text
Distributed Transactions
Saga
Outbox
```

So technology knowledge is connecting back to architecture.

---

# 26. MySQL Isn't Just "ACID"

This is an important mindset correction.

You might hear:

> "MySQL is good because it's ACID."

That's true but incomplete.

A better architectural description is:

```text
MySQL
=
Relational data model
+
SQL querying
+
Indexes
+
Transactions
+
Concurrency control
+
Durability
+
Mature replication/scaling mechanisms
```

The combination is what makes it useful.

---

# 27. Where MySQL Starts Showing Its Limits

Now imagine:

```text
100 million users
10 billion orders
50,000 writes/sec
500,000 reads/sec
```

We can use:

```text
Indexes
Read replicas
Caching
```

But eventually we hit a bigger problem:

> **One logical dataset is becoming too large for one MySQL primary.**

We need to distribute the data itself.

And now we return to something you've already learned in Module 2:

```text
Sharding
```

---

# 28. Sharding MySQL

Suppose we have:

```text
1 billion users
```

Instead of:

```text
             MySQL
               │
        1 billion users
```

we could partition them:

```text
                 Application
                      │
              Shard Routing
               /     |     \
              ▼      ▼      ▼
          MySQL A  MySQL B  MySQL C
          Users    Users    Users
          1-333M   334-666M 667M-1B
```

Now each database handles only a subset of the data.

This increases:

```text
Storage capacity
Write capacity
Read capacity
```

potentially.

But it introduces enormous complexity.

---

# 29. The Cost of Sharding

Once data is distributed:

```text
Before:

MySQL
 └── all users


After:

Router
 ├── Shard A
 ├── Shard B
 └── Shard C
```

Now we need to answer:

> Which shard contains user X?

And what happens when we need:

```sql
JOIN data across shards?
```

Or:

```text
transaction spanning two shards?
```

Or:

```text
resharding?
```

Or:

```text
one shard becomes much hotter than others?
```

These are the costs of horizontal scaling.

And this is exactly why:

> **"Just shard the database"**

is never a complete architecture answer.

---

# 30. MySQL's Scaling Journey

The progression should feel familiar:

```text
                    MySQL
                      │
                      ▼
                 Single DB
                      │
                Read pressure
                      │
                      ▼
                Read replicas
                      │
                Write pressure
                      │
                      ▼
          Better indexes / caching
                      │
                Data too large
                      │
                      ▼
                  Sharding
                      │
              Operational complexity
                      │
                      ▼
           Need careful architecture
```

Notice how the architecture evolves because each solution introduces another constraint.

That's exactly how we learned system design in Modules 1–8.

---

# 31. The Important MySQL Tradeoff

MySQL gives us powerful guarantees:

```text
Transactions
Strong consistency options
Relational integrity
Flexible queries
Durability
```

But these aren't free.

They require:

```text
Coordination
Locks
Indexes
Storage
Replication
Resource management
```

And at sufficiently large scale:

```text
More coordination
        ↓
Harder horizontal scaling
```

So MySQL is not "bad at scale."

That's an important distinction.

**MySQL can absolutely operate at enormous scale.**

The point is:

> Scaling a relational, transactional system while preserving those guarantees requires increasingly sophisticated architecture.

---

# 32. The Mental Model You Should Leave With

Picture MySQL like this:

```text
                       MySQL
                         │
              ┌──────────┴──────────┐
              │                     │
           Queries              Transactions
              │                     │
         ┌────┴────┐          ┌────┴─────┐
         │         │          │          │
      Indexes    Optimizer  Isolation   Locks
         │                     │          │
      B+ Trees                MVCC       │
         │                     │          │
         └─────────────┬───────┴──────────┘
                       │
                       ▼
                  Storage Engine
                       │
                       ▼
                     Disk
```

And at scale:

```text
                   Application
                       │
                ┌──────┴──────┐
                ▼             ▼
             Primary       Replicas
                │
                │
           Replication
                │
                ▼
          More read capacity

If primary becomes the bottleneck:

                ↓

             Sharding
                ↓
        ┌───────┼───────┐
        ▼       ▼       ▼
      Shard A Shard B Shard C
```

---

# What You Should Be Able to Explain Now

If I asked you in an interview:

> **"Why would you choose MySQL for an order system?"**

A strong answer should sound roughly like:

> "Orders are structured business data with relationships to users, products and payments. We need durable storage and transactional guarantees because operations such as creating an order and reserving inventory have correctness requirements. MySQL gives us a relational model, indexes for efficient access, transactions and concurrency control. As the system grows, we'd consider read replicas for read scaling, caching for frequently accessed data, and eventually sharding if the write volume or dataset exceeds what a single primary can handle. Each of those introduces tradeoffs such as replica lag, cache consistency and cross-shard complexity."

**That's the level we're aiming for.**

Not:

> "MySQL uses B+ trees and InnoDB."

Those details support the architectural reasoning; they aren't the answer themselves.

---

## Next: MySQL Replication & High Availability

We've now covered the core engine:

```text
Relational model
      ↓
Indexes
      ↓
Query execution
      ↓
Transactions
      ↓
Isolation
      ↓
Locking + MVCC
```

The next natural question is:

> **"What happens when the MySQL server itself fails, and how do we scale beyond one database instance?"**

We'll go into:

```text
Primary / Replica
Replication mechanics
Replication lag
Read-after-write problems
Failover
High availability
Automatic promotion
Split-brain
Replication vs backup
```

This is where MySQL starts becoming a genuine **distributed-system component**, rather than just a relational database.
