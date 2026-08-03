# Chapter 1 — Storage Fundamentals

## Why do we need a database?

Imagine building a simple Notes application.

A user creates a note.

```
Title: Grocery List
Content:
Milk
Bread
Eggs
```

Where does this data go?

If you store it only in your application's memory:

```
App Memory

User A
User B
User C
```

everything disappears when the server restarts.

So we need **persistent storage**.

A database is simply a system whose job is to **store data safely and retrieve it efficiently**.

---

# The four questions every database tries to answer

Whenever someone invents a new database, they're trying to improve one or more of these.

## 1. How do I store data?

Examples

- Tables
- Documents
- Key-Value pairs
- Graphs
- Wide Columns

Different databases organize data differently.

---

## 2. How do I find data quickly?

Imagine a table with

```
1 million users
```

Searching one by one would be far too slow.

Databases build indexes and use optimized storage structures to locate data efficiently.

---

## 3. How do I store huge amounts of data?

One machine has limits.

Eventually you'll need multiple machines.

This introduces concepts like

- Replication
- Partitioning
- Sharding

---

## 4. How do I survive failures?

Machines crash.

Disks fail.

Networks go down.

Good databases ensure your data isn't lost when that happens.

---

# The four properties we care about

Whenever we choose a storage technology, we're balancing these.

### Performance

How quickly can we read and write data?

---

### Scalability

Can the database handle

- 100 users?
- 1 million users?
- 100 million users?

---

### Reliability

Does the data remain safe if a machine crashes?

---

### Consistency

If one user updates their profile,

when another user reads it,

do they immediately see the latest value?

Or is a small delay acceptable?

---

# Not all applications need the same database

Consider these examples.

### Banking

Requirements

- Never lose money
- Strong transactions
- Correct balances

Correctness matters more than raw speed.

---

### Instagram Feed

Requirements

- Millions of reads
- Fast loading
- A slight delay is acceptable

Speed matters more than perfect consistency.

---

### Google Search

Requirements

- Search billions of documents
- Handle spelling mistakes
- Rank relevant results

A traditional relational database isn't designed for this.

---

### WhatsApp

Requirements

- Billions of messages
- Real-time delivery
- High availability

Different requirements lead to different architectural choices.

---

# Why are there so many databases?

A common beginner question is:

> "Why don't we just use MySQL for everything?"

Because optimizing for one thing usually means compromising another.

Examples:

- A relational database provides transactions and joins but can become harder to scale horizontally.
- A key-value store offers extremely fast lookups but supports much simpler queries.
- A search engine excels at full-text search but isn't meant to be the primary source of truth.
- An object store is ideal for large files but isn't suitable for relational queries.

There is no universally "best" database—only the one that best matches your application's requirements.

---

# Categories of databases

We'll study each of these separately.

| Category         | Best For                               | Examples          |
| ---------------- | -------------------------------------- | ----------------- |
| Relational (SQL) | Structured business data, transactions | MySQL, PostgreSQL |
| Key-Value        | Fast lookups, caching, sessions        | Redis             |
| Document         | Flexible schemas                       | MongoDB           |
| Wide Column      | Massive write throughput               | Cassandra         |
| Graph            | Relationships                          | Neo4j             |
| Search Engine    | Full-text search                       | Elasticsearch     |
| Object Storage   | Files, videos, images                  | Amazon S3         |

---

# Mental Model

Think of databases like different kinds of storage in a city.

- **SQL Database** → A well-organized filing cabinet with labeled folders and strict rules.
- **Redis** → Your desk drawer. Extremely fast to access, but limited in size.
- **MongoDB** → A collection of folders where each file can have a different structure.
- **Cassandra** → A huge warehouse built to accept deliveries continuously.
- **Elasticsearch** → The library's search catalog that helps you find books instantly.
- **S3** → A massive storage warehouse for boxes (photos, videos, backups, documents).

Each solves a different storage problem.

---

# Common Interview Questions

- Why can't one database solve every problem?
- How do you choose the right database?
- What factors influence database selection?
- What is the difference between persistent storage and in-memory storage?
- Why do modern systems often use multiple storage technologies instead of just one?

---

# Key Takeaways

- A database exists to store and retrieve data efficiently and reliably.
- Different applications have different storage requirements.
- There is no universally best database.
- Every database is a tradeoff between performance, scalability, reliability, consistency, and flexibility.
- In system design, your job isn't to memorize technologies—it's to understand the problem first, then choose the technology whose tradeoffs best fit that problem.

---

## What's next?

The next chapter should be **SQL Databases**, where we'll cover:

- Why relational databases became the standard
- Tables, rows, columns, and schemas
- Primary keys and foreign keys
- ACID transactions
- Indexes (at a high level)
- Strengths and weaknesses
- Why MySQL and PostgreSQL are still the default choice for many production systems
- When SQL is the right choice—and when it isn't

From there, moving into NoSQL and then Redis will feel much more intuitive because you'll already understand the problems they're trying to solve.
