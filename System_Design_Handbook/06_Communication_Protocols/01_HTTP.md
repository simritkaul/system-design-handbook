# Module 6 — Communication Protocols

# Chapter 1 — HTTP

## Goal

Understand **why applications need a communication protocol**, why HTTP became the standard for the web, and the engineering decisions behind how two computers exchange requests and responses.

---

# 1. The Problem

Imagine you're building Amazon.

A customer opens the homepage.

The browser needs to ask:

> "Give me the homepage."

The server responds with HTML.

Then the browser asks:

> "Give me the CSS."

Then:

> "Give me the JavaScript."

Then:

> "Give me the product image."

Then:

> "Give me today's recommendations."

Every interaction between the browser and the server is simply two computers trying to exchange information.

But immediately, many questions appear.

- How does the browser ask for a resource?
- How does the server know what is being requested?
- How does the server indicate success or failure?
- How does it send images instead of text?
- How does it know whether the user wants to download a PDF or view a webpage?
- How do millions of browsers communicate with millions of servers using the same rules?

If every browser and every server invented its own communication format, nothing on the Internet would work together.

Computers need a **shared language**.

---

# 2. Why Existing Solutions Fail

In Module 5, we learned how data reaches another computer.

We learned:

- IP addresses identify machines.
- TCP provides reliable communication.
- TLS encrypts communication.
- DNS finds servers.
- Routers forward packets.

But notice something important.

TCP only answers:

> "How do bytes reliably move from one machine to another?"

It does **not** answer:

- What do those bytes mean?
- Which file is being requested?
- Is this a login request?
- Is this an image?
- Was the request successful?
- Is the user authorized?

Imagine receiving this over TCP:

```
ajsd9823ksjdf9283
```

What does it mean?

Nobody knows.

TCP transports bytes.

It doesn't define their meaning.

We need another layer.

---

# 3. The Big Idea

> **HTTP is a common language that lets clients and servers understand each other's requests and responses.**

TCP delivers the message.

HTTP defines what the message means.

---

# 4. Detailed Explanation

Let's build this idea gradually.

Suppose you walk into a restaurant.

You don't simply shout random words.

Instead, both you and the waiter already understand the rules.

You say:

"I'd like one pizza."

The waiter writes it down.

The kitchen prepares it.

The waiter returns with either:

- Pizza served
- Sorry, unavailable

Both sides understand the conversation because they follow the same protocol.

HTTP works exactly like this.

---

A browser (the client) sends a request.

The server processes it.

The server sends a response.

```
Browser
    │
    │ Request
    ▼
Server
    │
    │ Response
    ▼
Browser
```

Everything on the web is built around this simple pattern.

---

## HTTP is an Application Layer Protocol

Earlier we studied the networking layers.

```
Application Layer
        ↑
HTTP

Transport Layer
        ↑
TCP

Internet Layer
        ↑
IP

Physical Network
```

Each layer has a different responsibility.

TCP says:

> "I'll make sure your data arrives."

HTTP says:

> "Here's how we'll talk."

---

## Request–Response Model

HTTP follows a request-response model.

One side always starts.

```
Client
    │
    │ Request
    ▼
Server
    │
    │ Response
    ▼
Client
```

The server never randomly sends information first.

It waits.

Only after receiving a request does it respond.

This simple rule makes HTTP easy to understand and scalable.

---

## What Does an HTTP Request Contain?

Every request answers several questions.

### Which resource?

```
/products
```

or

```
/login
```

or

```
/images/logo.png
```

---

### What action?

Examples include:

- Fetch something
- Create something
- Update something
- Delete something

We'll study these actions in detail later when discussing HTTP methods.

---

### Additional information

Sometimes the client also includes:

- User identity
- Browser information
- Accepted language
- Authentication token
- Cookies
- Request body

These provide context so the server knows how to process the request.

---

## What Does the Server Return?

The response usually contains three things.

### Status

Was the request successful?

Examples:

```
200 OK
```

```
404 Not Found
```

```
500 Internal Server Error
```

We'll study status codes later in this chapter.

