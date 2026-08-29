# Module 8 — Chapter 10: CQRS

## 1. Goal

**Separate the way a system changes data from the way it reads data when their requirements are fundamentally different.**

---

# 2. The Problem

Imagine a large e-commerce platform.

It has two very different workloads.

### Writes

Customers:

- place orders
- update addresses
- cancel orders
- add products
- change quantities

Maybe:

```text
10,000 writes / second
```

### Reads

But customers, sellers, search pages, recommendation systems, analytics, etc. may constantly ask:

```text
"Show me this product."

"Show me my orders."

"What's the current price?"

"How many products are available?"

```

Perhaps:

```text
1,000,000 reads / second
```

The system is therefore heavily read-oriented.

But here's the important part:

> **The best data model for safely writing data isn't necessarily the best data model for reading it.**

---

# 3. Why Existing Solutions Fail

Let's say we have one traditional model:

```text id="7thl6v"
              Application
              /         \
           Write        Read
             \           /
              \         /
               Database
```

That's perfectly reasonable for many systems.

But eventually the read and write requirements may diverge.

Suppose our normalized write model looks something like:

```text id="8es9s4"
Orders
Customers
Products
OrderItems
Payments
Addresses
```

To display an order history page, we might need:

```text id="5byvnr"
Orders
   ↓
JOIN Customers
   ↓
JOIN OrderItems
   ↓
JOIN Products
   ↓
JOIN Payments
```

That's fine at modest scale.

But now imagine millions of read requests.

The system is doing expensive work repeatedly just to construct a view that users frequently need.

We might want the read representation to look like:

```text id="w1gy0o"
OrderSummary
-------------------------
orderId
customerName
total
status
items
paymentStatus
shippingAddress
```

Now reading the page can be much simpler.

But this structure may not be ideal for transactional writes.

So we're facing a conflict:

```text id="r6jpzr"
Write requirements
       ≠
Read requirements
```

---

# 4. The Big Idea

> **CQRS separates the model used for commands (writes) from the model used for queries (reads).**

Instead of one model:

```text id="w3c5c9"
           Application
          /           \
      Write           Read
        \              /
         \            /
          Same Model
```

we create:

```text id="qk7q9v"
              Application
              /         \
             /           \
        Commands        Queries
           |               |
           v               v
      Write Model      Read Model
           |               |
           v               v
       Write Store      Read Store
```

The two sides can be optimized independently.

That's the core idea.

---

# 5. What Does "Command" Mean?

A **command** represents an instruction to change state.

Examples:

```text id="kik5zo"
CreateOrder
CancelOrder
UpdateAddress
ChangeQuantity
```

Commands answer:

> **"What should the system change?"**

---

# 6. What Does "Query" Mean?

A **query** asks for information without changing state.

Examples:

```text id="a9l7vi"
GetOrder
GetProduct
GetCustomerOrders
SearchProducts
```

Queries answer:

> **"What information do I need?"**

This gives us:

```text id="6g6qpr"
Command → change state

Query → read state
```

The important part of CQRS is not merely naming things "commands" and "queries."

It's that:

> **The models and potentially the storage systems behind them can be different.**

---

# 7. A Simple CQRS Architecture

```text id="3fqxgf"
                  Client
                 /      \
                /        \
        Command           Query
           |                |
           v                v
     Write Model        Read Model
           |                |
           v                v
       Write DB          Read DB
```

The write side is optimized for:

```text id="q8qpxm"
correctness
transactions
constraints
state changes
```

The read side is optimized for:

```text id="qsvx4d"
fast queries
specific views
high read throughput
easy retrieval
```

---

# 8. The Important Question

You might ask:

> "If we have two databases, how does the read database get updated?"

This is where CQRS connects directly to concepts we've already learned.

For example:

```text id="o1a2on"
Command
   ↓
Write Model
   ↓
Write DB
   ↓
Event
   ↓
Read Model
   ↓
Read DB
```

So when the write side changes:

```text id="w4j4yk"
OrderCreated
```

the read side receives that information and updates its representation.

This can produce:

```text id="e0y2n6"
Write DB
    |
    | event
    v
Read DB
```

And now we have an important consequence:

> **The read model may temporarily lag behind the write model.**

CQRS therefore often introduces **eventual consistency**.

---

# 9. Example: Creating an Order

Let's walk through the whole flow.

