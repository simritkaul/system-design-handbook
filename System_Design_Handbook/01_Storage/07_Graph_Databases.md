# Chapter 7 — Graph Databases

> **Goal:** Understand what graph databases are, why they exist, what problems they solve, and when they are a better choice than SQL, Key-Value, Document, or Wide Column databases.

---

# 1. The Problem

Imagine you're building LinkedIn.

You have users.

```text
Alice

Bob

Charlie

David
```

Now each user connects with others.

```text
Alice ↔ Bob

Bob ↔ Charlie

Charlie ↔ David

Alice ↔ David
```

Now someone asks

> "Show me Alice's friends."

Easy.

Then they ask

> "Show me friends of Alice's friends."

Still okay.

Then they ask

> "Find the shortest connection between Alice and Elon Musk."

Or

> "Recommend people Alice may know."

Or

> "Who are the mutual friends between Alice and Bob?"

Suddenly...

the relationships become the important part.

Not the users.

---

# 2. Why SQL Starts Struggling

Suppose we store friendships in SQL.

Users

| ID  | Name    |
| --- | ------- |
| 1   | Alice   |
| 2   | Bob     |
| 3   | Charlie |

Friendships

| User1 | User2 |
| ----- | ----- |
| 1     | 2     |
| 2     | 3     |
| 1     | 4     |

Finding

```text
Alice

↓

Friends

↓

Friends

↓

Friends
```

requires repeated joins.

As relationships become deeper,

queries become increasingly complex.

---

# 3. The Big Idea

Instead of storing relationships separately,

make relationships a first-class citizen.

Represent the world as

```text
Node

↓

Relationship

↓

Node

↓

Relationship
```

Everything becomes a graph.

---

# 4. What Is a Graph?

A graph has only two things.

### Nodes

The objects.

Examples

```text
Person

City

Company

Movie

Product
```

---

### Edges

The relationships.

Examples

```text
FRIEND_OF

WORKS_AT

BOUGHT

LIKED

FOLLOWS

LIVES_IN
```

That's it.

---

# 5. A Real Example

Imagine LinkedIn.

```text
Alice

↓

WORKS_AT

↓

Google

↓

LOCATED_IN

↓

California
```

Everything is connected.

The database stores both

the objects

and

their relationships.

---

# 6. Traversal

This is the superpower.

Instead of

joining table after table,

you simply

walk through the graph.

Example

```text
Alice

↓

Friends

↓

Friends

↓

Friends
```

This is called **graph traversal**.

Graph databases are optimized for this.

---

# 7. Why Is This Fast?

Suppose Alice has

500 friends.

Each friend has

500 friends.

In SQL,

finding connections often requires joining large tables repeatedly.

In a graph database,

relationships are stored directly between nodes.

The database can follow those relationships without reconstructing them through joins.

That's why traversals are efficient.

---

# 8. Where Graph Databases Shine

### Social Networks

Facebook

LinkedIn

Instagram

Connections are everything.

---

### Recommendation Engines

Amazon

Netflix

Spotify

People who bought

↓

Also bought

---

### Fraud Detection

Person

↓

Credit Card

↓

Merchant

↓

Bank Account

↓

Another Person

Relationships reveal fraud patterns.

---

### Network Topology

Servers

Routers

Switches

Connected together.

---

### Knowledge Graphs

Google Search

Wikipedia

Entities connected by meaning.

---

### Supply Chains

Factory

↓

Warehouse

↓

Truck

↓

Store

Everything is connected.

---

# 9. Where They Perform Poorly

Suppose you're building Payroll.

Need

Employee

Salary

Attendance

Tax

No complex relationships.

SQL is simpler.

---

Suppose you're storing billions of sensor readings.

Graph databases provide little advantage.

Wide Column databases are a better fit.

---

# 10. Graph DB vs SQL

| SQL                      | Graph Database                 |
| ------------------------ | ------------------------------ |
| Tables                   | Nodes                          |
| Foreign Keys             | Relationships                  |
| Joins                    | Graph Traversal                |
| Structured business data | Highly connected data          |
| Great for transactions   | Great for relationship queries |

