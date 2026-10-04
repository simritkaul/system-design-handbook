# MySQL — Part 2: How MySQL Finds Your Data

We established that MySQL's value isn't merely "it stores tables."

A major reason it can serve large workloads is:

> **It can avoid examining every row when answering a query.**

The mechanism that makes this possible is the **index**.

Today we'll go one level deeper:

```text
Index
  ↓
B+ Tree
  ↓
Clustered vs Secondary Index
  ↓
Composite Index
  ↓
How a query uses an index
  ↓
Why queries still become slow
```

The goal isn't to memorize B+ tree internals.

The goal is to be able to look at an HLD requirement and reason:

> "What access pattern do we have, and what kind of index would make that access cheap?"

---

# 1. Start With the Problem

Imagine our `orders` table has:

```text
500 million rows
```

And the application constantly asks:

```sql
SELECT *
FROM orders
WHERE user_id = 12345;
```

Without an index on `user_id`, the database could conceptually have to do:

```text
Row 1   → Is user_id 12345?
Row 2   → Is user_id 12345?
Row 3   → Is user_id 12345?
...
Row 500,000,000 → Is user_id 12345?
```

That's a **table scan**.

For a huge table, that's obviously expensive.

We want:

```text
Query
  ↓
Index
  ↓
Find relevant location
  ↓
Fetch rows
```

So the fundamental purpose of an index is:

> **Reduce the amount of data the database has to inspect.**

---

# 2. Why Not Just Use a Hash Table?

You might immediately think:

> "We need `user_id → rows`. Why not a hash table?"

That's a good question.

A hash-based structure is excellent for:

```text
WHERE user_id = 12345
```

because we want exact lookup.

But databases need much more than equality.

For example:

```sql
WHERE user_id >= 10000
```

or:

```sql
WHERE created_at BETWEEN
      '2026-01-01'
      AND '2026-02-01'
```

or:

```sql
ORDER BY created_at
```

We need **ordered access**.

That's why tree-based indexes are so useful.

---

# 3. B+ Tree — The Mental Model

Don't worry about the exact implementation.

Think of an ordered tree:

```text
                 [50]
               /      \
         [10,20,30]   [60,70,80]
```

The tree allows the database to repeatedly narrow down the search space.

Instead of:

```text
500 million rows
```

we might do something conceptually like:

```text
500M
 ↓
100M
 ↓
1M
 ↓
10K
 ↓
100
 ↓
10
```

The important property is:

> **The database doesn't need to inspect every row to find the relevant region.**

---

# 4. Why B+ Trees Are Particularly Useful for Databases

There's another problem.

Databases don't have infinite RAM.

Eventually, data must live on storage.

And storage access is much slower than memory access.

So databases want an index structure that minimizes expensive storage operations.

A B+ tree is designed to have:

```text
High branching factor
+
Shallow depth
```

Conceptually:

```text
                 Root
              /    |    \
             /     |     \
          Node    Node    Node
         / | \    / | \    / | \
        ...       ...       ...
```

Rather than:

```text
Root
 |
 Node
 |
 Node
 |
 Node
 |
 Node
```

A wide tree means fewer levels.

That matters because:

```text
fewer levels
    ↓
fewer storage/page accesses
    ↓
faster lookup
```

---

# 5. B+ Tree vs Binary Search Tree

You may have learned binary search trees in DSA.

A binary tree looks like:

```text
             50
           /    \
         25      75
        /  \    /  \
      10   30  60   90
```

A B+ tree is much wider:

```text
             [25 | 50 | 75]
           /      |      |    \
        ...      ...    ...   ...
```

Why?

Because database storage works in **pages/blocks**, not individual objects like your Java heap.

We want each database page to contain many useful index entries.

So a wide tree is much more storage-efficient.

You don't need to remember the exact number of children.

Remember:

> **Database indexes use wide, shallow tree structures because storage is accessed in pages.**

---

# 6. Why Is It Called a B+ Tree?

For our purposes, the distinction worth remembering is:

```text
Internal nodes
    ↓
mostly guide the search

Leaf nodes
    ↓
contain the actual indexed entries
```

And the leaves are linked in order:

```text
[10,20,30] → [40,50,60] → [70,80,90]
```

That linked ordering is extremely useful.

Because now a range query can efficiently move through adjacent entries.

For example:

```sql
WHERE user_id BETWEEN 100 AND 200
```

Instead of repeatedly searching from the root, we can:

```text
Find 100
   ↓
Walk through ordered leaves
   ↓
Stop after 200
```

This is one reason B+ trees are so useful for databases.

---

# 7. Equality AND Range Queries

Consider:

```sql
WHERE id = 500
```

The index can navigate directly toward 500.

But also:

```sql
WHERE id >= 500
AND id < 600
```

The ordered structure makes this efficient too.

So:

```text
B+ Tree
   │
   ├── equality lookup
   ├── range lookup
   └── ordered traversal
```

This is much more versatile than a pure hash structure.

---