Customer sends:

```text id="3k2w3k"
CreateOrder
```

The command goes to the write side:

```text id="xid9sf"
Client
  |
  | CreateOrder
  v
Write Model
  |
  v
Write DB
```

The write succeeds:

```text id="v7dhqv"
Order created
```

The system produces an event:

```text id="9v34b1"
OrderCreated
```

The read side consumes it:

```text id="d8c1ae"
OrderCreated
      |
      v
Read Model
      |
      v
Read DB
```

The read representation may become:

```text id="y7h9r4"
OrderSummary {
    orderId: 123,
    customer: "Alice",
    total: 80000,
    status: "CREATED"
}
```

Now queries can efficiently read that model.

---

# 10. Why Not Just Use One Database?

Sometimes we should.

CQRS is **not** something every application needs.

Suppose:

```text id="xsy31u"
100 reads/sec
20 writes/sec
```

and our database handles it comfortably.

Then:

```text id="yld9mt"
Simple model
```

is usually better.

Introducing:

```text id="x0g9bq"
Write DB
Read DB
Events
Synchronization
Retries
```

would add complexity without meaningful benefit.

CQRS becomes interesting when the read and write sides have **meaningfully different requirements**.

---

# 11. Read Scaling

One major reason to introduce CQRS is that reads may vastly outnumber writes.

Suppose:

```text id="n8d2rm"
Writes = 10,000/sec
Reads  = 1,000,000/sec
```

We can scale the read side independently:

```text id="90dz44"
                    Write DB
                       |
                       | events
                       v
                  Read Model
                 /    |    \
                /     |     \
              R1      R2      R3
```

The write system doesn't necessarily need to scale in the same way.

This gives us:

> **Independent scaling of read and write workloads.**

---

# 12. Different Data Models

This is where CQRS becomes particularly powerful.

Suppose the write side needs a normalized representation:

```text id="z4p0mq"
Customers
Orders
OrderItems
Products
```

The read side might instead maintain:

```text id="qz8yif"
CustomerOrderView
----------------------------
customerId
customerName
orderId
items
total
paymentStatus
shippingStatus
```

The read model is deliberately shaped around the queries we need.

Instead of repeatedly performing:

```text id="v1t72j"
JOIN
JOIN
JOIN
JOIN
```

we can read a representation that's already assembled.

This is sometimes called a **read projection**.

---

# 13. Projection

A projection is essentially:

> **A read-optimized representation derived from the underlying data/events.**

For example:

```text id="9yqucb"
OrderCreated
OrderPaid
OrderShipped
     |
     v
Order History Projection
```

produces:

```text id="0t83uj"
{
  orderId: 123,
  status: "SHIPPED",
  paid: true,
  shipped: true
}
```

The projection doesn't necessarily represent the authoritative state.

It represents:

> **The shape needed for a particular query.**

---

# 14. Multiple Read Models

This is one of the most interesting aspects of CQRS.

We don't necessarily need one read model.

We could have:

```text id="w4u1n4"
                 Write Model
                      |
                    Events
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
       Orders       Search      Analytics
       View         View          View
```

Each projection serves a different purpose.

For example:

### Order history

```text id="c4j03h"
OrderSummary
```

### Product search

```text id="g8x9fq"
ProductSearchView
```

### Analytics

```text id="f4k1p2"
SalesByDay
```

The same underlying business events can drive different read representations.

---

# 15. The Cost: Eventual Consistency

Suppose:

```text id="y5p6c9"
Client
  |
  | Change address
  v
Write DB
  |
  ✓
```

Immediately after:

```text id="7i0u0q"
Write DB → new address
Read DB  → old address
```

The read model hasn't caught up yet.

A moment later:

```text id="t1j4xw"
Read DB → new address
```

This is eventual consistency.

So CQRS often introduces a period where:

```text id="s6t5o0"
Write state ≠ Read state
```

That is an important tradeoff.

---

# 16. Is This a Problem?

It depends on the business requirement.

Suppose a user updates their profile:

```text id="6e2y9u"
Write succeeds
```

and the profile page takes:

```text id="4fs9o9"
100 ms
```

to show the updated value.

Probably acceptable.

But suppose we're dealing with:

```text id="d4p0j2"
Account balance
```

and the user transfers ₹10,000.

If the read model temporarily shows the old balance:

```text id="i3f1zy"
Actual balance = ₹40,000
Displayed = ₹50,000
```

that may be unacceptable depending on the system.

Therefore:

> **CQRS is a consistency tradeoff, not merely a performance technique.**

---

# 17. CQRS Does Not Require Two Databases

This is a common interview trap.

CQRS fundamentally means:

```text id="k8gh8s"
separate command model
from
query model
```

It does **not** strictly require:

```text id="6f2f4r"
two physical databases
```

You could have:

```text id="u1y1wx"
             Application
             /         \
        Command       Query
           |             |
      Write Model     Read Model
           \             /
             Same DB
```

The models are conceptually separated even if they happen to use the same storage system.

At larger scale, however, separate stores often become useful.

---

# 18. CQRS Does Not Require Event Sourcing

Another important distinction.

These are related concepts:

```text id="m0yq4d"
CQRS
Event Sourcing
```

but they are **not the same thing**.

CQRS:

```text id="8s7r2h"
Separate read and write models.
```

Event Sourcing:

```text id="0x7z8e"
Store state changes as events
rather than only storing current state.
```

You can have:

```text id="7c3s7k"
CQRS without Event Sourcing
```

and:

```text id="o4x2yo"
Event Sourcing without full CQRS
```

Although they are often used together.

---

# 19. CQRS Without Event Sourcing

For example:

```text id="p5e1cz"
Write DB
   |
   | change notification
   v
Read DB
```

The write database can still store ordinary current-state data.

For example:

```text id="ry4w2x"
Order
----------------
id = 123
status = SHIPPED
total = 80000
```

We don't need to retain every historical state transition to use CQRS.

---

# 20. CQRS + Event Sourcing

If we combine them:

```text id="g7s0z7"
Command
   ↓
Write Model
   ↓
Event Store
   ↓
Events
   ↓
Projections
   ↓
Read Models
```

Now the architecture becomes especially powerful for systems where historical events matter.

For example:

```text id="9vcm0r"
OrderCreated
PaymentCompleted
InventoryReserved
OrderShipped
```

The current state can be reconstructed from the events.

We'll study **Event Sourcing** in the next chapter.

---

# 21. Where CQRS Helps

### 1. Read-heavy systems

When:

```text id="n9d4dd"
Reads >>> Writes
```

we can scale the read side independently.

---

### 2. Complex queries

If queries require expensive joins or transformations:

```text id="qj8r0p"
Write model
     ↓
Projection
     ↓
Read-optimized model
```

can make reads much simpler.

---

### 3. Different scaling requirements

For example:

```text id="2z6xme"
Write side → 10 servers
Read side  → 100 servers
```

---

### 4. Multiple views of the same data

```text id="m6k0r3"
One source
   ↓
+--------+--------+
|        |        |
Orders Search Analytics
```

---

### 5. Complex domains

In systems where business operations and read requirements are fundamentally different, separating the models can make the architecture cleaner.

---

# 22. Where It Doesn't Help

### Simple CRUD applications

If you have:

```text id="t9n0j8"
Create
Read
Update
Delete
```

and a single database handles everything comfortably, CQRS may simply add unnecessary complexity.

---

### When strong immediate consistency is essential

If users must always see the latest authoritative state immediately, asynchronous read projections may be problematic.

---

### Low read/write scale

If the system has modest traffic:

```text id="0qf4lw"
100 requests/sec
```

don't introduce an elaborate architecture just to prepare for hypothetical scale.

---

### Teams that cannot handle the operational complexity

CQRS can introduce:

- synchronization
- projection failures
- duplicate events
- replay
- lag monitoring
- consistency issues

The architecture has to be worth that complexity.

---

# 23. Tradeoffs

## Advantages

### Independent scaling

```text id="f3y0m5"
Read scale ≠ Write scale
```

### Query optimization

Read models can be designed specifically around query patterns.

### Multiple representations

Different consumers can have different projections.

### Separation of responsibilities

The write model focuses on correctness and state changes.

The read model focuses on retrieval.

---

## Disadvantages

### Eventual consistency

Read models may temporarily lag.

### More infrastructure

Potentially:

```text id="c1i1hk"
Write DB
Read DB
Event mechanism
Projection workers
Monitoring
```

### Synchronization complexity

You now need to ensure the read side catches up.

### Debugging becomes harder

A user may ask:

> "Why does the read model still show the old value?"