---

### Metadata

The server may describe:

- Content type
- Size
- Cache instructions
- Compression
- Expiration time

This helps the client handle the response correctly.

---

### Actual Data

The response body could contain:

- HTML
- JSON
- Image
- Video
- PDF
- CSS
- JavaScript

HTTP doesn't care what the data is.

It only defines how it's exchanged.

---

## A Simple Conversation

```
Browser

"Please give me /products"

↓

Server

"200 OK

Here is the product list."
```

Or:

```
Browser

"Please give me /xyz"

↓

Server

"404

I don't have that resource."
```

Very similar to asking a librarian for a book.

---

## HTTP Doesn't Store Conversations

This is one of HTTP's most important characteristics.

Every request is independent.

Imagine asking a librarian:

```
Give me Book A.
```

Five minutes later:

```
Give me Book B.
```

The librarian doesn't automatically remember who you are unless you identify yourself again.

HTTP works the same way.

Every request is treated as a brand-new request.

This property is called **statelessness**.

We'll explore it in depth shortly because it has huge architectural implications.

---

# 5. Key Concepts

## Client

The requester.

Usually:

- Browser
- Mobile app
- Backend service

---

## Server

The provider of data or functionality.

Examples:

- Web server
- API server
- Authentication server

---

## Resource

Anything that can be requested.

Examples:

```
/users

/products

/orders

/images/logo.png
```

---

## URI (Uniform Resource Identifier)

A resource needs an address.

```
https://shop.com/products/123
```

This identifies what the client wants.

---

## Headers

Headers carry additional information about the request or response.

Examples:

```
Who is sending this?

What format do I accept?

Can this response be cached?

How large is the response?
```

Think of headers as labels attached to a package.

The package contains the actual item.

The label tells the delivery company how to handle it.

---

## Body

The body contains the actual data.

Example:

```
Login credentials

Product details

Uploaded image

JSON response
```

---

# 6. HTTP Methods (High-Level Introduction)

Every request usually represents an action.

The most common methods are:

| Method | Meaning                  |
| ------ | ------------------------ |
| GET    | Retrieve data            |
| POST   | Create something         |
| PUT    | Replace something        |
| PATCH  | Update part of something |
| DELETE | Remove something         |

We'll revisit these in much greater depth in the REST chapter, where they become central to API design.

---

# 7. HTTP Status Codes (High-Level Introduction)

Instead of inventing custom success messages, HTTP defines standard response categories.

### 2xx — Success

```
200 OK

201 Created
```

---

### 3xx — Redirection

```
301 Moved Permanently

302 Found
```

---

### 4xx — Client Error

```
400 Bad Request

401 Unauthorized

403 Forbidden

404 Not Found
```

---

### 5xx — Server Error

```
500 Internal Server Error

503 Service Unavailable
```

This shared vocabulary allows every browser, mobile app, and server to understand each other consistently.

---

# 8. One Complete HTTP Exchange

Imagine opening a product page.

```
Browser

↓

GET /products/123

↓

Server

↓

Find product

↓

Return

200 OK

{
   Product Information
}
```

That single exchange is the foundation of nearly every web application.

---

# 9. Real-World Usage

Nearly every modern internet application uses HTTP in some form.

### Amazon

- Product pages
- Search
- Reviews
- Checkout
- Orders

---

### Netflix

- Homepage
- Movie details
- Recommendations
- User profiles

Streaming video itself often uses specialized protocols optimized for media delivery, but the surrounding application—such as browsing titles, logging in, and retrieving metadata—relies heavily on HTTP.

---

### Google

- Search requests
- Gmail
- Maps
- Drive

---

### GitHub

- Repository pages
- Pull requests
- Commits
- API calls

---

### Uber

- Booking requests
- User information
- Driver details
- Payment APIs

---

# 10. Where It Helps

HTTP provides:

- A universal communication language
- Interoperability between different systems
- Clear request-response semantics
- Standardized error handling
- Support for many content types
- Extensibility through headers

Most importantly, it allows independently built clients and servers to communicate without needing custom protocols.

---

