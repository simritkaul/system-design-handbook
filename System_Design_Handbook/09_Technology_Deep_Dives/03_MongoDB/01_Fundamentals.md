# Module 9 — Chapter 3: MongoDB

We've now seen two technologies that solve the same broad problem:

```text
                 Structured business data
                         │
                  Relational model
                    /          \
                   ▼            ▼
                MySQL      PostgreSQL
```

Both assume that relationships between entities are important.

MongoDB starts from a different premise:

> **What if the thing we usually retrieve is naturally a self-contained object?**

That is the problem we're going to explore.

---

# 1. The Problem

Imagine we're building an e-commerce product catalog.

A product might look like:

```text
Product
{
    id: 101,
    name: "MacBook Pro",
    price: 199999,

    specifications: {
        ram: "32GB",
        storage: "1TB",
        screenSize: 14
    },

    variants: [
        {
            color: "Silver",
            storage: "1TB"
        },
        {
            color: "Black",
            storage: "2TB"
        }
    ],

    tags: [
        "laptop",
        "apple",
        "premium"
    ]
}
```

Notice something.

When the application displays a product page, it usually wants:

> **The product and all of its associated information.**

It doesn't necessarily want to perform a dozen relational joins to reconstruct the product.

This leads to a different modeling idea:

```text
Relational thinking:

Product
   ↓
Specifications
   ↓
Variants
   ↓
Tags


Document thinking:

Product
   └── everything needed for the product
```

That's the fundamental motivation behind a document database.

---

# 2. The Big Idea

> **MongoDB stores data as documents that can contain nested and variable structures, making it natural to model data around the way an application accesses it.**

Instead of primarily thinking:

```text
Tables → rows → relationships
```

we think:

```text
Collections → documents → nested data
```

Conceptually:

```text
Collection
    │
    ├── Document
    │     ├── field
    │     ├── field
    │     └── nested object
    │
    ├── Document
    │
    └── Document
```

This changes how we design the data model.

---

# 3. MongoDB's Fundamental Model

The basic hierarchy is:

```text
MongoDB
   │
   └── Database
          │
          └── Collection
                 │
                 ├── Document
                 ├── Document
                 └── Document
```

Compare that with MySQL:

```text
MySQL
  │
  └── Database
         │
         └── Table
                │
                ├── Row
                ├── Row
                └── Row
```

But the important difference isn't the terminology.

It's the **shape of the data**.

---

# 4. A Document

A MongoDB document is conceptually similar to JSON:

```text
{
    "userId": 42,
    "name": "Simrit",
    "email": "simrit@example.com",

    "preferences": {
        "theme": "dark",
        "notifications": true
    },

    "skills": [
        "React",
        "C#",
        "System Design"
    ]
}
```

The document can contain:

```text
Strings
Numbers
Booleans
Arrays
Nested objects
Dates
Identifiers
```

This makes hierarchical data natural.

---

# 5. Why Does This Matter?

Let's take a social-media profile.

A user's profile might contain:

```text
User
 ├── name
 ├── bio
 ├── profilePicture
 ├── socialLinks
 ├── preferences
 └── interests
```

In a relational model, you could normalize these into:

```text
User
UserPreferences
SocialLinks
UserInterests
...
```

That's perfectly valid.

But if the dominant operation is:

```text
GET /users/123
```

and you almost always want the entire profile, the document model can represent the aggregate directly:

```text
User Document
{
    ...
    preferences: {...},
    socialLinks: [...],
    interests: [...]
}
```

The database model starts resembling the application's object model.

---

# 6. This Leads to the Most Important MongoDB Concept

## Model Around Access Patterns

This is a major difference from the way you were taught to think about relational modeling.

With a relational database, we often start with:

```text
What entities exist?
What relationships exist?
How should we normalize them?
```

With a document database, we also ask:

> **What does the application usually retrieve together?**

Suppose:

```text
Order
 ├── items
 ├── shippingAddress
 └── pricing
```

If the application almost always retrieves all of these together:

```text
GET /orders/123
```

it may make sense to keep them in one document.

