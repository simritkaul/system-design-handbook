# Chapter 3 — GraphQL

## Goal

Understand why REST APIs can become inefficient when different clients need different combinations of data, and how GraphQL allows clients to ask for **exactly the data they need**.

---

# 1. The Problem

Imagine you're building Netflix.

The homepage displays:

- User profile
- Continue watching
- Trending shows
- Recommended shows
- Genres

A mobile client may need:

```text
Movie Name
Thumbnail
Rating
```

But a web client may need:

```text
Movie Name
Thumbnail
Rating
Description
Cast
Genre
Trailer
```

And a TV application may need something else.

With REST, we might expose:

```text
GET /movies/123
```

The server decides what data that endpoint returns.

For example:

```json
{
  "id": 123,
  "name": "Example Movie",
  "description": "...",
  "rating": 8.5,
  "genre": "Drama",
  "cast": [...],
  "director": "...",
  "releaseDate": "...",
  "reviews": [...]
}
```

But what if the mobile screen only needs:

```text
name
thumbnail
rating
```

It receives much more data than necessary.

This is **over-fetching**.

---

Now consider the opposite problem.

A product page needs:

```text
Product
+
Reviews
+
Seller
+
Recommendations
```

The client might make:

```text
GET /products/123

GET /products/123/reviews

GET /sellers/456

GET /products/123/recommendations
```

One screen requires several API calls.

This is a form of **under-fetching**: one response does not contain enough data for the client, so additional requests are needed.

As applications grow and clients become more diverse, these problems become increasingly noticeable.

---

# 2. Why Existing Solutions Fail

REST is still perfectly useful.

The problem is not that REST is "bad."

The problem is that REST generally works like this:

```text
Client
   │
   │ Requests endpoint
   ▼
Server
   │
   │ Decides response structure
   ▼
Client
```

The server defines:

> "This is what `/products/123` returns."

But different clients may need different views of the same data.

```text
Mobile App ──► Small amount of data

Web App ─────► More detailed data

Admin Panel ─► Different data entirely
```

One solution is to create more endpoints.

```text
/products/123

/products/123/mobile

/products/123/details

/products/123/summary
```

But eventually the API starts growing around individual client requirements.

Another solution is to create one giant response.

But then clients receive unnecessary data.

We need a different idea.

---

# 3. The Big Idea

> **Instead of the server deciding exactly what every endpoint returns, let the client describe the data it wants.**

The client can effectively say:

> "I need the product's name, price, and seller name. Nothing else."

The server returns exactly that.

This is the central idea behind GraphQL.

---

# 4. Detailed Explanation

Imagine going to a restaurant.

REST is somewhat like ordering from predefined meals.

```text
Meal A
- Burger
- Fries
- Drink

Meal B
- Pizza
- Salad
- Dessert
```

You choose one of the options the restaurant has already defined.

GraphQL is more like saying:

> "I want a burger, no fries, and orange juice."

You specify exactly what you need.

---

## A GraphQL Query

Suppose we want information about a product.

The client might ask conceptually:

```graphql
{
  product(id: 123) {
    name
    price
    rating
  }
}
```

The important thing is not the syntax.

The important idea is:

```text
Client chooses fields
        │
        ▼
Server retrieves data
        │
        ▼
Server returns selected fields
```

The response might be:

```json
{
  "product": {
    "name": "Keyboard",
    "price": 3000,
    "rating": 4.5
  }
}
```

Nothing extra.

---

# 5. One Request Can Traverse Multiple Resources

This is where GraphQL becomes especially interesting.

Suppose a product page needs:

```text
Product
Seller
Reviews
```

In REST:

```text
Client
   │
   ├── GET /products/123
   │
   ├── GET /products/123/reviews
   │
   └── GET /sellers/456
```

With GraphQL, the client can describe the entire data requirement in one query.

Conceptually:

```graphql
{
  product(id: 123) {
    name
    price

    seller {
      name
      rating
    }

    reviews {
      rating
      comment
    }
  }
}
```

The client describes the **shape of the response it wants**.

---

## Before

```text
                Client
                   │
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
    Products    Reviews     Sellers
       API        API         API
```

Multiple API calls may be required.

---

## After

```text
Client
   │
   │ One data request
   ▼
GraphQL Layer
   │
   ├── Products
   │
   ├── Reviews
   │
   └── Sellers
```

The GraphQL layer coordinates the data gathering.

---

# 6. GraphQL Is Not a Database

This is a very important distinction.

When developers first see GraphQL, they sometimes imagine:

```text
Client
   │
   ▼
GraphQL
   │
   ▼
Database
```

But GraphQL does not have to directly query a database.

A GraphQL layer might combine data from:

```text
                GraphQL
                    │
      ┌─────────────┼─────────────┐
      ▼             ▼             ▼
Product Service  User Service  Review Service
      │             │             │
      ▼             ▼             ▼
 Database        Database       Database
```

GraphQL acts as a **data aggregation layer**.

It gives the client one interface even when the actual data comes from multiple backend systems.

---

# 7. The Schema

