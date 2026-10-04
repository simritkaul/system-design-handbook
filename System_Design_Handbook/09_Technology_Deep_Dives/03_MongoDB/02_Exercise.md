Absolutely. Let's continue from the MongoDB exercise and close the loop before moving deeper.

### The food-delivery example

The natural answer is:

```text
Order
 ├── items[]          ← Embed
 ├── deliveryAddress  ← Embed
 ├── pricing          ← Embed
 └── customerId       ← Reference
```

Why?

### 1. Order items → Embed

```text
Order
{
    orderId: 123,
    items: [
        { productId: 10, quantity: 2 },
        { productId: 20, quantity: 1 }
    ]
}
```

The items are:

- owned by this order
- bounded in size
- normally retrieved with the order
- not independently shared between orders

So they form part of the **Order aggregate**.

---

### 2. Delivery address → Embed

This is particularly important.

You might initially think:

> "We already have the customer's address, so reference it."

But that's dangerous.

Suppose the customer changes their address tomorrow.

The historical order must still say:

```text
Order #123
Delivery:
221B Baker Street
...
```

It shouldn't suddenly change because the customer's current address changed.

So the order should contain a **snapshot**:

```text
deliveryAddress: {
    street: "...",
    city: "...",
    postalCode: "..."
}
```

This is a great example of how **business semantics**, not just database relationships, determine document design.

---

### 3. Pricing → Embed

Same reasoning:

```text
pricing: {
    subtotal: 500,
    deliveryFee: 40,
    discount: 50,
    total: 490
}
```

The price is a historical fact about **this particular order**.

If the restaurant changes its menu price tomorrow, Order #123 shouldn't recalculate itself.

So again:

> **Embed the snapshot belonging to the aggregate.**

---

### 4. Customer → Orders → Reference

This is where embedding breaks down.

Imagine:

```text
Customer
{
    orders: [
        ...
        millions of orders
    ]
}
```

That's an unbounded relationship.

Instead:

```text
Customer
{
    customerId: 42,
    ...
}

Order
{
    orderId: 123,
    customerId: 42,
    ...
}
```

Now:

```text
GET /customers/42/orders
          ↓
query Orders by customerId
```

Much healthier.

---

# The Deeper Lesson

Notice what we just did.

We didn't memorize:

> "One-to-many → reference."

That's too simplistic.

We asked:

```text
Is it owned by the aggregate?
        ↓
Is it bounded?
        ↓
Is it usually retrieved together?
        ↓
Does it have an independent lifecycle?
        ↓
Does it need to remain historically consistent?
```

That is **document modeling**.

And this is exactly why MongoDB requires more thought than simply saying:

> "It's a NoSQL database that stores JSON."

---

# One More Important Connection

There's a subtle distinction here between:

### Current state

```text
Customer.address
```

and

### Historical state

```text
Order.deliveryAddress
```

The latter is intentionally duplicated.

In a normalized relational model, we often try to minimize duplication.

In a document model, **controlled duplication can be desirable** when it makes the aggregate self-contained and preserves historical meaning.

That's one of the biggest mindset shifts from relational modeling to document modeling.

---

# MongoDB — What You Should Now Know

At this point, your HLD mental model should look like:

```text
                 MongoDB
                    │
        ┌───────────┴───────────┐
        │                       │
   Document Model          Flexible Schema
        │                       │
        ▼                       ▼
   Model Aggregates       Variable Structures
        │
   ┌────┴─────┐
   ▼          ▼
Embed      Reference
   │          │
Bounded      Independent /
Owned        Unbounded
Data         Data
        │
        ▼
Access Patterns
        │
        ▼
   Sharding / Scaling
```

And the core tradeoff:

> **MongoDB gives you a natural way to model application aggregates and flexible data, but you become responsible for choosing good document boundaries and access patterns.**

That is the part worth carrying into interviews.

---

## Next: MongoDB → Cassandra

Now we've got an interesting progression:

```text
MySQL
  ↓
Relational model

PostgreSQL
  ↓
Relational model + richer/extensible capabilities

MongoDB
  ↓
Document model + aggregate-oriented access
```

But imagine the workload gets dramatically larger:

```text
Millions → Billions → Trillions of records

+
Very high write throughput

+
Massive horizontal scale

+
Predictable access patterns

+
We care more about availability and scale
  than complex relational queries
```

Now we encounter a different problem:

> **What if the database's primary job is to distribute enormous amounts of data and writes across many machines predictably?**

That leads us to:

# **Chapter 4 — Cassandra**

And Cassandra is going to be particularly useful because it forces us to revisit several things you've already learned:

**partitioning, replication, consistency, CAP, quorum reads/writes, hot partitions, and data modeling around access patterns.**