```text
Order
{
    id: 123,

    items: [...],

    shippingAddress: {...},

    pricing: {...}
}
```

The access pattern influences the data model heavily.

---

# 7. Embedding

When related data is stored inside a document, we call this **embedding**.

For example:

```text
Order
{
    id: 123,

    customerId: 42,

    items: [
        {
            productId: 10,
            quantity: 2,
            price: 500
        },
        {
            productId: 20,
            quantity: 1,
            price: 1000
        }
    ]
}
```

Instead of:

```text
Orders
OrderItems
Products
```

we've embedded the order items inside the order.

This can make reads extremely straightforward:

```text
GET order 123
        ↓
one document
        ↓
everything needed
```

---

# 8. But Embedding Has a Cost

Imagine we embed a user's orders:

```text
User
{
    id: 42,

    orders: [
        {...},
        {...},
        {...},
        ...
    ]
}
```

What happens after:

```text
10 orders
100 orders
10,000 orders
1,000,000 orders
```

The document keeps growing.

That's obviously problematic.

So embedding isn't:

> "Always put related data inside the parent."

It's:

> **Embed data when the relationship and access pattern make the embedded data a natural part of the aggregate.**

---

# 9. Referencing

The alternative is to keep documents separate and store references.

For example:

```text
User
{
    id: 42
}

Order
{
    id: 123,
    userId: 42
}
```

Now the relationship is represented through an identifier.

Conceptually:

```text
User
  │
  │ userId
  ▼
Orders
```

This is closer to the relational approach.

So MongoDB gives us two important modeling strategies:

```text
Related data
    │
    ├── Embed
    │
    └── Reference
```

The decision depends heavily on access patterns and lifecycle.

---

# 10. When Is Embedding Attractive?

Consider:

```text
Blog Post
 ├── title
 ├── content
 └── comments
```

If:

- comments belong exclusively to the post
- they're usually retrieved with the post
- the number of comments is bounded/manageable

then embedding can be attractive:

```text
Post
{
    title: "...",
    content: "...",

    comments: [
        {...},
        {...},
        {...}
    ]
}
```

One read can retrieve the entire aggregate.

---

# 11. When Is Referencing Better?

Imagine:

```text
Post
   │
   └── 10 million comments
```

Embedding them would create a giant document.

Instead:

```text
Post
   │
   └── id

Comment
   ├── postId
   └── ...
```

Now comments can grow independently.

This gives us an important rule:

> **Bounded, tightly coupled data tends to be a better embedding candidate. Unbounded or independently accessed data tends to favor separate documents.**

That's much more useful than memorizing "embed vs reference."

---

# 12. MongoDB and Schema Flexibility

Now let's revisit something we saw with PostgreSQL.

MongoDB is **schema-flexible**.

Imagine:

```text
Document A
{
    name: "Laptop",
    price: 1000
}
```

and:

```text
Document B
{
    name: "Phone",
    price: 500,
    camera: {
        megapixels: 48
    }
}
```

The documents don't have to have exactly the same fields.

This can be extremely useful when the data structure naturally varies.

---

# 13. But "Schema-less" Is a Dangerous Phrase

You may hear:

> "MongoDB is schema-less."

Don't interpret that as:

> "There is no schema."

Your application still has an expected structure.

For example:

```text
Product
{
    name: string,
    price: number,
    category: string
}
```

Your application probably depends on these fields existing.

The difference is:

> **The database doesn't force every document into one rigid relational table schema in the same way.**

You can still enforce structure through application-level validation or database mechanisms where appropriate.

So think:

```text
Schema flexibility
```

rather than:

```text
No schema
```

---

# 14. Why Would a Team Want This?

Imagine a rapidly evolving product.

Version 1:

```text
Product
{
    name,
    price
}
```

Version 2:

```text
Product
{
    name,
    price,
    dimensions
}
```

Version 3:

```text
Product
{
    name,
    price,
    dimensions,
    warranty
}
```

Version 4:

```text
Product
{
    name,
    price,
    dimensions,
    warranty,
    regionalAttributes
}
```

A document model can accommodate these changes naturally.

But again:

> **Flexibility is useful only when the domain actually needs flexibility.**