You now need to trace the whole propagation path.

### Data duplication

The same business information may exist in multiple representations.

---

# 24. Common Interview Questions

## Q1. What is CQRS?

CQRS separates the model used to modify state from the model used to query state, allowing each side to be optimized independently.

---

## Q2. Does CQRS require two databases?

**No.**

CQRS is a separation of command and query models.

Separate physical stores are optional.

---

## Q3. Does CQRS require Event Sourcing?

**No.**

They are independent patterns that are often combined.

---

## Q4. Why would we use CQRS?

Common reasons include:

- read/write workload asymmetry
- complex queries
- independent scaling
- different data models
- multiple specialized read views

---

## Q5. What's the biggest consistency issue?

The read model can lag behind the write model.

For example:

```text id="g0n8h5"
Write:
status = SHIPPED

Read:
status = PROCESSING
```

temporarily.

---

## Q6. How does the read model stay synchronized?

Typically through some propagation mechanism:

```text id="u3z8x1"
Write
  ↓
Change/Event
  ↓
Projection
  ↓
Read Model
```

The exact technology isn't important at this stage.

---

## Q7. Why not simply optimize the original database?

Sometimes that's exactly what you should do.

Before introducing CQRS, consider:

- indexes
- caching
- query optimization
- read replicas
- schema improvements
- partitioning

CQRS is useful when the underlying **modeling and workload requirements** themselves have diverged.

---

# 25. Before vs After Architecture

### Traditional model

```text id="azc4s2"
                 Client
                /      \
               /        \
            Write       Read
               \         /
                \       /
                Database
```

One model serves both purposes.

---

### CQRS

```text id="l1m0ck"
                  Client
                 /      \
                /        \
          Commands       Queries
             |              |
             v              v
        Write Model     Read Model
             |              |
             v              v
         Write DB        Read DB
             |
             | changes/events
             v
        Read Model
```

Now each side can evolve independently.

---

# 26. The Deeper Connection

Look at what we've learned so far.

### Saga

We separated:

```text id="jv8z5q"
one giant transaction
```

into:

```text id="m0x0d5"
multiple local transactions
```

because distributed atomicity was too expensive.

### CQRS

We separate:

```text id="f2e4o6"
one data model
```

into:

```text id="x2m4m5"
write model + read model
```

because one model may not efficiently serve both workloads.

The recurring system-design principle is:

> **When one abstraction is being forced to satisfy fundamentally different requirements, separating the responsibilities can make the system easier to scale and reason about.**

---

# 27. Connections

The next chapter is:

## Chapter 11 — Event Sourcing

CQRS gave us:

```text id="h7t4wq"
Write Model
     ↓
Read Model
```

But we still haven't answered an interesting question:

> **What exactly should the write side store?**

Traditional systems store current state:

```text id="q4v6cs"
Order:
status = SHIPPED
```

But what if instead we stored the sequence of things that happened?

```text id="2c7d9x"
OrderCreated
PaymentCompleted
InventoryReserved
OrderShipped
```

Then the current state can be derived from those events.

That leads us to:

```text id="8a2f4v"
             Commands
                |
                v
           Event Store
                |
       +--------+--------+
       |        |        |
       v        v        v
   Projection Projection Projection
       |        |        |
       v        v        v
    Read DB   Read DB   Read DB
```

This is **Event Sourcing**.

And now the relationship between the two patterns becomes important:

```text id="n0u4f7"
CQRS
  ↓
separates reads from writes

Event Sourcing
  ↓
changes how writes/state history are stored
```

We'll examine why storing **events instead of just current state** can be incredibly powerful—and why it also introduces significant complexity.

---

# 28. Key Takeaways

1. **CQRS separates command/write models from query/read models.**
2. Commands change state; queries retrieve state.
3. The write and read sides can be optimized independently.
4. CQRS does **not** require two physical databases.
5. CQRS does **not** require Event Sourcing.
6. Read models can be projections specifically designed for particular queries.
7. Multiple read models can be derived from the same underlying changes.
8. CQRS often introduces **eventual consistency** between write and read models.
9. It is useful when read/write requirements genuinely diverge—not merely because the application is large.
10. CQRS trades architectural simplicity for independent scaling and specialized data models.

### One sentence to remember

> **CQRS says that the model optimized for safely changing state doesn't have to be the same model optimized for retrieving it.**
