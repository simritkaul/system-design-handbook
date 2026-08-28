# Chapter 2 — REST APIs

## Goal

Understand how we design APIs in a consistent way once HTTP gives us the basic ability to communicate.

HTTP answered:

> **How can two applications exchange requests and responses?**

REST answers a different question:

> **How should we organize and expose our application's functionality over HTTP?**

---

# 1. The Problem

Imagine a company has built an e-commerce application.

It has:

- Users
- Products
- Orders
- Payments
- Reviews
- Shopping carts

The frontend needs to communicate with the backend.

Without any design conventions, different developers might create APIs like this:

```text
/getUser?id=123

/createNewOrder

/updateUserProfile

/removeProduct?id=456

/getAllProductsForCategory
```

Another team might use:

```text
/user/fetch/123

/order/create

/product/delete/456
```

Another might use:

```text
/api?action=getUser&id=123
```

All of these can technically work.

HTTP does not stop us from designing APIs this way.

But now imagine Amazon has thousands of engineers and hundreds of services.

If everyone invents their own style:

- APIs become harder to understand.
- Clients must learn different conventions.
- Documentation becomes inconsistent.
- Tooling becomes harder.
- Developers spend more time figuring out _how_ to call an API.

We need a common way to think about API design.

---

# 2. Why HTTP Alone Isn't Enough

In the previous chapter, we learned that HTTP gives us:

- URLs
- Methods
- Headers
- Status codes
- Request bodies
- Response bodies

But HTTP itself does not tell us:

> What should `/users/123` mean?

It also doesn't tell us whether we should use:

```text
/createUser
```

or:

```text
/users
```

HTTP provides the **communication mechanism**.

We still need a way to organize our application interface.

That is where REST comes in.

---

# 3. The Big Idea

> **REST is an architectural style for designing APIs around resources and using standard HTTP semantics to operate on those resources.**

The key word is:

## Resource

Instead of thinking:

> "What functions does my backend have?"

REST encourages us to think:

> "What things exist in my system?"

For an e-commerce application:

```text
Users
Products
Orders
Reviews
Carts
```

These are resources.

---

# 4. Detailed Explanation

Let's start with a familiar example.

Imagine a library.

The library contains:

```text
Books
Authors
Members
Loans
```

If you want to interact with a book, you don't say:

> "Execute the retrieveBook function."

You identify the thing you care about.

```text
Book #123
```

Then you decide what you want to do with it.

- Read it
- Add it
- Update it
- Remove it

REST separates these two ideas.

### The URL identifies the resource.

```text
/books/123
```

### The HTTP method identifies the operation.

```text
GET    /books/123
```

means:

> Retrieve book 123.

```text
DELETE /books/123
```

means:

> Remove book 123.

Same resource.

Different action.

---

# 5. Resources and URLs

A REST API generally models resources through URLs.

Suppose we have users.

```text
/users
```

represents the collection.

A specific user:

```text
/users/123
```

Now suppose we have orders.

```text
/orders
```

A specific order:

```text
/orders/987
```

This gives the API a predictable structure.

```text
/users
/users/123

/products
/products/456

/orders
/orders/789
```

The URL describes **what we are talking about**.

The HTTP method describes **what we want to do**.

---

# 6. HTTP Methods Become API Operations

REST takes advantage of the semantics HTTP already provides.

## GET — Retrieve

```text
GET /products
```

Get all products.

```text
GET /products/123
```

Get one product.

---

## POST — Create

```text
POST /products
```

Request body:

```json
{
  "name": "Keyboard",
  "price": 3000
}
```

Conceptually:

> Create a new product.

---

## PUT — Replace

```text
PUT /products/123
```

Conceptually:

> Replace the representation of product 123.

---

## PATCH — Partially Update

Suppose only the price changes.

```text
PATCH /products/123
```

```json
{
  "price": 2500
}
```

Conceptually:

> Change part of this resource.

---

## DELETE — Remove

```text
DELETE /products/123
```

Conceptually:

> Remove product 123.

---

# 7. The Important Design Principle

Notice the difference.

### Function-oriented API

```text
/createProduct

/getProduct

/updateProduct

/deleteProduct
```

The action is embedded in the URL.