For the server to understand what clients are allowed to request, there must be a contract.

GraphQL uses a **schema**.

Conceptually:

```text
Product

- id
- name
- price
- rating
- seller
- reviews
```

The schema defines:

- What data exists
- What relationships exist
- What clients are allowed to request

Think of it as a menu.

The client can choose from the menu.

But it cannot ask for something that doesn't exist.

---

## Example

Imagine the schema allows:

```text
Product
    │
    ├── name
    ├── price
    ├── seller
    └── reviews
```

The client might request:

```text
Product
   │
   ├── name
   │
   └── seller
          │
          └── name
```

The server knows exactly how to resolve each piece.

---

# 8. Queries, Mutations, and Subscriptions

GraphQL commonly divides operations into three broad categories.

## Queries

Used to retrieve data.

Conceptually:

```text
"Give me this information."
```

Example:

```graphql
{
  product(id: 123) {
    name
    price
  }
}
```

---

## Mutations

Used to change data.

Conceptually:

```text
"Create, update, or change something."
```

For example:

```text
Create product

Update user

Place order
```

This is similar in purpose to the state-changing operations we discussed with REST.

---

## Subscriptions

Used for receiving ongoing updates.

Conceptually:

```text
Client subscribes

        │

Server sends updates when something changes
```

For example:

```text
New chat message

↓

Client receives update
```

This starts moving us beyond the simple request-response model.

We'll encounter this idea again when we study WebSockets and other real-time communication approaches.

---

# 9. How GraphQL Works Conceptually

Suppose the client sends:

```text
Give me:

Product
    ├── name
    ├── price
    └── seller
            └── name
```

The GraphQL server breaks the request into pieces.

```text
Client Query
      │
      ▼
GraphQL Server
      │
      ├── Get Product
      │
      └── Get Seller
```

Then it assembles the response.

```text
Product Data + Seller Data
            │
            ▼
      Combined Response
            │
            ▼
           Client
```

This is an important architectural difference.

The client thinks in terms of:

> "What data do I need?"

The GraphQL layer figures out:

> "Where do I get that data?"

---

# 10. The N+1 Problem

GraphQL introduces some interesting challenges.

Imagine requesting:

```text
100 Products

For each product:
    Seller
```

A naive implementation might do:

```text
Get 100 Products
```

Then:

```text
Get Seller for Product 1

Get Seller for Product 2

Get Seller for Product 3

...
```

Now we may have:

```text
1 query
+
100 additional queries
```

This is called the **N+1 problem**.

GraphQL makes it easy to ask for connected data.

But the backend must be designed carefully so that resolving each field doesn't create an explosion of database or network calls.

A better implementation might batch requests.

```text
Get Products

↓

Collect Seller IDs

↓

Get all required Sellers together
```

The lesson is important:

> Giving clients flexibility can increase complexity inside the backend.

---

# 11. Where GraphQL Helps

GraphQL is particularly useful when different clients need different data.

### Multiple client types

```text
Mobile

Web

Tablet

Smart TV
```

Each can request the fields it needs.

---

### Complex UI screens

Imagine a dashboard requiring:

```text
User Information
+
Notifications
+
Recent Orders
+
Recommendations
+
Analytics
```

Instead of several independent API calls, one query can describe the complete data requirement.

---

### Aggregating Multiple Services

Suppose data lives across:

```text
User Service

Order Service

Payment Service

Recommendation Service
```

A GraphQL layer can provide a unified interface.

```text
Client
   │
   ▼
GraphQL
   │
   ├── User Service
   ├── Order Service
   ├── Payment Service
   └── Recommendation Service
```

This can simplify the client considerably.

---

# 12. Where GraphQL Doesn't Help

GraphQL is not automatically better than REST.

## Simple CRUD APIs

Suppose you have:

```text
GET /users/123
```

and every client needs roughly the same user information.

REST may be simpler.

Adding GraphQL introduces:

- Schema design
- Resolvers
- Query planning
- Additional operational complexity

without solving a meaningful problem.

---

## Uncontrolled Queries

Clients can potentially request large or deeply nested responses.

For example:

```text
Users
 └── Orders
      └── Products
           └── Reviews
                └── Users
```

A poorly controlled query can generate enormous backend work.

The server needs safeguards.

For example, conceptually:

- Maximum query depth
- Complexity limits
- Pagination
- Rate limiting

---

## Caching Can Be More Complex

REST works naturally with HTTP caching.

For example:

```text
GET /products/123
```

can be cached by URL.

GraphQL often uses a single endpoint such as:

```text
/graphql
```

But different queries sent to that endpoint may request completely different data.

So caching requires more awareness of the query and its variables.

This does not mean GraphQL cannot be cached.

It means caching is usually less straightforward.

---

# 13. Real-World Usage

GraphQL is particularly attractive for organizations with:

- Many frontend clients
- Complex user interfaces
- Large numbers of backend services
- Rapidly evolving frontend data requirements

A common architecture looks like:

```text
                    Mobile
                       │
Web ───────────────────┤
                       ▼
                 GraphQL Gateway
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
    User Service   Product Service  Order Service
```