For stable financial data, uncontrolled schema flexibility can become a liability.

---

# 15. MongoDB and Transactions

This is an important correction to a common misconception.

You may hear:

> "NoSQL databases don't support transactions."

That's false as a blanket statement.

MongoDB supports transactions, including multi-document transactions.

However, the architectural lesson is more nuanced.

If your entire application constantly needs:

```text
Transaction
    ↓
modify 17 documents
    ↓
all succeed or all rollback
```

you should question whether a document-oriented model is actually the best fit.

A good document model tries to make the most common atomic operation fit naturally inside one document.

For example:

```text
Order
{
    items: [...],
    pricing: {...},
    shipping: {...}
}
```

Then many updates can happen atomically at the document level.

This is an important design philosophy:

> **Good document modeling reduces the need for cross-document transactions.**

---

# 16. Atomicity at the Document Boundary

Think about:

```text
Order
{
    status: "PAID",
    payment: {...}
}
```

If both fields belong to the same logical aggregate, updating them together inside the same document is much simpler than coordinating multiple tables.

Conceptually:

```text
One document
     ↓
One atomic update
```

Whereas a normalized relational design might involve:

```text
Orders
Payments
OrderStatus
```

and require a transaction across them.

Neither design is universally better.

The question is:

> **Where is the natural boundary of the business object?**

---

# 17. MongoDB's Scaling Model

Now let's move toward HLD.

MongoDB is designed to scale horizontally.

The basic idea is:

```text
                  Application
                       │
                  MongoDB Cluster
                  /      |      \
                 ▼       ▼       ▼
              Shard A  Shard B  Shard C
```

Each shard is responsible for some portion of the data.

This is conceptually similar to the sharding you learned in Module 2.

---

# 18. Why Sharding?

Suppose we have:

```text
10 TB data
```

and:

```text
100,000 writes/sec
```

One machine may eventually become the bottleneck.

Instead:

```text
              Dataset
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
      Shard A  Shard B  Shard C
```

Each shard handles part of the workload.

Now:

```text
Storage capacity ↑
Write capacity  ↑
Read capacity   ↑
```

assuming the workload distributes well.

---

# 19. The Hard Part: Choosing the Shard Key

Here's where our old friend appears again.

> **The distribution key matters enormously.**

Suppose we shard by:

```text
userId
```

and users are evenly distributed:

```text
Shard A → users 1–1M
Shard B → users 1M–2M
Shard C → users 2M–3M
```

Good.

But suppose we shard by:

```text
country
```

and:

```text
India → 50%
US    → 20%
Others → 30%
```

We could end up with:

```text
Shard A → India → overloaded
Shard B → US
Shard C → everyone else
```

Now we have a **hot shard**.

Sharding only helps if the partitioning strategy distributes the workload.

---

# 20. A Bad Shard Key

Imagine an events collection:

```text
Event
{
    timestamp,
    userId,
    eventType
}
```

You choose:

```text
timestamp
```

as the shard key.

If new events constantly arrive with increasing timestamps, new writes may disproportionately target the same region of the key space.

You can create a hotspot.

This connects directly to the sharding concepts from Module 2:

```text
Partition key
      ↓
Distribution
      ↓
Hotspots?
      ↓
Scalability
```

---

# 21. Replication

MongoDB also uses replication for availability.

Conceptually:

```text
              Replica Set
           /      |       \
          ▼       ▼        ▼
       Primary  Secondary Secondary
```

Writes normally go through the primary.

Secondaries maintain copies.

If the primary fails:

```text
Primary
   X
   ↓
Election
   ↓
Secondary promoted
```

Notice the pattern.

You already learned this with MySQL.

The concepts are the same:

```text
Replication
Failover
Leader election
Replication lag
Read scaling
```

The technology-specific implementation differs.

---

# 22. MongoDB Is Not "Just a JSON Database"

This is another mental trap.

MongoDB provides:

```text
Document model
Indexing
Querying
Aggregation
Transactions
Replication
Sharding
```

The interesting part isn't:

> "It stores JSON."

The interesting part is:

> **The document model changes how we represent aggregates and access patterns, and the database is designed around that model.**

---

# 23. Aggregation

Suppose we have:

```text
Orders
{
    userId,
    total,
    status,
    createdAt
}
```

We want:

> Revenue per month.

A document database still needs to perform data processing.

MongoDB provides an aggregation framework conceptually like:

```text
Documents
    ↓
Filter
    ↓
Transform
    ↓
Group
    ↓
Aggregate
    ↓
Result
```

For example:

```text
Orders
   ↓
only completed orders
   ↓
group by month
   ↓
sum total
```

This means MongoDB isn't limited to:

```text
GET by ID
```

It can support more sophisticated querying and aggregation.

But there's still an architectural question:

> **Is this workload primarily operational document access, or are we turning the database into an analytics engine?**

That distinction matters at scale.

---

# 24. MongoDB's Sweet Spot

A useful mental model:

```text
MongoDB
   │
   ├── Data naturally represented as documents
   │
   ├── Nested/hierarchical structures
   │
   ├── Schema flexibility
   │
   ├── Access-pattern-driven modeling
   │
   ├── Horizontal scaling
   │
   └── High availability
```

Good examples can include:

```text
Product catalogs
Content management
User profiles
Event metadata
Configuration data
Rapidly evolving application domains
```

But remember: each workload still needs evaluation.

---

# 25. Where MongoDB Is Less Attractive

Suppose we're building:

```text
Banking ledger
```

with:

```text
Account
Transaction
Balance
Audit
Settlement
```

and the system heavily depends on:

```text
Complex relationships
Strong transactional invariants
Cross-entity consistency
Complex relational queries
```

A relational database is likely a more natural starting point.

MongoDB can support transactions, but that doesn't mean we should ignore the fundamental modeling tradeoff.

The question is:

> **Are we fighting the document model to recreate a relational system?**

If yes, that's a warning sign.

---

# 26. Another Warning Sign: Lots of Cross-Document Joins

Imagine you keep doing:

```text
Customer
   ↓
Orders
   ↓
Order Items
   ↓
Products
   ↓
Payments
   ↓
Shipments
```

and every request requires stitching together many independently stored documents.

At that point, ask:

> **Why are we using a document database?**

MongoDB does have mechanisms for joining/reference-style queries, but if relational joins are central to your workload, a relational database may be more natural.

---

# 27. MongoDB vs MySQL

Now let's make the comparison meaningful.

| Requirement                    | MySQL                          | MongoDB                                            |
| ------------------------------ | ------------------------------ | -------------------------------------------------- |
| Relational data                | Excellent                      | Possible, but not its core model                   |
| Complex relationships          | Excellent                      | Less natural                                       |
| Joins                          | Strong                         | Available, but not the primary modeling philosophy |
| Transactions                   | Strong                         | Supported                                          |
| Flexible nested data           | Possible                       | Natural                                            |
| Schema flexibility             | Lower                          | Higher                                             |
| Aggregate/document retrieval   | Possible                       | Natural                                            |
| Horizontal scaling             | Possible, increasingly complex | Core design capability                             |
| Strong relational constraints  | Strong                         | Different model                                    |
| Access-pattern-driven modeling | Important                      | Very important                                     |

The key difference:

```text
MySQL
→ model relationships

MongoDB
→ model aggregates/documents
```

That's the mental model.

---

# 28. A Concrete Decision

Suppose we're building **Instagram's user profile service**.

A profile contains:

```text
User
 ├── name
 ├── bio
 ├── profile picture
 ├── preferences
 ├── links
 └── flexible metadata
```

The dominant operation:

```text
GET /profile/{userId}
```

and most of the information is naturally retrieved together.

MongoDB becomes attractive.

Why?

```text
User ID
  ↓
one document
  ↓
profile data
```

Very natural.

---

# 29. Now Change the Requirement

Suppose instead we're designing:

> A financial reporting system that constantly asks questions across customers, accounts, transactions, branches, currencies, and time periods.

Now:

```text
Customer
    │
Account
    │
Transaction
    │
Branch
    │
Currency
```

and complex relationships and aggregations dominate.

