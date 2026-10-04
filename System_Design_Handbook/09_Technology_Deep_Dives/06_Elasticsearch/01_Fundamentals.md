# Module 9 — Chapter 6: Elasticsearch

We now make another important shift.

So far, our question has mostly been:

> **Where should application data live?**

Now imagine the data already exists, but users need to **find things by meaning, words, partial matches, relevance, filters, and ranking**.

That's a fundamentally different problem.

---

# 1. The Problem

Imagine Amazon has millions of products.

A user searches:

```text
"wireless headphones"
```

We don't merely want:

```text
WHERE product_name = "wireless headphones"
```

We want results like:

```text
1. Sony Wireless Noise Cancelling Headphones
2. Bose Bluetooth Headphones
3. Apple AirPods Pro
4. Wireless Earbuds...
```

And ideally:

- `"wireless"` matches
- `"headphones"` matches
- `"headphone"` also matches
- results are ranked by relevance
- filters work quickly
- typo tolerance may work
- synonyms may work
- results can be aggregated
- search remains fast across millions/billions of documents

Now try doing that with a conventional relational database.

You can certainly create indexes.

But as search requirements become richer, we run into a different class of problem.

---

# 2. Why a Normal Database Isn't Enough

Suppose our database contains:

```text
products
------------------------------
id
name
description
category
brand
price
...
```

We can create:

```text
INDEX(name)
```

Great.

For:

```text
WHERE name = 'iPhone'
```

an index is extremely useful.

But search becomes much harder when we ask:

```text
Find documents containing:
"wireless headphones"
```

Then:

```text
rank them by relevance
```

Then:

```text
support prefix matching
```

Then:

```text
handle typos
```

Then:

```text
filter by price/category/brand
```

Then:

```text
calculate aggregations
```

Then:

```text
search billions of documents
```

We're no longer simply asking:

> "Find this exact value."

We're asking:

> **"Find the most relevant documents according to the words and conditions in this query."**

That's a search problem.

---

# 3. The Big Idea

> **Elasticsearch is a distributed search and analytics engine that builds specialized indexes over documents so that complex searches can be performed quickly and ranked by relevance.**

The critical word is:

**index.**

Elasticsearch doesn't want to repeatedly scan every document.

Instead, it transforms the data into structures optimized for search.

---

# 4. The Fundamental Mental Model

Think about a book.

Suppose I give you a 2,000-page book and ask:

> "Find every page containing the word `distributed`."

Without an index:

```text
Page 1
Page 2
Page 3
...
Page 2000
```

You scan everything.

But imagine the back of the book contains:

```text
distributed → pages 17, 48, 91, 302, 781...
database    → pages 22, 91, 450...
replication → pages 78, 302...
```

Now:

```text
"distributed"
      ↓
look up index
      ↓
candidate documents
      ↓
rank/filter
      ↓
results
```

That is the intuition behind the **inverted index**.

---

# 5. Inverted Index

Suppose we have three documents:

```text
D1:
"Redis is an in-memory database"

D2:
"Redis provides fast data access"

D3:
"Database systems store data"
```

Instead of thinking only:

```text
Document → words
```

we build:

```text
Word → documents
```

Conceptually:

```text
database
   → D1
   → D3

Redis
   → D1
   → D2

data
   → D2
   → D3

fast
   → D2
```

Now searching for:

```text
Redis database
```

doesn't require scanning every document.

We can retrieve candidate documents from the index.

This is the foundational idea behind search engines.

---

# 6. Why Is It Called "Inverted"?

Because we've inverted the relationship.

Normal representation:

```text
Document
   ↓
contains words
```

Inverted index:

```text
Word
   ↓
appears in documents
```

So:

```text
Document → Terms
```

becomes:

```text
Term → Documents
```

This is one of the few Elasticsearch internals you should absolutely understand for HLD.

You don't need to know the implementation details of the indexing library.

You need to know:

> **Elasticsearch pre-builds data structures that make text search efficient.**

---

# 7. But What Exactly Is a "Word"?

Here's where search becomes more interesting.

Suppose the document contains:

```text
"running shoes"
```

User searches:

```text
"run shoe"
```

Should that match?

A search engine needs to understand text.

This involves **analysis**.

Conceptually:

```text
Raw text
   ↓
Tokenization
   ↓
Normalization
   ↓
Terms
   ↓
Inverted index
```

For example:

```text
"Running Shoes"
        ↓
running
shoes
```

Depending on the configured analysis process, `"running"` might also be reduced to a form related to `"run"`.

The important concept:

> **Elasticsearch doesn't necessarily index the raw string exactly as provided.**

It analyzes text to make search useful.

---

# 8. Indexing vs Searching

This creates two distinct phases.

### Indexing

When a document enters Elasticsearch:

```text
Document
   ↓
Analyze fields
   ↓
Build search structures
   ↓
Store indexed representation
```

### Searching

When a user searches:

```text
Query
   ↓
Analyze query
   ↓
Consult indexes
   ↓
Find candidate documents
   ↓
Score/rank
   ↓
Return results
```

So Elasticsearch is doing work **ahead of time** during indexing to make future searches fast.

That's an important architectural tradeoff.

---

# 9. The Tradeoff

We are effectively saying:

> **Do more work when data is indexed so that reads/searches are much faster.**

Compare:

```text
Traditional scan

Write → cheap
Read/search → expensive
```

versus:

```text
Search engine

Write/index → more expensive
Search → much faster
```

This is why Elasticsearch isn't simply another primary database.

---

# 10. Relevance

Finding matching documents isn't enough.

Suppose:

```text
D1:
"wireless headphones"

D2:
"wireless headphones with noise cancellation"

D3:
"laptop with wireless connectivity"
```

Search:

```text
wireless headphones
```

We want:

```text
D2
D1
D3
```

rather than arbitrary ordering.

Elasticsearch calculates a **relevance score** based on characteristics of the match.

You don't need to memorize the exact scoring formula for HLD.

Understand the idea:

```text
Query
 ↓
Matching documents
 ↓
How well does each document match?
 ↓
Score
 ↓
Ranked results
```

This is a major difference from a basic key-value lookup.

---

# 11. Elasticsearch Is Not Just "Text Search"

It can also handle structured filtering and aggregations.

Suppose Amazon wants:

```text
Search:
"wireless headphones"

Filters:
brand = Sony
price < ₹20,000

Aggregations:
How many results by brand?
How many by price range?
```

A search engine can combine:

```text
Text matching
+
Structured filtering
+
Aggregations
+
Ranking
```

This is why Elasticsearch becomes useful as a **search and discovery layer**.

---

# 12. Where Does the Data Come From?

Here's an important architectural question.

Suppose our source of truth is PostgreSQL:

```text
                PostgreSQL
                Source of truth
                     │
                     │
                     ▼
               Elasticsearch
                 Search index
                     │
                     ▼
                  Users
```

A common architecture is:

```text
Application
    │
    ├──────────→ PostgreSQL
    │
    └──────────→ Elasticsearch
```

But we now have a synchronization problem.

If a product changes:

```text
PostgreSQL
   ↓
product updated
```

we need Elasticsearch to eventually reflect that change.

This introduces an important principle:

> **Elasticsearch is often a derived search index rather than the system of record.**

---

# 13. Why Not Make Elasticsearch the Primary Database?

You technically can store documents there.

But consider:

```text
Order
Payment
Inventory
Customer
```

These often require:

- transactional guarantees
- strong business constraints
- relationships
- reliable source-of-truth semantics

Elasticsearch isn't designed to replace a relational database for those responsibilities.

A much healthier architecture is often:

```text
                 Source of Truth
                       │
                PostgreSQL/MySQL
                       │
                  indexing pipeline
                       │
                       ▼
                 Elasticsearch
                       │
                       ▼
                 Search requests
```

So:

```text
Database
    = authoritative state

Elasticsearch
    = optimized search representation
```

---

# 14. This Creates Eventual Consistency

Suppose:

```text
T0:
Product price = ₹10,000
```

User updates it:

```text
T1:
Database → ₹9,000
```

But Elasticsearch hasn't received the update yet:

```text
T1:
Database       → ₹9,000
Elasticsearch  → ₹10,000
```

For a short period, search may return stale information.

Eventually:

```text
Elasticsearch → ₹9,000
```

So your architecture now contains:

```text
Source of truth
       ↓
Asynchronous indexing
       ↓
Search index
```

which means:

> **Search results can be eventually consistent with the primary database.**

This is a very common HLD tradeoff.

---

# 15. Elasticsearch Scaling

Now let's connect this to the distributed-systems concepts you've already learned.

Imagine:

```text
1 billion documents
```

One machine isn't enough.

So Elasticsearch distributes data across **shards**.

Conceptually:

```text
                  Elasticsearch Index
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
          Shard 1     Shard 2     Shard 3
             │           │           │
          Documents    Documents   Documents
```

Each shard contains a portion of the index.

This gives us horizontal scale.

---

# 16. Shards and Replicas

Suppose:

