# Chapter 6 — HTTP Request Lifecycle

---

# Goal

Understand the complete journey of an HTTP request—from the moment a user enters a URL until the webpage is rendered—and see how all the networking concepts we've learned fit together.

---

# 1. The Problem

Imagine you open your browser and type:

```text
https://www.amazon.com
```

You press **Enter**.

Less than a second later, Amazon's homepage appears.

Seems simple.

But under the hood, dozens of things happen.

Questions immediately arise:

- How does the browser know where amazon.com is?
- How does it find Amazon's server?
- How does it establish a secure connection?
- How does the request travel across the Internet?
- How does Amazon know which server should handle it?
- How does the browser know how to display the page?

This single click involves almost everything we've learned so far.

---

# 2. Why Existing Solutions Fail

Until now, we've studied each networking concept separately.

We learned:

- IP addresses
- Ports
- TCP
- TLS
- Routers
- DNS (briefly in Module 4)

But in reality, users don't experience these concepts individually.

They experience:

> "I clicked a link."

The real engineering challenge is understanding **how all these pieces work together**.

This chapter connects everything into one continuous flow.

---

# 3. The Big Idea

> **An HTTP request is not a single action. It is a sequence of networking steps that work together to deliver data from a server to a client.**

Understanding this sequence is one of the most valuable mental models in system design.

---

# 4. Detailed Explanation

Let's follow one request from start to finish.

---

# Step 1 — User Enters a URL

Suppose you type:

```text
https://www.amazon.com
```

Notice something.

You didn't type:

```text
54.239.xxx.xxx
```

Humans remember names.

Computers communicate using IP addresses.

Someone must translate the name into an IP.

---

# Step 2 — DNS Lookup

The browser asks:

> "What is the IP address for [www.amazon.com](http://www.amazon.com)?"

This request goes to the DNS system.

```text
Browser

↓

DNS

↓

amazon.com

↓

54.x.x.x
```

Now the browser knows where to send the request.