---

### Resource-oriented API

```text
POST   /products

GET    /products/123

PATCH  /products/123

DELETE /products/123
```

The URL identifies the resource.

The HTTP method expresses the operation.

This is the central intuition behind RESTful API design.

---

# 8. Collections vs Individual Resources

A useful pattern is:

```text
/users
```

for a collection.

And:

```text
/users/123
```

for an individual resource.

For example:

```text
GET /users
```

Retrieve users.

```text
POST /users
```

Create a user.

```text
GET /users/123
```

Retrieve one user.

```text
DELETE /users/123
```

Delete one user.

The structure remains consistent.

---

# 9. Relationships Between Resources

Real systems contain relationships.

A user may have many orders.

We can represent this naturally.

```text
/users/123/orders
```

Conceptually:

> Orders belonging to user 123.

Similarly:

```text
/products/456/reviews
```

Conceptually:

> Reviews associated with product 456.

This creates a hierarchy that reflects the domain.

However, this should not be taken too far.

Imagine:

```text
/users/123/orders/456/products/789/reviews/10/comments/5
```

The API is now becoming difficult to understand.

Deep nesting is usually a sign that we should reconsider the resource model.

Often this is simpler:

```text
/orders/456
```

or:

```text
/reviews/10/comments
```

The goal is clarity, not creating the longest possible URL.

---

# 10. REST Is More Than Pretty URLs

This is important.

Many developers think REST simply means:

> Use nouns in URLs.

That is only a small part of it.

REST is based on a broader architectural style.

The commonly discussed constraints include:

- Client-server separation
- Statelessness
- Uniform interface
- Cacheability
- Layered system
- Code on demand is optional

Let's understand the important ones.

---

# 11. Client–Server Separation

The client is responsible for things such as:

- User interface
- Displaying data
- Collecting user input

The server is responsible for things such as:

- Business logic
- Data access
- Authentication
- Processing requests

```text
Client
   │
   │ HTTP Request
   ▼
API Server
   │
   ▼
Business Logic
   │
   ▼
Database
```

This separation allows both sides to evolve independently.

For example:

```text
Web Application
        │
Mobile Application
        │
Partner Application
        │
        ▼
      REST API
        │
        ▼
      Backend
```

Multiple clients can use the same backend interface.

---

# 12. Statelessness

We encountered statelessness in the HTTP chapter.

REST embraces this idea.

Each request should contain enough information for the server to process it.

For example:

```text
GET /orders/123

Authorization: Bearer <token>
```

The server should not need to remember:

> "Which request did this user make five minutes ago?"

Instead:

```text
Request 1
    │
    ▼
Server

Request 2
    │
    ▼
Server

Request 3
    │
    ▼
Server
```

Each request can be handled independently.

This becomes extremely useful when we scale.

---

## Why Statelessness Helps Scaling

Imagine the server remembers every user's session locally.

```text
User A ───► Server 1

User B ───► Server 2

User C ───► Server 3
```

Now suppose User A's next request goes to Server 2.

Server 2 may not know anything about User A.

This creates a problem.

```text
             Load Balancer
            /      |      \
           ▼       ▼       ▼
         App 1   App 2   App 3
```

With stateless requests, any server can process any request.

```text
             Load Balancer
            /      |      \
           ▼       ▼       ▼
         App 1   App 2   App 3

Any request → Any server
```

This makes horizontal scaling much easier.

---

# 13. Uniform Interface

REST tries to create a predictable interface.

Once you understand:

```text
GET /products
```

you can reasonably guess:

```text
GET /users

GET /orders

GET /reviews
```

Similarly:

```text
DELETE /users/123
```

has a predictable meaning.

This reduces cognitive load.

Developers don't need to memorize an entirely different API style for every resource.

---

# 14. Representations

This is another important idea.

The server contains some internal state.

For example, a database row might look conceptually like:

```text
Product Table

id
name
price
inventory_count
internal_supplier_id
created_at
```

But the client doesn't necessarily receive all of that.

The server sends a **representation** of the resource.

```json
{
  "id": 123,
  "name": "Keyboard",
  "price": 3000
}
```

The API is not exposing the database directly.