# 11. Where It Doesn't Help

HTTP also has limitations.

It is not ideal when:

- The server needs to push data continuously without waiting for requests.
- Extremely low-latency bidirectional communication is required.
- Millions of tiny, continuous updates must flow efficiently.

For example:

- Live chat
- Multiplayer games
- Stock market feeds
- Real-time collaborative editing

These scenarios motivated protocols like WebSockets and Server-Sent Events, which we'll study later in this module.

---

# 12. Mental Model

## Restaurant Ordering

Imagine a restaurant.

```
Customer
     │
     │ Orders food
     ▼
Waiter
     │
     │ Takes order
     ▼
Kitchen
     │
     │ Prepares food
     ▼
Waiter
     │
     ▼
Customer
```

In this analogy:

- Customer → Client
- Waiter → HTTP protocol (the agreed way of communicating)
- Kitchen → Server
- Food → Response
- Menu item → Requested resource

The waiter doesn't cook the food (just as HTTP doesn't process business logic), and the kitchen doesn't decide how orders are phrased. The protocol simply ensures both sides understand each other.

---

# 13. Tradeoffs

## Advantages

- Simple and widely understood
- Standardized across the internet
- Works with many data formats
- Easy to extend with headers
- Decouples clients and servers
- Built on reliable transport (typically TCP)

## Disadvantages

- Request-response model can be inefficient for real-time updates.
- Statelessness means applications must manage user state separately (for example, using cookies or tokens).
- Repeated metadata in every request adds overhead.
- Each interaction generally requires the client to initiate communication.

---

# 14. Common Interview Questions

### Why do we need HTTP if TCP already exists?

TCP guarantees reliable delivery of bytes.

HTTP defines the structure and meaning of those bytes so that applications can understand each other.

---

### What is meant by HTTP being stateless?

The server does not automatically remember information from previous requests.

Each request contains the information needed to process it, or the application explicitly provides identifiers (such as cookies or tokens) to associate it with prior interactions.

---

### Can HTTP send images?

Yes.

HTTP is content-agnostic.

It can carry HTML, JSON, images, videos, PDFs, CSS, JavaScript, and many other formats. The metadata in the response tells the client how to interpret the content.

---

### Is HTTP responsible for encryption?

No.

Encryption is provided by TLS.

HTTP running over TLS becomes HTTPS, which we explored in Module 5.

---

### Does the server always respond?

Normally yes, but failures such as timeouts, network issues, or server crashes may prevent a successful response from reaching the client.

---

# 15. Before vs After Architecture

## Before (No Common Protocol)

```
Browser A  ----\
                \
Browser B -------> Server

Each browser speaks differently.

Server must understand every format.
```

This approach doesn't scale.

---

## After (HTTP)

```
Browser A
        \
Browser B \
           \
Mobile App ---> HTTP ---> Server
           /
Backend Service
```

Everyone follows the same communication rules, making independent development and interoperability possible.

---

# 16. Connections

Now that we know **how clients and servers communicate using HTTP**, a natural question arises.

HTTP tells us **how to send requests**, but it doesn't tell us **how to design an application's interface**.

For example:

- How should we represent users, products, or orders?
- Should creating a product use `/createProduct` or `/products`?
- How should updates and deletions be modeled?
- How can thousands of developers design APIs consistently?

HTTP provides the communication protocol.

We still need a design philosophy for building web APIs.

That leads us naturally to the next chapter:

> **Chapter 2 — REST APIs**

---

# 17. Key Takeaways

- HTTP is an application-layer communication protocol for the web.
- TCP transports data; HTTP defines the structure and meaning of that data.
- HTTP follows a client-initiated request-response model.
- A request typically includes a target resource, an action, metadata, and optionally a body.
- A response includes a status, metadata, and optionally data.
- HTTP is stateless, making it scalable but requiring separate mechanisms to maintain user sessions.
- Standard methods and status codes allow clients and servers from different vendors to interoperate seamlessly.
- HTTP is ideal for request-response interactions but less suited for continuous, real-time communication, motivating later protocols in this module.