---

# 11. Graph DB vs Document DB

Document DB

Stores one object.

```json
User

Orders

Preferences
```

Everything about one object.

---

Graph DB

Stores

```text
User

↓

Friend

↓

Company

↓

City

↓

Movie
```

The relationships are the important part.

---

# 12. Graph DB vs Wide Column

Wide Column

Optimizes

Huge datasets

High writes

Known queries.

---

Graph DB

Optimizes

Unknown relationship queries.

Very different problem.

---

# 13. Horizontal Scaling

This is actually one of the hardest problems for graph databases.

Why?

Imagine

```text
Alice

↓

Bob

↓

Charlie

↓

David
```

If Alice and Bob are on Machine A,

but Charlie is on Machine B,

graph traversals now cross machines.

Unlike key-value or document databases, related data is often highly interconnected, making partitioning much more challenging.

This is one reason graph databases are typically chosen for workloads where relationship queries matter more than massive horizontal scale.

---

# 14. Popular Technologies

Examples include:

- Neo4j
- Amazon Neptune
- TigerGraph
- JanusGraph

We'll study them individually later.

---

# 15. Mental Model

Imagine Google Maps.

Cities

↓

Roads

↓

Cities

↓

Roads

↓

Cities

You're not interested in the cities alone.

You're interested in

> **How they're connected.**

A graph database is built exactly for this kind of problem.

---

# 16. Tradeoffs

### Advantages

- Excellent for relationship-heavy queries
- Efficient graph traversals
- Natural way to model connected data
- Easier to express complex relationship queries

---

### Disadvantages

- Not ideal for traditional business applications
- Horizontal scaling is difficult
- Less efficient for simple CRUD workloads
- Specialized query languages and modeling approaches

---

# 17. Real-World Examples

### LinkedIn

Professional connections.

---

### Facebook

Friend graph.

---

### Google Knowledge Graph

People

Places

Movies

Concepts

Connected together.

---

### Fraud Detection

Banks detect suspicious transaction networks.

---

### Recommendation Systems

Netflix

Spotify

Amazon

Recommendations based on interconnected users, items, and behaviors.

---

# 18. Common Interview Questions

- Why use a graph database?
- What kinds of problems are graph databases designed for?
- Why are joins expensive for highly connected data?
- What is graph traversal?
- Why are graph databases popular for recommendation systems?
- Why is horizontal scaling harder than in key-value databases?
- When would SQL still be the better choice?

---

# 19. Connections

You've now covered the major database families:

- **SQL** → Structured, relational, transactional data.
- **Key-Value** → Fast lookups by key.
- **Document** → Flexible, self-contained objects.
- **Wide Column** → Massive write throughput and predictable access patterns.
- **Graph** → Relationship-centric data and graph traversals.

Together, these form the core storage models you'll encounter in modern system design.

---

# Key Takeaways

- Graph databases model data as **nodes and relationships** rather than tables and joins.
- They excel when **relationships are the primary focus** of the application.
- Traversing connected data is much more efficient than repeatedly joining relational tables.
- Social networks, recommendation engines, fraud detection, and knowledge graphs are classic use cases.
- Their biggest tradeoff is scalability: highly interconnected data is inherently harder to partition across many machines.

---

## I think we've now reached a milestone

If you step back, you'll notice we've built a decision framework rather than a list of technologies:

- **Need transactions and structured business data?** → SQL
- **Know the key and need blazing-fast access?** → Key-Value
- **Need flexible, evolving objects?** → Document
- **Need enormous write throughput?** → Wide Column
- **Need to navigate complex relationships?** → Graph

That's a much stronger foundation than memorizing "MongoDB is for X" or "Cassandra is for Y." When we eventually study MongoDB, Cassandra, Redis, or Neo4j, they'll feel like natural implementations of concepts you already understand, rather than isolated technologies.
