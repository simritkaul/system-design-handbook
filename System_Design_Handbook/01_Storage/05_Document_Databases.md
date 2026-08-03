# Chapter 5 — Document Databases

> **Goal:** Understand what document databases are, why they exist, what problems they solve, and when they are a better choice than SQL or Key-Value databases.

---

# 1. The Problem

Imagine you're building Amazon.

Every product is different.

A Laptop has

```text
RAM
CPU
Storage
Screen Size
```

A T-shirt has

```text
Size
Color
Material
Sleeve Length
```

A Book has

```text
Author
ISBN
Publisher
Pages
```

A Refrigerator has

```text
Capacity
Power Rating
Compressor Type
```

Now imagine trying to store all of these in one SQL table.

| ID  | Name | RAM | CPU | Storage | Author | ISBN | Color | Material | Capacity |
| --- | ---- | --- | --- | ------- | ------ | ---- | ----- | -------- | -------- |

Most columns would be empty.

Your schema becomes huge, sparse, and difficult to evolve.

---

# 2. Why SQL Struggles Here

Relational databases expect every row in a table to follow the same schema.

For example:

Users

```text
ID
Name
Email
Age
```

Every user has those fields.

Products are different.

Every product category has its own attributes.

Changing the schema every week becomes painful.

---

# 3. The Big Idea

Instead of forcing every record to look the same,

store each object as a **document**.

Example

Laptop

```json
{
  "name": "MacBook Pro",
  "ram": "32 GB",
  "cpu": "M4 Pro",
  "storage": "1 TB"
}
```

Book

```json
{
  "name": "Clean Code",
  "author": "Robert C. Martin",
  "isbn": "9780132350884"
}
```

T-Shirt

```json
{
  "name": "Polo T-Shirt",
  "size": "L",
  "color": "Blue"
}
```

Each document can have a different structure.

That's the key insight.

---

# 4. What Is a Document?

A document is simply a self-contained object.

Usually represented as JSON (or a binary JSON format internally).

Think of it as:

```text
Key

↓

Entire Object
```

Unlike a key-value store where the database often treats the value as opaque, document databases understand the structure of the document and can query individual fields inside it.

---

# 5. Collections Instead of Tables

SQL has

```text
Table
```

Document databases have

```text
Collection
```

Example

Products Collection

contains

```text
Laptop Document

Book Document

Phone Document

TV Document
```

Every document can look different.

---

# 6. Flexible Schema

This is the biggest advantage.

Imagine tomorrow laptops add

```text
AI Accelerator
```

Only new laptop documents need that field.

Nothing else changes.

Books don't care.

Furniture doesn't care.

No table migration required.

---

# 7. Embedded Data

Suppose every product has reviews.

SQL

Usually

```text
Products

↓

Reviews
```

Two tables.

Linked by joins.

Document databases often embed related data.

Example

```json
{
  "name": "MacBook",

  "reviews": [
    {
      "user": "Alice",
      "rating": 5
    },
    {
      "user": "Bob",
      "rating": 4
    }
  ]
}
```

Everything is stored together.

Reading the product also reads its reviews.

This can make common read operations much simpler.

---

# 8. Why Is This Useful?

Imagine opening a product page.

What do you need?

- Product
- Images
- Reviews
- Specifications

If they're all stored together,

one read may be enough.

In SQL,

you might need multiple tables and joins.

---

# 9. Where Document Databases Shine

### Product Catalogs

Every product differs.

Perfect fit.

---

### User Profiles

Some users have

```text
Twitter
LinkedIn
GitHub
```

Others don't.

No problem.

---

### CMS (Content Management Systems)

Every page has different fields.

---

### Blogs

Articles

Comments

Tags

Metadata

---

### Mobile Apps

Settings vary by user.

Flexible documents work well.

---

### Configuration Data

Different services require different configuration fields.

---

# 10. Where They Perform Poorly

Suppose you're building a banking system.

You have

Accounts

Transactions

Loans

Payments

Customers

Everything is deeply related.

Strong transactions and joins are important.

SQL remains a better fit.