# 8. Now Something Very Important: The Index Doesn't Necessarily Contain the Whole Row

Suppose:

```text
users
--------------------------------
id
name
email
age
address
created_at
```

We create:

```text
INDEX(email)
```

The index primarily organizes:

```text
email → associated row location
```

Conceptually:

```text
alice@example.com → row X
bob@example.com   → row Y
charlie@example.com → row Z
```

So when we execute:

```sql
SELECT *
FROM users
WHERE email = 'alice@example.com';
```

MySQL can:

```text
1. Search email index
2. Find corresponding row
3. Fetch complete row
```

This leads to an important distinction.

---

# 9. Clustered Index

With InnoDB, the table's primary key is associated with the **clustered index**.

The useful mental model is:

```text
Primary-key index
        ↓
Rows are organized around primary-key order
```

Conceptually:

```text
Primary Key Index

        100
       /   \
     50     150
    /  \    /  \
  rows ... rows ...
```

The leaf level contains the row data associated with those primary keys.

So if we do:

```sql
SELECT *
FROM users
WHERE id = 123;
```

and `id` is the primary key, MySQL can navigate the clustered index and reach the row.

This is one reason primary-key lookups are so fundamental.

---

# 10. Secondary Indexes

Now create:

```text
INDEX(email)
```

That's a **secondary index**.

Conceptually:

```text
Secondary Index

alice@example.com → primary_key = 123
bob@example.com   → primary_key = 456
```

Notice what happened.

The secondary index doesn't need to duplicate the entire row.

It can point toward the primary key.

So:

```text
Query
 ↓
Secondary index
 ↓
Primary key
 ↓
Clustered index
 ↓
Actual row
```

There can therefore be an additional lookup.

---

# 11. Why This Matters

Suppose:

```sql
SELECT email
FROM users
WHERE email = 'alice@example.com';
```

The index may already contain everything needed to answer the query.

But:

```sql
SELECT *
FROM users
WHERE email = 'alice@example.com';
```

may require retrieving the rest of the row.

This distinction becomes important when queries are large and frequent.

It leads to the idea of a **covering index**.

---

# 12. Covering Index

Suppose our query is:

```sql
SELECT user_id, created_at
FROM orders
WHERE user_id = 123;
```

And we have an index that contains:

```text
(user_id, created_at)
```

The database may be able to answer the query entirely from the index without fetching the full table row.

The index "covers" the query.

Conceptually:

```text
Normal:

Index
 ↓
Primary key
 ↓
Table row
 ↓
Result


Covering index:

Index
 ↓
Result
```

That's potentially much cheaper.

But don't take this to mean:

> "Let's put every column into every index."

Remember the earlier tradeoff:

```text
Larger index
    ↓
More storage
+
More memory pressure
+
More write work
```

Again, optimization is workload-dependent.

---

# 13. Composite Indexes

Now we reach one of the most important MySQL interview topics.

Suppose our application frequently runs:

```sql
SELECT *
FROM orders
WHERE user_id = 123
ORDER BY created_at DESC;
```

We might create:

```text
INDEX(user_id, created_at)
```

This is a **composite index**.

The ordering is important.

Think of it as:

```text
(user_id, created_at)
```

not simply:

```text
(user_id)
+
(created_at)
```

The index is ordered first by `user_id`, and then by `created_at` within each user.

Conceptually:

```text
user 1
  ├── Jan
  ├── Feb
  └── Mar

user 2
  ├── Jan
  ├── Feb
  └── Mar

user 3
  ├── Jan
  ├── Feb
  └── Mar
```

That matches our access pattern extremely well.

---

# 14. The Leftmost Prefix Idea

Here's where interviewers like to test understanding.

Given:

```text
INDEX(user_id, created_at)
```

This can efficiently support access beginning with:

```text
user_id
```

and potentially:

```text
user_id + created_at
```

But a query primarily filtering by:

```text
created_at
```

doesn't get the same benefit from this index.

Why?

Because the index is organized:

```text
user_id
    ↓
created_at
```

not:

```text
created_at
    ↓
user_id
```

So:

```sql
WHERE user_id = 123
```

fits the index.

```sql
WHERE user_id = 123
AND created_at > X
```

fits even better.

But:

```sql
WHERE created_at > X
```

cannot efficiently jump directly into the relevant regions because the index is first partitioned by `user_id`.

This is the **leftmost-prefix principle**.

You don't need to memorize the phrase.

Understand the physical ordering.

---

# 15. Index Design Is Really Query Design

Suppose the application has these queries:

### Query A

```sql
WHERE user_id = ?
```

### Query B

```sql
WHERE user_id = ?
AND created_at > ?
ORDER BY created_at DESC
```

### Query C

```sql
WHERE status = ?
AND created_at > ?
```

There isn't a single magical index.

We need to examine the workload.

Potentially:

```text
(user_id, created_at)

(status, created_at)
```

The important question becomes:

> **Which queries are important enough to justify the additional indexes?**

This is why database design and application design are deeply connected.

