# Chapter 9 — Search Databases / Search Engines

> **Goal:** Understand what search databases are, why they exist, what problems they solve, and why traditional databases aren't sufficient for search.

---

# 1. The Problem

Imagine you're building Amazon.

A user types

```text
iphone charger fast charging
```

into the search bar.

They don't know

- Product ID
- SKU
- Exact title

They just know a few words.

Your job is to return

the most relevant products.

---

# 2. The First Thought

Suppose we store products in MySQL.

| ID  | Name                         |
| --- | ---------------------------- |
| 1   | Apple 20W USB-C Fast Charger |
| 2   | Samsung Wireless Charger     |

Now someone searches

```text
fast charger
```

Can SQL do this?

Yes.

Maybe

```sql
WHERE Name LIKE '%fast%'
```

But now imagine

- 500 million products
- Millions of searches every second

Problems begin to appear.

---

# 3. Why SQL Isn't Enough

Suppose a user searches

```text
iphon charger
```

Notice the typo.

Should they still get

```text
iPhone Charger
```

Yes.

SQL cannot naturally understand that.

---

Another example.

Search

```text
running shoes
```

Should

```text
running shoe
```

match?

Yes.

Should

```text
best shoes for running
```

match?

Yes.

Should

```text
running sneakers
```

also be considered?

Maybe.

Traditional databases aren't built to understand language.

---

# 4. The Big Idea

Instead of storing data for transactions,

store data for **search**.

Search engines optimize one thing:

> Finding the most relevant documents for a user's query.

Notice the word.

Not

Rows.

Not

Objects.

They call everything

**Documents.**

---

# 5. What Is a Search Document?

Imagine a product.

```text
Product

↓

Title

Description

Brand

Category

Price
```

The search engine creates a searchable version of this information.

Each product becomes a search document.

---

# 6. The Secret — Inverted Index

This is the heart of every search engine.

Imagine a book.

At the back,

there's an index.

```text
Algorithms

↓

Page 32

Page 90

Page 112
```

Instead of scanning every page,

you jump directly.

Search engines do the same thing.

Instead of

```text
Document

↓

Words
```

they store

```text
Word

↓

Documents
```

Example

```text
iphone

↓

Doc1

Doc4

Doc20

Doc91
```

```text
charger

↓

Doc1

Doc7

Doc20
```

This structure is called an **Inverted Index**.

That's why search is fast.

---

# 7. Tokenization

Suppose the document says

```text
Apple 20W Fast Charger
```

The search engine splits it into

```text
Apple

20W

Fast

Charger
```

These are called

**Tokens.**

Searching works on tokens,

not entire sentences.

---

# 8. Stemming

Suppose users search

```text
run
```

Should

```text
running

runner

runs
```

match?

Usually yes.

Search engines reduce related words to a common root.

This process is called **stemming**.

---

# 9. Ranking

Suppose

1000 products contain

```text
iphone
```

Which should appear first?

Search engines rank results.

They consider factors like:

- Query relevance
- Term frequency
- Field importance (title vs description)
- Popularity (application-specific)
- Business rules (application-specific)

The goal is not just to find matches, but to return the **best** matches first.

---

# 10. Where Search Databases Shine

### E-commerce

Amazon

Flipkart

---

### Google Search

Web pages.

---

### YouTube

Video search.

---

### Netflix

Movie search.

---

### GitHub

Repository search.

---

### Log Search

Millions of application logs.

---

### Documentation

Searching APIs.

Knowledge bases.

---

# 11. Where They Perform Poorly

Suppose you're transferring money.

Need

Transactions.

Consistency.

Relationships.

Search engines are not designed for this.

SQL remains the correct choice.

---

# 12. Search Engine vs SQL

| SQL            | Search Engine     |
| -------------- | ----------------- |
| Exact matching | Full-text search  |
| Transactions   | Relevance ranking |
| Tables         | Documents         |
| Joins          | Inverted indexes  |
| Business data  | Search workloads  |

---

# 13. Search Engine vs Document Database

This confuses many people.

Both store

Documents.

But...

Document Database

stores documents as the **source of truth**.

Search Engine

stores documents as an **optimized searchable copy**.

Often,

the same product exists in both.

MongoDB

↓

Source

Elasticsearch

↓

Search index

---

# 14. Updating Data

Suppose the product changes.

```text
Price

↓

₹999 → ₹899
```

Usually,

the primary database updates first.

Then the search index is updated.

This means search engines are often **eventually consistent** with the primary database.

---

# 15. Popular Technologies

Examples include:

- Elasticsearch
- Apache Solr
- OpenSearch

We'll study their architectures later.

---

# 16. Mental Model

Imagine a library.

SQL Database

is the storage room containing every book.

Search Engine

is the library catalog.

You don't browse every shelf.

You search the catalog,

find the relevant books,

then retrieve them.

The catalog doesn't replace the books.

It helps you find them.

---

# 17. Tradeoffs

### Advantages

- Extremely fast full-text search
- Relevance ranking
- Typo tolerance
- Flexible search capabilities
- Scales well for search workloads

---

### Disadvantages

- Not the source of truth
- Not designed for transactions
- Indexes consume additional storage
- Data synchronization with the primary database is required

---

# 18. Real-World Examples

### Amazon

Product search.

---

### Google

Web search.

---

### Stack Overflow

Question search.

---

### GitHub

Repository and code search.

---

### Netflix

Movie search.

---

### Kibana + Elasticsearch

Searching application logs.

---

# 19. Common Interview Questions

- Why can't MySQL power Google Search?
- What is an inverted index?
- Why are search engines fast?
- Why maintain a separate search database?
- Why is Elasticsearch usually not the primary database?
- What happens when product data changes?
- Why are search indexes often eventually consistent?

---

# 20. Connections

A typical e-commerce architecture looks like this:

```text
           Product Created
                  │
                  ▼
            SQL Database
          (Source of Truth)
                  │
          Sync / Events
                  │
                  ▼
        Search Database
      (Searchable Index)
                  │
                  ▼
          User Searches
```

Each system has a different responsibility:

- **SQL** → Stores the authoritative product data.
- **Search Engine** → Makes that data searchable.
- **Object Storage** → Stores product images.
- Later, **Redis** may cache popular search results.

---

# Key Takeaways

- Search engines are optimized for **finding relevant documents**, not storing transactional data.
- Their core data structure is the **inverted index**, which maps words to documents.
- Features like tokenization, stemming, and ranking enable natural language search.
- Search databases are typically **secondary systems** that index data from a primary database.
- They complement SQL or document databases rather than replacing them.

---