---

# 11. Document DB vs SQL

| SQL                        | Document Database                       |
| -------------------------- | --------------------------------------- |
| Fixed schema               | Flexible schema                         |
| Tables                     | Collections                             |
| Rows                       | Documents                               |
| Relationships via joins    | Often embed related data                |
| Strong normalization       | Often duplicate some data intentionally |
| Excellent for transactions | Excellent for evolving data models      |

Neither is universally better.

It depends on the problem.

---

# 12. Document DB vs Key-Value

At first glance they look similar.

Both have

```text
Key

↓

Value
```

The difference is important.

Key-Value

```text
user123

↓

Blob of data
```

The database mainly cares about the key.

The value is often opaque.

---

Document Database

```json
{
  "name": "John",
  "age": 30,
  "city": "Delhi"
}
```

The database understands the document's fields.

You can search by

```text
City

Age

Name

Country
```

without retrieving every document first.

This is a major advantage.

---

# 13. Horizontal Scaling

Documents are usually independent.

That makes them relatively straightforward to distribute across multiple machines.

This is one reason document databases are popular in large web applications.

---

# 14. Popular Technologies

Examples include:

- MongoDB
- Couchbase
- CouchDB
- Amazon DocumentDB (MongoDB-compatible API)

At this stage, think of these as implementations of the document database concept. We'll study their architectures later.

---

# 15. Mental Model

Imagine filing cabinets.

SQL

Every folder must have exactly the same form.

If a field is added,

every form changes.

---

Document Database

Every folder can contain different papers.

One folder has

```text
Passport

License

PAN
```

Another has

```text
Passport

Visa
```

Another has

```text
Passport

Driving License

Insurance
```

Each folder is complete in itself.

The filing cabinet doesn't require every folder to contain identical documents.

---

# 16. Tradeoffs

### Advantages

- Flexible schema
- Natural representation of JSON objects
- Easier application development for evolving data
- Often fewer joins because related data can be embedded
- Good horizontal scalability

---

### Disadvantages

- Data duplication is more common
- Complex joins are weaker than in relational databases
- Updating duplicated data can become challenging
- Not ideal for highly relational business domains

---

# 17. Real-World Examples

### Amazon

Product catalog.

Different products have different attributes.

---

### Content Management Systems

Pages with different structures.

---

### Social Media Profiles

Different users expose different profile information.

---

### Gaming

Each player may have a different inventory, achievements, cosmetics, and settings.

---

# 18. Common Interview Questions

- Why use a document database instead of SQL?
- What is schema flexibility?
- When should you embed data instead of referencing it?
- Why do document databases often duplicate data?
- Why are they popular for product catalogs?
- How are they different from key-value stores?
- When is SQL still the better choice?

---

# 19. Connections

You've now learned three major storage models:

- **SQL** → Structured, relational, transaction-heavy data.
- **Key-Value** → Extremely fast lookups when you already know the key.
- **Document** → Flexible, semi-structured objects with queryable fields.

The next concept we'll study is **Wide Column Databases**, which tackle a different problem entirely:

> "What if I need to store and write enormous amounts of structured data across hundreds or thousands of machines?"

That's where systems like Cassandra and Bigtable enter the picture.

---

# Key Takeaways

- A document database stores **self-contained documents**, typically represented as JSON-like structures.
- Documents within the same collection **do not need identical schemas**.
- The database understands the fields inside each document, enabling queries beyond simple key lookups.
- Document databases are a natural fit for evolving, semi-structured data such as product catalogs, user profiles, CMS content, and application settings.
- The tradeoff for flexibility is that you often denormalize or duplicate data, making some updates and complex relationships harder than in a relational database.

---

One small note about the handbook as a whole: I think we should avoid saying "NoSQL databases are schema-less." That's a common shortcut, but it's misleading. A better way to think about it is:

- **SQL:** Schema is defined first and enforced by the database.
- **Document databases:** Schema is flexible and often enforced by the application rather than being rigidly fixed in the database.

That distinction is both more accurate and much closer to how these systems are used in production.