```text
Primary shards = 3
Replicas = 1
```

Conceptually:

```text
Shard 1 ── Replica 1
Shard 2 ── Replica 2
Shard 3 ── Replica 3
```

If a node fails:

```text
Primary shard
      X
      ↓
Replica
      ↓
Can become primary
```

So the same concepts appear again:

```text
Partitioning
+
Replication
+
Failure recovery
```

This is why Module 2 wasn't just theoretical.

---

# 17. Querying Across Shards

Here's an important consequence.

Suppose the query is:

```text
"wireless headphones"
```

and relevant documents might exist on:

```text
Shard 1
Shard 2
Shard 3
```

A coordinating node can conceptually do:

```text
                Query
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
    Shard 1    Shard 2    Shard 3
       │          │          │
    results     results     results
       └──────────┼──────────┘
                  ▼
              merge/rank
                  ▼
              final results
```

So distributed search itself has coordination overhead.

Again:

> **Horizontal scale introduces distributed-system complexity.**

---

# 18. The Hot Shard Problem

What happens if the data or queries are badly distributed?

Suppose one shard receives disproportionately large traffic:

```text
Shard 1 → 10,000 req/sec
Shard 2 → 100 req/sec
Shard 3 → 100 req/sec
```

Adding more total nodes doesn't automatically solve the problem.

You can still have:

```text
hot shard
   ↓
CPU pressure
   ↓
latency
```

Same fundamental lesson:

> **Distribution only helps when work is distributed well.**

You've now seen this in:

- Cassandra
- DynamoDB
- Elasticsearch

That is not a coincidence.

---

# 19. Elasticsearch vs Database Indexes

This is a classic interview question.

Someone might say:

> "Why don't we just put indexes on PostgreSQL?"

Sometimes you absolutely should.

If you need:

```text
WHERE user_id = 42
```

a database index is perfect.

But if you need:

```text
"best wireless headphones under ₹20k"
```

with:

- relevance
- tokenization
- stemming
- fuzzy matching
- phrase search
- autocomplete
- faceting
- ranking

then a dedicated search engine is much better suited.

So:

```text
Database index
     ↓
Efficient lookup over structured data

Search index
     ↓
Efficient information retrieval
```

The distinction is important.

---

# 20. Elasticsearch vs MongoDB

MongoDB has a document model and supports text-search capabilities.

So why use Elasticsearch?

Because Elasticsearch is fundamentally optimized around:

```text
Search
Relevance
Text analysis
Ranking
Aggregations
Distributed search
```

MongoDB's primary abstraction remains:

```text
Document database
```

while Elasticsearch's primary abstraction is:

```text
Search engine
```

This is another example of choosing a technology based on the **dominant workload**, rather than the data format.

---

# 21. Elasticsearch vs Cassandra

These two are particularly different.

### Cassandra

Optimized around:

```text
Massive distributed storage
+
High write throughput
+
Predictable key-based queries
```

### Elasticsearch

Optimized around:

```text
Search
+
Ranking
+
Text retrieval
+
Filtering
+
Aggregations
```

Both distribute data.

But distribution isn't the purpose.

It's the mechanism that allows them to achieve different goals.

That's an important distinction.

---

# 22. What Happens When Elasticsearch Goes Down?

Suppose:

```text
PostgreSQL ✓
Elasticsearch ✗
```

If Elasticsearch is only the search layer:

```text
Core application data
      ↓
still safe
```

But:

```text
Search functionality
      ↓
degraded/unavailable
```

That's a very desirable architecture.

Compare that with making Elasticsearch your only source of truth:

```text
Elasticsearch ✗
      ↓
entire application data unavailable
```

This illustrates an important architectural principle:

> **Don't give a specialized derived system responsibilities that belong to the system of record unless you have a deliberate reason to do so.**

---

# 23. Index Rebuilding

Here's another consequence of treating Elasticsearch as a derived index.

Suppose the index becomes corrupted or we change the indexing strategy.

If PostgreSQL is the source of truth:

```text
PostgreSQL
    │
    │ reindex
    ▼
Elasticsearch
```

We can rebuild the search index from authoritative data.

This is extremely useful operationally.

It means:

> **The search index is disposable/reconstructible.**

That's a powerful architectural property.

---

# 24. When Would I Choose Elasticsearch?

Strong signals:

```text
Full-text search
+
Relevance ranking
+
Autocomplete
+
Fuzzy search
+
Faceted navigation
+
Search across huge document sets
+
Fast filtering + aggregation
```

For example:

```text
E-commerce product search
Job search
Log search
Document search
Content discovery
```