_(We already studied DNS conceptually in Module 4, so we won't repeat its internals here.)_

---

# Step 3 — Is the Server Already Connected?

Before opening a new connection, the browser checks:

> "Do I already have an open connection to this server?"

If yes:

Reuse it.

If not:

Create a new connection.

Why?

Creating connections is expensive.

We'll dedicate the next two chapters to:

- HTTP Keep-Alive
- Connection Pooling

because they're important performance optimizations.

---

# Step 4 — TCP Connection

Suppose no existing connection is available.

The browser creates one.

Conceptually:

```text
Browser

↓

Can we communicate?

↓

Amazon Server

↓

Yes
```

This is called establishing a TCP connection.

Internally, this happens using the TCP three-way handshake.

For HLD purposes, remember:

Before reliable communication begins, both sides establish a connection.

---

# Step 5 — TLS Handshake

The URL begins with:

```text
https://
```

That means secure communication is required.

The browser and server now:

- Verify identities.
- Exchange certificates.
- Agree on encryption keys.
- Create a secure channel.

```text
Browser

↓

TLS Handshake

↓

Secure Connection Established
```

After this point:

Everything sent between browser and server is encrypted.

---

# Step 6 — Browser Creates the HTTP Request

Now the browser finally creates an HTTP request.

For example:

```http
GET / HTTP/1.1
Host: www.amazon.com
```

This request includes many additional headers.

Example:

```text
User-Agent

Accept

Cookies

Authorization

Accept-Language
```

We'll study HTTP itself in Module 6.

For now, think of the request as:

> "Please send me this resource."

---

# Step 7 — Data Moves Down the Networking Stack

Remember the TCP/IP model.

The browser created an HTTP request.

Now each networking layer contributes something.

```text
Application

↓

Transport

↓

Internet

↓

Network Access
```

---

## Application Layer

Creates:

```text
GET /
```

---

## Transport Layer

TCP:

- Splits data if necessary.
- Adds sequence numbers.
- Prepares reliable delivery.

---

## Internet Layer

Adds:

- Source IP
- Destination IP

```text
From:

192.168.1.5

To:

54.x.x.x
```

---

## Network Access Layer

Converts everything into signals suitable for:

- Wi-Fi
- Ethernet
- Cellular

The request begins traveling across the network.

---

# Step 8 — The Internet Routes the Request

The request now travels through many devices.

```text
Browser

↓

Wi-Fi Router

↓

ISP

↓

Internet Routers

↓

Amazon Data Center
```

Every router asks:

> "Where should this packet go next?"

Eventually, the packet reaches Amazon.

---

# Step 9 — Load Balancer Receives the Request

Remember Module 4.

The request doesn't usually go directly to an application server.

Instead:

```text
Internet

↓

Load Balancer

↓

App Server
```

The load balancer chooses an available server.

This allows Amazon to scale to millions of users.

---

# Step 10 — Application Processes the Request

The selected server receives:

```http
GET /
```

The server may now:

- Authenticate the user.
- Read cookies.
- Query databases.
- Read Redis cache.
- Call microservices.
- Generate HTML.
- Return JSON.
- Log analytics.

Everything we've learned in earlier modules now comes into play.

Example:

```text
App Server

↓

Redis

↓

Database

↓

Search Engine

↓

Recommendation Service
```

The server constructs a response.

---

# Step 11 — HTTP Response

Example:

```http
HTTP/1.1 200 OK
```

Along with:

- HTML
- JSON
- Images
- CSS
- JavaScript

The response travels back through the exact same networking layers.

---

# Step 12 — Browser Receives the Response

The browser:

- Verifies TCP delivery.
- Decrypts TLS.
- Parses HTTP.

Now it has the webpage.

But we're not done yet.

---

# Step 13 — Browser Parses HTML

Suppose the HTML contains:

```html
<img>

<link>

<script>
```

The browser realizes:

"I need more files."

So it creates additional HTTP requests.

Example:

```text
index.html

↓

styles.css

↓

app.js

↓

logo.png

↓

fonts
```

A single webpage often results in **dozens or even hundreds of HTTP requests**.

---

# Step 14 — Browser Renders the Page

Finally:

The browser:

- Parses HTML
- Parses CSS
- Executes JavaScript
- Builds the page
- Displays it

The user sees:

Amazon's homepage.

---

# Entire Journey

Let's put everything together.

```text
User Types URL
        │
        ▼
DNS Lookup
        │
        ▼
TCP Connection
        │
        ▼
TLS Handshake
        │
        ▼
HTTP Request Created
        │
        ▼
Network Transmission
        │
        ▼
Routers
        │
        ▼
Load Balancer
        │
        ▼
Application Server
        │
        ▼
Cache / Database / Services
        │
        ▼
HTTP Response
        │
        ▼
Browser Renders Page
```

Notice how every chapter we've studied contributes one piece of this journey.

---

# Where Does Caching Fit?

Module 3 wasn't separate from networking.

It fits naturally into the lifecycle.

Example:

```text
Browser

↓

Browser Cache

↓

CDN

↓

Reverse Proxy Cache

↓

Redis

↓

Database
```

Each cache prevents unnecessary work.

The browser may never even reach the application server.

---

# Where Does DNS Fit?

Module 4 also fits naturally.

```text
URL

↓

DNS

↓

IP Address
```

Without DNS:

The browser wouldn't know where to send the request.

---

# Where Does the API Gateway Fit?

Microservice architectures may look like:

```text
Browser

↓

Load Balancer

↓

API Gateway

↓

Orders Service

↓

Inventory Service

↓

Payment Service
```

Again, every module we've already studied becomes part of one request.

---

# 5. Types / Variations

Not every HTTP request follows exactly the same path.

### Static Content

```text
Browser

↓

CDN

↓

Response
```

The application server may never be involved.

---

### Dynamic Content

```text
Browser

↓

Load Balancer

↓

Application

↓

Database
```

Requires backend processing.

---

### Cached API

```text
Browser

↓

API

↓

Redis

↓

Response
```

The database isn't queried.

---

### Cache Miss

```text
Browser

↓

API

↓

Redis

↓

Database

↓

Redis Updated

↓

Response
```

---

# 6. Real-World Usage

### Amazon

A homepage request may involve:

- CDN for images.
- API Gateway.
- Authentication service.
- Recommendation service.
- Inventory service.
- Redis cache.
- Product database.

Even though the user perceives it as one page load, many internal services collaborate to generate the response.

---

### Netflix

Streaming begins with an HTTP request for metadata. Video segments are then fetched repeatedly, often from nearby CDN servers rather than the origin infrastructure.

---

### Google Search

A search request passes through global load balancers, frontend servers, ranking systems, indexing services, and caching layers before the results page is returned.

---

# 7. Where It Helps

Understanding the request lifecycle helps you:

- Debug slow requests.
- Identify latency bottlenecks.
- Decide where caching is most effective.
- Understand browser behavior.
- Design scalable web applications.
- Explain end-to-end request flow in interviews.

---

# 8. Where It Doesn't Help

The HTTP request lifecycle describes a typical request-response flow.

It does not explain:

- Long-lived connections (WebSockets).
- Streaming protocols.
- Message queues.
- Event-driven systems.
- Peer-to-peer communication.

Those topics will be covered in later modules.

---

# 9. Mental Model

Imagine ordering food from a restaurant.

```text
You
 │
Host
 │
Waiter
 │
Kitchen
 │
Chef
 │
Food
 │
Waiter
 │
You
```

Now map this to networking:

- **You** → Browser.
- **Restaurant address** → DNS.
- **Road** → Internet.
- **Host** → Load Balancer.
- **Kitchen** → Application Server.
- **Pantry** → Cache.
- **Storage Room** → Database.
- **Meal** → HTTP Response.

You only experience receiving the meal, but many coordinated steps happen behind the scenes.

---

# 10. Tradeoffs

## Advantages

- Modular architecture allows each component to specialize.
- Caching can eliminate unnecessary backend work.
- Load balancers improve scalability and availability.
- Layered networking makes the system easier to evolve.
- Clear separation of responsibilities simplifies debugging.

## Disadvantages

- Each additional step introduces some latency.
- More infrastructure means more operational complexity.
- Failures in DNS, TLS, load balancers, or backend services can affect the request.
- Optimizing one stage may simply expose a bottleneck in another.

---

# 11. Common Interview Questions

### Walk me through what happens when you type `https://google.com` into your browser.

A strong answer should mention, in order:

1. DNS resolves the domain to an IP address.
2. A TCP connection is established (or an existing one is reused).
3. A TLS handshake occurs for HTTPS.
4. The browser sends an HTTP request.
5. Routers forward packets across the Internet.
6. A load balancer routes the request to an application server.
7. The server processes the request, potentially using caches, databases, and other services.
8. An HTTP response is returned.
9. The browser parses the response and renders the page.

---

### Why does a webpage trigger many HTTP requests?

The initial HTML often references additional resources such as CSS, JavaScript, images, fonts, and API endpoints. Each typically requires its own request.

---

### Where does DNS occur?

Before the browser can contact the server, it must translate the domain name into an IP address.

---

### Where does TLS fit?

TLS is negotiated after the transport connection is established and before sensitive HTTP data is exchanged.

---

### Where does caching reduce latency?

Caching can occur at multiple levels:

- Browser cache.
- CDN.
- Reverse proxy.
- Application cache (such as Redis).

Each successful cache hit avoids work further down the request path.

---

# 12. Before vs After Architecture

### Conceptual View

```text
Browser
    │
Server
```

A useful simplification, but it hides most of the system.

↓

### Real Request Lifecycle

```text
Browser
     │
DNS
     │
TCP
     │
TLS
     │
Internet
     │
Load Balancer
     │
API Gateway
     │
Application Server
     │
Redis
     │
Database
     │
Response
     │
Browser Renders
```

This is much closer to what happens in a modern distributed system.

---

# 13. Connections

We've now seen the complete lifecycle of an HTTP request.

One detail in that journey deserves special attention.

Notice that before sending the request, the browser checked:

> **"Do I already have an open connection to this server?"**

Why?

Because creating a new TCP connection—and performing a TLS handshake—for every single request would be expensive.

Imagine a webpage that loads:

- 1 HTML document
- 10 JavaScript files
- 8 CSS files
- 50 images

Would we really establish **69 separate TCP connections**?

That would waste time and network resources.

The next chapter explores the first major optimization modern web systems use to avoid this overhead:

**HTTP Keep-Alive**, which allows multiple requests to reuse the same connection instead of creating a new one every time.