It exposes a representation designed for clients.

This is an important engineering boundary.

```text
Database Model
      ≠
API Representation
```

The internal implementation can change without necessarily breaking clients.

---

# 15. Cacheability

REST also works naturally with HTTP caching.

Suppose:

```text
GET /products/123
```

returns product information.

The server may indicate:

> This response can be cached.

Then:

```text
Browser / CDN / Cache
          │
          ▼
      Product Data
```

Future requests may not need to reach the application server.

```text
Before

User
  │
  ▼
App Server
  │
  ▼
Database
```

↓

```text
After

User
  │
  ▼
Cache
  │
  ├── HIT ──► Response
  │
  └── MISS
        │
        ▼
    App Server
        │
        ▼
     Database
```

This connects directly to everything we learned in Module 3.

REST doesn't invent caching.

Instead, it fits naturally into HTTP's existing caching mechanisms.

---

# 16. REST Is an Architectural Style, Not a Protocol

This distinction matters.

HTTP is a protocol.

It defines communication rules.

REST is an architectural style.

It defines principles for organizing APIs.

You can technically use HTTP without following REST.

For example:

```text
POST /executeComplexBusinessOperation
```

is still a valid HTTP endpoint.

REST is not something the server enforces automatically.

It is a design approach.

---

# 17. Real-World Usage

REST-style APIs are common because they work well for many request-response interactions.

Examples include APIs involving:

- Users
- Products
- Orders
- Payments
- Profiles
- Content
- Administrative operations

For example, a conceptual GitHub-style API might expose:

```text
/repos
/repos/123

/repos/123/issues

/repos/123/pull-requests
```

An Uber-like system could conceptually expose:

```text
/rides
/rides/123

/drivers
/drivers/456
```

The important lesson is not to memorize exact endpoint structures.

The important lesson is:

> Model the things in your domain clearly and expose a consistent interface for interacting with them.

---

# 18. Where REST Helps

REST works particularly well when:

### You have clear resources

Examples:

- Users
- Products
- Orders
- Posts

---

### You have request-response interactions

```text
Client asks

↓

Server responds
```

---

### You need a simple public API

REST is easy to understand because it builds on HTTP concepts that developers already know.

---

### Multiple clients consume the same backend

```text
Web
 │
Mobile
 │
Third Party
 │
 ▼
REST API
 │
 ▼
Backend
```

---

### Caching is useful

HTTP's existing caching ecosystem can be leveraged.

---

# 19. Where REST Doesn't Help

REST is not perfect.

## Complex Data Requirements

Imagine a product page needs:

- Product
- Seller
- Reviews
- Recommendations
- Inventory
- Shipping estimate

The client may need several requests.

```text
GET /products/123

GET /products/123/reviews

GET /products/123/recommendations

GET /inventory/123

GET /shipping-estimate?product=123
```

This can lead to multiple round trips.

Later, we'll see why GraphQL emerged to address some of these problems.

---

## Real-Time Communication

REST generally follows:

```text
Request

↓

Response

↓

Done
```

But what if the server needs to continuously push updates?

Examples:

- Chat messages
- Live sports scores
- Collaborative documents

REST is not the natural solution.

This leads us later to:

- Long Polling
- Server-Sent Events
- WebSockets

---

## Operations Don't Always Fit CRUD

Not every business operation is naturally:

- Create
- Read
- Update
- Delete

Consider:

```text
Transfer money

Generate report

Checkout cart

Cancel subscription
```

Trying to force every business operation into a perfect CRUD-shaped resource can make an API unnatural.

REST is a useful design model, not a religion.

---

# 20. Mental Model

## Library Catalog

Imagine a library.

The catalog identifies things:

```text
/books
/authors
/members
```

A specific book:

```text
/books/123
```

Then your action determines what happens.

```text
Look at book → GET

Add book → POST

Replace book details → PUT

Change some details → PATCH

Remove book → DELETE
```

The **address identifies the thing**.

The **method identifies what you want to do with it**.

That is the simplest mental model for REST.

---

# 21. Tradeoffs

## Advantages

- Simple and easy to understand
- Built naturally on HTTP
- Clear separation between resources and operations
- Statelessness makes horizontal scaling easier
- Works well with HTTP caching
- Familiar to most developers
- Good interoperability between clients and services

