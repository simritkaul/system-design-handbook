# Chapter 2 — SQL Databases (Relational Databases)

> **Goal:** Understand why SQL databases were invented, what problems they solve, why they are still the default choice for most applications, and where they start to struggle.

---

# 1. The World Before SQL

Imagine you're building an online bookstore.

You need to store:

- Users
- Books
- Orders
- Payments
- Reviews

At first, you might simply save everything into files.

```
users.txt

orders.txt

books.txt
```

This works for a few hundred records.

Now imagine:

- 10 million users
- 500 million orders
- Thousands of requests every second

Finding a single order by scanning a file line by line becomes far too slow.

We need something smarter.

---

# 2. The Problems We Need to Solve

A proper database should answer questions like:

```
Find user 48291

Find all orders placed today

Calculate total revenue

Show all books written by Tolkien

Delete cancelled orders
```

And it should do this:

- Quickly
- Reliably
- Without losing data
- Even if multiple users are accessing it simultaneously

This is the problem relational databases were designed to solve.

---

# 3. The Big Idea

Instead of storing random files,

store information in **tables**.

Example:

### Users

| UserID | Name  | City  |
| ------ | ----- | ----- |
| 1      | Alice | Delhi |
| 2      | Bob   | Pune  |

---

### Orders

| OrderID | UserID | Amount |
| ------- | ------ | ------ |
| 100     | 1      | ₹500   |
| 101     | 2      | ₹700   |

Notice something.

Orders don't repeat the user's name.

Instead,

they store

```
UserID = 1
```

This links the two tables together.

This relationship is why they're called **Relational Databases**.

---

# 4. Why Relationships Matter

Suppose Alice changes her name.

Without relationships:

```
Orders

Alice
Alice
Alice
Alice
Alice
```

You'd have to update every record.

With relationships:

```
Users

1 → Alice

Orders

UserID = 1
```

Only one row changes.

Everything remains consistent.

This reduces duplication and keeps data accurate.

---

# 5. What is a Schema?

A relational database doesn't allow random data.

Before inserting anything,

you define the structure.

Example:

```
Users

ID → Integer

Name → String

Age → Integer

Email → String
```

This predefined structure is called the **schema**.

It ensures every row follows the same format.

---

# 6. SQL — The Language

Once data is organized into tables,

we need a way to communicate with the database.

That's SQL (Structured Query Language).

SQL lets you:

- Create tables
- Insert data
- Read data
- Update data
- Delete data
- Join multiple tables
- Aggregate results
- Sort and filter records

Think of SQL as the language you use to ask questions about your data.

---

# 7. Why SQL Became So Popular

Relational databases solve many real-world business problems naturally.

Imagine Amazon.

A customer:

- creates an account
- places an order
- pays
- leaves a review

All of these entities are related.

```
Customer

↓

Orders

↓

Payments

↓

Products

↓

Reviews
```

SQL databases are exceptionally good at representing these relationships.

---

# 8. ACID Transactions — The Superpower

Imagine transferring ₹1000 from Account A to Account B.

Steps:

```
Subtract ₹1000

↓

Add ₹1000
```

What if the server crashes in between?

Money disappears.

That would be catastrophic.

SQL databases solve this with **transactions**.

A transaction guarantees that a group of operations behaves as a single unit.

Either:

```
Everything succeeds
```

or

```
Nothing happens
```

No half-completed work.

---

## The Four ACID Properties

### Atomicity

All operations happen together or not at all.

Example:

Bank transfer.

---

### Consistency

The database always remains valid.

Example:

Money cannot magically disappear.

---

### Isolation

Two users modifying data simultaneously shouldn't interfere with each other.

---

### Durability

Once the database confirms success,

the data survives crashes and power failures.

---

# 9. Primary Keys

Every row needs a unique identity.

Example:

```
Users

ID

1

2

3
```

No duplicates.

This unique identifier is the **Primary Key**.

Without it,

finding and updating records becomes unreliable.

---

# 10. Foreign Keys

Relationships between tables are created using **Foreign Keys**.

Example:

```
Orders

OrderID

UserID
```

UserID points to

```
Users.ID
```

This maintains referential integrity.

You can't create an order for a user who doesn't exist.

---

# 11. Indexes — Why Queries Become Fast

Imagine a 1000-page textbook.

To find "Binary Search"

you don't read every page.

You open the index.

A database index serves the same purpose.

Without an index:

```
Check row 1

Check row 2

Check row 3

...

Check row 10 million
```

With an index:

The database jumps directly to the relevant records.

Indexes dramatically improve read performance.

---

# 12. Where SQL Excels

SQL databases are the best choice when you need:

- Strong consistency
- Transactions
- Complex relationships
- Joins
- Reporting
- Financial data
- Inventory systems
- ERP software
- CRM systems

This is why banks, hospitals, airlines, and e-commerce companies rely heavily on SQL.

---

# 13. Where SQL Starts Struggling

Now imagine Instagram.

```
500 million users

10 billion posts

Thousands of writes every second

Petabytes of data
```

Can a single SQL server handle that?

Eventually, no.

Problems begin to appear:

- Vertical scaling has limits.
- Joins become expensive at massive scale.
- Horizontal scaling is difficult.
- Global replication becomes complex.

These limitations led to the rise of NoSQL databases.

---

# 14. Popular SQL Databases

## MySQL

- Open source
- Extremely popular
- Powers millions of web applications
- Default choice for many startups

---

## PostgreSQL

- More advanced SQL support
- Excellent extensibility
- Rich indexing options
- Strong standards compliance

Often preferred for complex enterprise applications.

---

## SQL Server

Popular in Microsoft ecosystems.

---

## Oracle Database

Designed for very large enterprise workloads.

---

# 15. Mental Model

Imagine a city's government office.

Every document has:

- a fixed format
- unique ID
- relationships with other records

Changing one official record updates the entire system consistently.

A relational database is that well-organized filing system.

It values **correctness, structure, and reliability** over raw flexibility.

---

# 16. Tradeoffs

### Advantages

- Strong consistency
- ACID transactions
- Powerful querying
- Joins
- Mature ecosystem
- Reliable
- Easy to reason about data

---

### Disadvantages

- Fixed schema
- Harder to scale horizontally
- Joins become costly at very large scale
- Not ideal for rapidly changing data models
- Can become a bottleneck for write-heavy global systems

---

# 17. Real-World Examples

### Banking

Transactions must never lose money.

SQL is the obvious choice.

---

### E-commerce

Orders, customers, inventory, and payments are highly related.

SQL is an excellent fit.

---

### HR Systems

Employees, departments, salaries, and attendance all have clear relationships.

---

### Airline Reservation Systems

Seat booking requires strong consistency to prevent double-booking.

---

# 18. Common Interview Questions

- Why use a relational database?
- What is a schema?
- Why do we need primary and foreign keys?
- What are ACID properties?
- Why are transactions important?
- How do indexes improve performance?
- Why doesn't Instagram store everything in MySQL?
- MySQL vs PostgreSQL—when would you choose one over the other?
- What challenges arise when scaling a SQL database horizontally?

---

# 19. Connections

This chapter lays the foundation for many concepts we'll revisit:

- **Replication** → Scale reads while keeping a single source of truth.
- **Sharding** → Distribute data across multiple databases to handle more writes.
- **Caching (Redis)** → Reduce load on the database by serving frequent reads from memory.
- **NoSQL** → Sacrifice some relational features to gain flexibility or scalability.
- **Distributed Transactions** → What happens when one database is no longer enough?

---