The common thread is:

> **The user is trying to find information rather than retrieve a known object by its primary key.**

---

# 25. When Would I NOT Choose Elasticsearch?

Don't introduce it simply because:

> "We have lots of data."

That's not enough.

If you only need:

```text
GET user by ID
UPDATE user
DELETE user
```

a database is sufficient.

Similarly, don't make Elasticsearch the primary source of truth merely because it can store documents.

Use the specialized tool because you have a **search problem**.

---

# 26. The Biggest Tradeoffs

### Advantages

- Excellent full-text search
- Relevance ranking
- Powerful filtering
- Aggregations
- Horizontal scaling
- Specialized search indexes
- Fast search over large datasets

### Disadvantages

- Additional infrastructure
- Data synchronization complexity
- Eventual consistency when used as a derived index
- Indexing consumes resources
- Poorly designed shards can create hot spots
- Not a replacement for a transactional database
- Operational complexity can become significant at scale

The important one for HLD:

> **You are introducing another distributed system and another copy of your data.**

That should never be free in your architecture diagram.

---

# 27. A Typical HLD Architecture

Suppose we're building an e-commerce system.

```text
                    ┌───────────────┐
                    │   Client      │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Application   │
                    └───────┬───────┘
                            │
                 ┌──────────┴──────────┐
                 │                     │
                 ▼                     ▼
          ┌─────────────┐       ┌─────────────┐
          │ PostgreSQL  │       │ Elasticsearch│
          │             │       │             │
          │ Source of   │       │ Search      │
          │ truth       │       │ index       │
          └──────┬──────┘       └─────────────┘
                 │
                 │ changes/events
                 ▼
          ┌─────────────┐
          │ Indexing    │
          │ Pipeline    │
          └─────────────┘
```

Read paths might then be:

```text
Product detail
    ↓
PostgreSQL

Search
    ↓
Elasticsearch
```

This separation is extremely common conceptually:

> **Use the database to own the data. Use Elasticsearch to make that data searchable.**

---

# 28. The Technology Selection Question

Imagine an interviewer says:

> "We need to build search for 500 million products. Users search by product name and description, expect relevance ranking, filters by brand/category/price, and autocomplete."

Your reasoning should be:

```text
Text-heavy search
       ↓
Relevance
       ↓
Filtering
       ↓
Autocomplete
       ↓
Large dataset
       ↓
Dedicated search engine
       ↓
Elasticsearch is a strong candidate
```

Then immediately ask:

> **Where is the source of truth?**

Probably:

```text
MySQL / PostgreSQL
```

Then:

> **How does data reach Elasticsearch?**

```text
DB changes
   ↓
indexing pipeline
   ↓
Elasticsearch
```

Then:

> **What if Elasticsearch is unavailable?**

Search degrades, but the core data remains safe.

That is HLD thinking.

---

# 29. The Mental Model

Keep this:

```text
                 Elasticsearch
                       │
                 Search Engine
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
       Inverted Index       Distributed
             │                Shards
             ▼                   │
        Text Search             ▼
        Relevance           Horizontal Scale
             │
             ├── Filtering
             ├── Ranking
             ├── Aggregations
             └── Autocomplete
```

The central idea:

> **Elasticsearch spends additional work and storage building specialized search indexes so that information retrieval can be extremely fast and expressive.**

---

# 30. Where We Are in the Technology Map

Our database family now looks like:

```text
                         Data systems
                              │
          ┌───────────────────┼────────────────────┐
          │                   │                    │
          ▼                   ▼                    ▼
     Relational           NoSQL/Distributed     Search
          │                   │                    │
      MySQL              MongoDB               Elasticsearch
      PostgreSQL         Cassandra
                         DynamoDB
```

And each exists because of a different dominant problem:

```text
MySQL/PostgreSQL
→ relationships + transactions

MongoDB
→ document/aggregate modeling

Cassandra
→ massive distributed scale + availability

DynamoDB
→ managed massive-scale key-value/document access

Elasticsearch
→ finding and ranking information
```

That's the pattern I want you to keep seeing.

---

## Next: RabbitMQ

We've now finished the major **data-storage/search** portion of our technology journey.

Next we leave databases entirely.

The question becomes:

> **What if one service needs to tell another service to do something, but we don't want the sender to wait for the receiver?**

You already learned the concept in Module 7:

```text
Asynchronous Communication
        ↓
Message Queues
        ↓
Delivery Guarantees
        ↓
Retries
        ↓
Ordering
```

Now we'll see how a real technology realizes those ideas.

**Chapter 7 — RabbitMQ**.
