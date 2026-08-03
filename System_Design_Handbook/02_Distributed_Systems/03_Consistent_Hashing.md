# Module 2 — Chapter 3 — Consistent Hashing

> **Goal:** Understand why normal hashing fails in distributed systems, how consistent hashing solves the problem, and why it is widely used in scalable architectures.

---

# 1. The Problem

Imagine we have

4 database shards.

```text
Shard A

Shard B

Shard C

Shard D
```

We decide to store users using

```text
UserID % 4
```

Example

```text
User 1 → A

User 2 → B

User 3 → C

User 4 → D
```

Everything works.

---

Months later,

traffic grows.

We add

```text
Shard E
```

Now the formula changes.

```text
UserID % 5
```

Suddenly

```text
User 1

↓

Shard B
```

Instead of

```text
Shard A
```

User 2 moves.

User 3 moves.

User 4 moves.

Almost every key changes location.

---

# 2. Why Is This Bad?

Imagine

```text
500 million users
```

Every user suddenly belongs on a different machine.

You would have to move

hundreds of millions

of records.

The system becomes extremely expensive to rebalance.

---

# 3. The Big Idea

We need a way to add

or remove servers

without moving

every piece of data.

Only a small portion should move.

That's the entire purpose of

**Consistent Hashing.**

---

# 4. The Mental Shift

Instead of

```text
Key

↓

Machine
```

We introduce

a circle.

Think of a clock.

```text
         0
      /     \
   11         1

10             2

9               3

8               4

   7         5

      \     /

         6
```

This circle is called

the **Hash Ring**.

---

# 5. Placing Servers

Hash every server.

Example

```text
Server A

↓

Position 20

----------------

Server B

↓

Position 90

----------------

Server C

↓

Position 170

----------------

Server D

↓

Position 300
```

Now every server has a place

on the ring.

Imagine the ring:

```text
          A
      ●

                 B
                  ●


D ●

                ● C
```

---

# 6. Placing Keys

Now hash every key.

Example

```text
User123

↓

Position 110
```

Question

Who owns User123?

Rule:

Move **clockwise**

until you find the first server.

```text
User123

↓

110

↓

Server C
```

That's it.

Every key follows the same rule.

---

# 7. What Happens When a New Server Is Added?

Suppose we insert

Server E

between

B

and

C.

Old Ring

```text
A

↓

B

↓

C
```

New Ring

```text
A

↓

B

↓

E

↓

C
```

Which keys move?

Only the keys

between

B

and

E.

Everything else stays exactly where it was.

That's the magic.

Instead of moving

500 million keys,

maybe only

20 million move.

---

# 8. Removing a Server

Suppose

Server B crashes.

Only B's portion of the ring

is reassigned.

Everything else remains untouched.

Again,

minimal movement.

---

# 9. Why Is This So Powerful?

Compare.

### Normal Hashing

```text
Add one server

↓

Almost every key moves
```

---

### Consistent Hashing

```text
Add one server

↓

Only nearby keys move
```

This is why distributed systems love it.

---

# 10. Virtual Nodes

Here's a problem.

Suppose

Server A

lands here.

Server B

lands here.

```text
A

---------------------------

B
```

A now owns

almost half

the ring.

Unbalanced.

Instead,

pretend every physical server appears multiple times.

Example

```text
A1

A2

A3

B1

B2

B3

C1

C2

C3
```

These are called

**Virtual Nodes (VNodes).**

Now ownership is much more evenly distributed.

---

# 11. Why Virtual Nodes Matter

Suppose

Server C fails.

Without virtual nodes,

one server suddenly inherits

a huge chunk.

With virtual nodes,

its load is distributed across

many machines.

Much smoother.

---

# 12. Where Consistent Hashing Is Used

### Distributed Cache

Redis Cluster

Memcached

---

### Distributed Databases

Cassandra

DynamoDB

Riak

---

### Messaging

Kafka (conceptually similar partition distribution problems)

---

### Load Balancers

Distributing requests.

---

### CDNs

Finding which edge server should serve content.

---

# 13. Where It Isn't Needed

Suppose you have

one SQL database.

No distribution.

No problem.

Consistent Hashing only matters

when multiple machines

own different pieces of data.

---

# 14. Normal Hashing vs Consistent Hashing

| Normal Hashing                                | Consistent Hashing   |
| --------------------------------------------- | -------------------- |
| `% NumberOfServers`                           | Hash Ring            |
| Server count changes invalidate most mappings | Minimal remapping    |
| Large data movement                           | Small data movement  |
| Poor elasticity                               | Excellent elasticity |

---

# 15. Mental Model

Imagine a pizza.

Each slice belongs

to one delivery driver.

When a new driver joins,

you don't redraw

the whole pizza.

You simply give

that driver

one or two slices.

Only customers in those slices

change drivers.

Everyone else keeps the same one.

That's consistent hashing.

---

# 16. Tradeoffs

### Advantages

- Minimal data movement.
- Easy horizontal scaling.
- Handles server failures gracefully.
- Better load balancing (with virtual nodes).
- Widely applicable in distributed systems.

---

### Disadvantages

- More complex than simple hashing.
- Requires careful implementation.
- Still requires data migration for affected ranges.
- Virtual node management adds operational complexity.

---

# 17. Real-World Examples

### Cassandra

Uses consistent hashing to partition data across nodes in a ring.

---

### DynamoDB

Uses a partitioning strategy based on hashed partition keys to distribute data evenly across storage partitions.

---

### Redis Cluster

Distributes keys across multiple nodes (Redis Cluster uses **hash slots**, which solve a similar distribution problem but are not implemented as a classic consistent hash ring).

---

### Distributed Cache

Keys are assigned so adding cache servers doesn't invalidate the entire cache.

---

# 18. Common Interview Questions

- Why is normal hashing bad for distributed systems?
- What is consistent hashing?
- Why does it reduce data movement?
- What is a hash ring?
- What are virtual nodes?
- Why are virtual nodes important?
- Where is consistent hashing commonly used?

---

# 19. Connections

We've now solved

```text
Growing Database
```

But another problem appears.

Suppose

Shard A

fails.

Should the application stop working?

Or should another machine continue serving users?

That brings us to the next topic:

> **High Availability & Failover** (or, if we follow our roadmap, **CAP Theorem**, because it explains the fundamental tradeoffs behind availability and consistency).

---

# Key Takeaways

- Simple modulo hashing performs poorly when the number of servers changes because it remaps most keys.
- Consistent hashing arranges both **servers and keys on a logical hash ring**.
- A key is assigned to the **first server encountered clockwise** on the ring.
- Adding or removing a server only affects a small portion of the keys.
- **Virtual nodes** improve load balancing and make failures less disruptive.
- Consistent hashing is a foundational technique used throughout distributed systems.

---