A relational database becomes much more attractive.

Same broad concept:

```text
Data storage
```

Different workload:

```text
Different model
```

---

# 30. The Most Important MongoDB Tradeoff

Here's the tradeoff I want you to remember:

> **MongoDB makes it easier to model and retrieve related data together as a document, but you take on more responsibility for designing the document boundaries and access patterns correctly.**

With a relational database:

```text
Schema
   ↓
relationships are explicit
```

With MongoDB:

```text
Document design
   ↓
application access patterns strongly influence schema
```

That flexibility is powerful.

But it means poor modeling can hurt you badly.

---

# 31. A Poor MongoDB Design

Imagine:

```text
User
{
    orders: [
        ...all orders...
    ]
}
```

Then the user has:

```text
10,000 orders
```

Now:

```text
User document
     ↓
huge
     ↓
frequent updates
     ↓
expensive operations
```

We modeled based on:

> "Orders belong to users."

instead of:

> "How are orders actually accessed and how large can this relationship become?"

That's the wrong reasoning.

---

# 32. A Better Design

Separate them:

```text
User
{
    id: 42,
    ...
}

Order
{
    id: 123,
    userId: 42,
    ...
}
```

Now:

```text
GET /users/42
      ↓
User document

GET /users/42/orders
      ↓
Orders queried by userId
```

The model matches the access patterns.

---

# 33. The HLD Decision Framework

When you see MongoDB as a candidate, ask:

### 1. What is the natural aggregate?

```text
What data is normally retrieved together?
```

### 2. Is the relationship bounded?

```text
Can the nested data grow without bound?
```

### 3. What are the access patterns?

```text
Point lookup?
Range?
Filtering?
Aggregation?
```

### 4. How flexible is the schema?

```text
Stable?
Rapidly evolving?
Different shapes?
```

### 5. Do we need relational guarantees?

```text
Foreign keys?
Complex joins?
Cross-entity transactions?
```

### 6. How will the data distribute?

```text
What is the shard key?
Will it create hotspots?
```

### 7. What happens when the system grows?

```text
Document size?
Write throughput?
Shard distribution?
Indexes?
```

That's how I'd expect you to reason about MongoDB in an interview.

---

# 34. The MongoDB Mental Model

If you remember one sentence:

> **MongoDB is a document-oriented database that is particularly useful when application data naturally forms aggregates, nested structures are common, schema flexibility is valuable, and horizontal scaling is important.**

And the tradeoff:

> **You gain flexibility and natural aggregate-oriented access, but you give up some of the relational model's natural strengths around complex relationships, joins, and relational constraints.**

---

# 35. One Level Deeper: The Real Question

There's an even more important lesson here.

You might be tempted to ask:

> "Is MongoDB better than MySQL?"

That's the wrong question.

Ask:

> **"What is the natural unit of data in this system?"**

If it's:

```text
Customer
   ├── Accounts
   ├── Transactions
   └── Orders
```

and those relationships are central:

```text
Relational model
```

may be natural.

If it's:

```text
Product
   ├── variants
   ├── specifications
   ├── images
   └── metadata
```

and the product is usually retrieved as one aggregate:

```text
Document model
```

may be natural.

That is the real decision.

---

## Your Turn

Before we move on, I want to test whether the document model has actually clicked.

Imagine we're designing a **food-delivery application**.

An `Order` contains:

```text
orderId
restaurantId
customerId

items:
  - burger × 2
  - fries × 1
  - coke × 2

deliveryAddress:
  street
  city
  postalCode

pricing:
  subtotal
  deliveryFee
  discount
  total
```

And consider these facts:

- An order's items never belong to another order.
- The delivery address is the address used **for that specific order**.
- Pricing is specific to that order.
- An order can have anywhere from 1–20 items.
- We frequently retrieve the complete order when a customer opens order history.
- We don't want a customer's millions of historical orders embedded inside the customer document.

**Question:**

Would you **embed or reference**:

1. Order items
2. Delivery address
3. Pricing
4. Customer → Orders

And most importantly, **why?**

Don't just give me "embed/reference." Explain the reasoning using the ideas we've just developed.