## Disadvantages

- Can require multiple requests for complex screens
- May over-fetch or under-fetch data
- Not naturally designed for continuous real-time communication
- Complex business workflows don't always fit cleanly into CRUD
- Poor API design can still happen even when using REST

---

# 22. Common Interview Questions

## What is REST?

REST is an architectural style for designing networked applications around resources, a uniform interface, stateless communication, and other constraints.

In practice, REST APIs commonly use HTTP methods and URLs to interact with resources.

---

## Is REST a protocol?

No.

HTTP is a protocol.

REST is an architectural style.

---

## What is the difference between REST and HTTP?

HTTP defines how messages are exchanged.

REST defines a way to structure an application's interface using principles that commonly leverage HTTP semantics.

```text
HTTP

How do we communicate?

        ↓

REST

How should we organize the API?
```

---

## Why is statelessness useful?

Because requests can be handled independently.

This makes load balancing and horizontal scaling easier because requests do not need to be permanently tied to one application server.

---

## What is the difference between PUT and PATCH?

Conceptually:

```text
PUT
```

replaces the representation of a resource.

```text
PATCH
```

partially modifies it.

In practice, API behavior must be clearly documented because implementations can vary.

---

## Is every HTTP API a REST API?

No.

An API can use HTTP while not following REST principles.

For example:

```text
POST /performAction
```

is a valid HTTP API endpoint but is not necessarily resource-oriented REST design.

---

## Why can REST lead to over-fetching or under-fetching?

Because the server decides the shape of each endpoint's response.

A client might receive more data than it needs.

Or it might need multiple endpoints to collect all the data required for one screen.

This problem naturally motivates GraphQL.

---

# 23. Before vs After Architecture

## Before — Ad Hoc APIs

```text
Client
   │
   ├── /getUser
   │
   ├── /createOrder
   │
   ├── /updateProduct
   │
   └── /deleteReview
          │
          ▼
        Backend
```

Every endpoint can invent its own naming and behavior.

As the system grows:

```text
Hundreds of APIs
        │
        ▼
Inconsistent conventions
        │
        ▼
Higher cognitive load
```

---

## After — Resource-Oriented API

```text
                 Client
                    │
                    ▼

        ┌──────────────────────┐
        │      REST API        │
        └──────────────────────┘
                    │

     GET    /users/123
     POST   /orders
     GET    /products/456
     PATCH  /products/456
     DELETE /reviews/789

                    │
                    ▼
                 Backend
```

The API has a predictable structure.

Resources identify **what**.

Methods identify **what action**.

---

# 24. Connections

REST gave us a clean and predictable way to design APIs.

But now a new problem appears.

Imagine a mobile application's product screen.

It needs:

```text
Product Information
        +
Reviews
        +
Seller Information
        +
Recommendations
        +
Inventory
```

With a REST API, the client may need several requests.

```text
Mobile App
    │
    ├── GET /products/123
    │
    ├── GET /products/123/reviews
    │
    ├── GET /products/123/recommendations
    │
    └── GET /inventory/123
```

Or the backend may create a very large endpoint that returns everything.

But another client may need only:

```text
Product Name
Price
```

Now we face two problems:

> **Over-fetching** — receiving more data than needed.

and:

> **Under-fetching** — needing additional requests to get everything needed.

So the next question becomes:

> **What if the client could ask for exactly the data it needs?**

That leads naturally to:

# Chapter 3 — GraphQL

---

# 25. Key Takeaways

- REST is an architectural style, not a communication protocol.
- HTTP tells applications **how to communicate**; REST provides a way to **organize APIs**.
- REST encourages modeling the domain around resources such as users, products, and orders.
- URLs generally identify resources, while HTTP methods express operations.
- Statelessness allows requests to be handled independently, making horizontal scaling easier.
- A REST API exposes representations of resources rather than directly exposing internal database models.
- REST works naturally with HTTP caching.
- REST is useful for many request-response APIs but is not ideal for every problem.
- Complex client data requirements can cause over-fetching and under-fetching.
- That limitation naturally motivates the next architectural approach: **GraphQL**.