---

# 16. A Dangerous Mistake

Imagine we have:

```text
orders
500 million rows
```

And the team says:

> "The query is slow. Add an index."

That's not enough.

First ask:

```text
What is the query?
What is the data distribution?
What index exists?
How selective is the condition?
How many rows are returned?
Is the query using the index?
What execution plan was chosen?
```

Because an index can exist and still not solve the problem.

---

# 17. Example: Low Selectivity

Suppose:

```text
orders
500 million rows
```

and:

```text
status
```

has only:

```text
PENDING
COMPLETED
CANCELLED
```

Now query:

```sql
WHERE status = 'COMPLETED'
```

Maybe:

```text
450 million rows = COMPLETED
```

An index on `status` doesn't magically make retrieving 450 million rows cheap.

The query still needs an enormous amount of data.

So:

> **Indexes help most when they allow the database to eliminate a significant amount of irrelevant data.**

This is why selectivity matters.

---

# 18. The Database Can Ignore Your Index

This is an important mental model.

Suppose:

```text
INDEX(status)
```

exists.

You run:

```sql
WHERE status = 'COMPLETED'
```

The optimizer might decide:

> "Using the index is actually more expensive than scanning the table."

And choose a table scan.

That doesn't mean the index is broken.

It means:

> **The optimizer is choosing what it estimates to be the cheaper execution plan.**

This is why understanding query plans matters.

---

# 19. Query Execution Plan

You can conceptually ask MySQL:

> "How are you planning to execute this query?"

The database can show information about its execution plan.

Conceptually:

```text
Query
 ↓
Optimizer
 ↓
Execution Plan
 ├── use index?
 ├── which index?
 ├── how many rows estimated?
 ├── join order?
 └── access method?
```

In production debugging, engineers inspect these plans when queries unexpectedly become slow.

For HLD, remember the principle:

> **Query performance depends on the execution plan, not simply on whether an index exists.**

---

# 20. Why Database Queries Become Slower as Data Grows

Imagine this query:

```sql
SELECT *
FROM orders
WHERE user_id = 123;
```

With:

```text
10,000 rows
```

even a bad query may appear fast.

With:

```text
10 million rows
```

you notice the problem.

With:

```text
1 billion rows
```

the same query could become disastrous if it requires scanning a large fraction of the table.

This is why:

> **Database performance must be evaluated at the expected data scale, not merely on today's dataset.**

A query that takes 20 ms in development doesn't prove it will take 20 ms at production scale.

---

# 21. Now Connect This to HLD

Imagine an interviewer says:

> "Design an order service for 100 million users."

You shouldn't just draw:

```text
Application
     ↓
MySQL
```

You should start asking:

```text
What queries do we have?
       ↓
What is the data volume?
       ↓
What are the read/write ratios?
       ↓
Which queries are latency-sensitive?
       ↓
What indexes support those access patterns?
       ↓
Will one MySQL instance handle the workload?
```

Then the architecture evolves naturally.

---

# 22. One More Important Concept: Write Amplification

Imagine we have:

```text
1 table
+
5 secondary indexes
```

Every insert/update may need to update multiple structures.

So:

```text
1 logical write
      ↓
multiple physical changes
```

This is one reason a database with many indexes can have excellent read performance but poor write throughput.

So when someone says:

> "Let's index everything so reads are fast."

You should immediately think:

```text
Reads ↑
Writes ↓
Storage ↑
Memory usage ↑
```

Not necessarily bad — but it's a tradeoff.

---

# 23. The HLD Mental Model

You should now picture a MySQL query like this:

```text
                    SQL Query
                       │
                       ▼
                Query Optimizer
                       │
                chooses strategy
                       │
          ┌────────────┴────────────┐
          │                         │
      Use Index                 Table Scan
          │
          ▼
      B+ Tree
          │
          ▼
   Find relevant entries
          │
          ▼
    Primary key / rows
          │
          ▼
       Result
```

And the key optimization question is:

> **How much irrelevant data can we avoid touching?**

That's what indexes are fundamentally about.

---

# 24. Interview Scenario

Let's test your understanding before we move deeper.

We have:

```text
orders
--------------------------------
id
user_id
status
created_at
total
```

There are **500 million orders**.

Our application has two extremely common queries:

### Query 1

```sql
SELECT *
FROM orders
WHERE user_id = ?
ORDER BY created_at DESC
LIMIT 20;
```

### Query 2

```sql
SELECT *
FROM orders
WHERE status = 'PENDING'
AND created_at < ?
ORDER BY created_at ASC
LIMIT 100;
```

And the interviewer asks:

> **"What indexes would you consider, and why?"**

Don't just give me the indexes.

I want you to reason:

1. What does Query 1 need?
2. What does Query 2 need?
3. Why does column order matter in a composite index?
4. Why wouldn't one `(user_id, created_at)` index solve Query 2?
5. What tradeoff are we accepting by adding these indexes?

Give me your answer as if you're explaining it in an interview.