The GraphQL layer acts as a unified interface between clients and backend systems.

The client does not need to know where every piece of data lives.

---

# 14. Mental Model

## Restaurant Custom Order

REST:

```text
Choose Meal A
```

The restaurant decides what comes with it.

GraphQL:

```text
I want:

Burger
No fries
Orange juice
Extra cheese
```

The customer specifies exactly what they want.

But this flexibility means the kitchen has more work.

If thousands of customers create completely different combinations, the kitchen must handle that complexity efficiently.

That is GraphQL's central tradeoff.

> **More flexibility for clients often means more complexity for servers.**

---

# 15. Tradeoffs

## Advantages

- Clients can request exactly the data they need.
- Reduces over-fetching.
- Can reduce the number of client-server round trips.
- Provides a unified interface over multiple services.
- Works well for complex and evolving user interfaces.
- Schema provides a strongly defined contract.

## Disadvantages

- More backend complexity.
- Can create inefficient queries if poorly implemented.
- N+1 problems can cause performance issues.
- Caching is often more complex than simple REST resource caching.
- Requires query validation and complexity controls.
- May be unnecessary for simple APIs.

---

# 16. Common Interview Questions

## What problem does GraphQL solve?

GraphQL primarily addresses the problem of clients having different or complex data requirements.

It allows clients to request the specific fields and relationships they need rather than being limited to a fixed response shape for each endpoint.

---

## How is GraphQL different from REST?

A simplified comparison:

| REST                            | GraphQL                                |
| ------------------------------- | -------------------------------------- |
| Server defines response shape   | Client specifies required fields       |
| Usually many resource endpoints | Often one query interface              |
| HTTP caching is straightforward | Caching can require more awareness     |
| Simple for standard CRUD        | Flexible for complex data requirements |
| Multiple calls may be needed    | Multiple data sources can be combined  |

Neither is universally better.

The right question is:

> What problem does the application actually have?

---

## Does GraphQL replace REST?

No.

Many systems use both.

For example:

```text
External Clients
       │
       ▼
    GraphQL
       │
       ▼
Internal REST APIs
```

GraphQL may act as an aggregation layer while backend services continue communicating through REST or other mechanisms.

---

## What is the N+1 problem?

A request retrieves N items, then separately performs another lookup for each item.

For example:

```text
Get 100 products

+

Get seller for each product
```

This can result in 101 database or service calls instead of a small number of batched calls.

---

## How do you prevent expensive GraphQL queries?

Common approaches include:

- Query depth limits
- Query complexity limits
- Pagination
- Rate limiting
- Timeouts
- Batching related requests

---

## Is GraphQL always faster than REST?

No.

GraphQL may reduce client-server round trips, but resolving a complex query can create significant backend work.

Performance depends on:

- Query design
- Data fetching strategy
- Batching
- Caching
- Service architecture

GraphQL trades fixed server-defined responses for flexible client-defined queries.

That flexibility is not free.

---

# 17. Before vs After Architecture

## Before — Client Coordinates Everything

```text
                  Client
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
      Product    Reviews    Seller
       API         API        API
```

The client needs to understand:

- Which APIs exist
- What order to call them
- How to combine the results

---

## After — Data Aggregation Layer

```text
                  Client
                    │
                    │
              One Query
                    │
                    ▼
              GraphQL Layer
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
     Product      Reviews      Seller
     Service      Service      Service
```

The client focuses on:

> What data do I need?

The GraphQL layer coordinates:

> How do I get it?

---

# 18. Connections

GraphQL solves an important problem:

> **How can a client efficiently ask for exactly the data it needs?**

But REST and GraphQL still primarily follow a familiar pattern:

```text
Client asks

↓

Server responds

↓

Connection ends
```

Now imagine a chat application.

Alice sends Bob a message.

Does Bob's application repeatedly ask:

```text
Any new messages?

Any new messages?

Any new messages?
```

That is inefficient.

What we really want is:

```text
Alice sends message

        ↓

Server

        ↓

Bob immediately receives update
```

Now the problem changes.

We are no longer only asking:

> "How should the client request data?"

We are asking:

> **"How can the server send information to the client when something happens?"**

That leads naturally to our next chapter.

# Chapter 4 — WebSockets

---

# 19. Key Takeaways

- GraphQL allows clients to describe the shape of the data they need.
- It helps reduce over-fetching and under-fetching.
- A GraphQL layer can aggregate data from multiple backend services.
- GraphQL is not a database; it is an API query and data-fetching layer.
- Queries retrieve data, mutations change data, and subscriptions support ongoing updates.
- Flexibility for clients introduces complexity for the backend.
- Poorly implemented GraphQL systems can suffer from N+1 queries and expensive nested requests.
- REST is often simpler for straightforward APIs.
- GraphQL is especially useful for complex applications with multiple clients and diverse data requirements.
- The next limitation is more fundamental: both REST and typical GraphQL queries are primarily client-initiated request-response interactions.

**Next: WebSockets — persistent, bidirectional communication between client and server.**
