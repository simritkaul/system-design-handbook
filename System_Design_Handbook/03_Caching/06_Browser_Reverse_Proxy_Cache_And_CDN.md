# Module 3 — Chapter 6 — Browser Cache, Reverse Proxy Cache & CDN

> **Goal:** Understand the different layers of caching between the user and your application, why CDNs exist, and how they reduce latency and backend load.

---

# 1. The Problem

Imagine you're running Netflix.

A user in Delhi opens

```text id="drw2oh"
Stranger Things
```

Your servers are located in

```text id="mjlwmr"
Virginia, USA
```

Every request must travel

India

↓

USA

↓

India

Thousands of kilometers.

Even if your server is incredibly fast,

the speed of light still limits how quickly data can travel.

The farther the user,

the higher the latency.

---

# 2. The Big Idea

Instead of bringing

every user

to the data,

bring

the data

closer to the users.

This is exactly what

Content Delivery Networks

do.

---

# 3. The Layers of Caching

A modern web request may pass through several caches.

```text id="r8h9zs"
Browser Cache

↓

CDN

↓

Reverse Proxy Cache

↓

Application Cache (Redis)

↓

Database
```

Each layer serves requests before the next layer is contacted.

The closer the cache is to the user,

the lower the latency.

---

# 4. Browser Cache

The first cache

is already on

the user's computer.

Suppose you visit

```text id="u2v4es"
amazon.com
```

Your browser downloads

- Logo
- CSS
- JavaScript
- Fonts

The next page

uses

the same files.

Should it download them again?

No.

The browser already has them.

---

### Best For

- Images
- CSS
- JavaScript
- Fonts
- Static assets

---

### Advantages

- Fastest possible cache
- Zero network request (in many cases)
- Reduces bandwidth

---

# 5. Reverse Proxy Cache

Imagine

10 million users

request

the homepage.

Instead of

every request

reaching

the application,

place a cache

in front of it.

```text id="k7mtq4"
Users

↓

Reverse Proxy

↓

Application
```

Popular responses

are served

directly.

Examples include

NGINX

Varnish

HAProxy (with caching capabilities)

---

# 6. Content Delivery Network (CDN)

Now imagine

users

across

the world.

```text id="xvnt3z"
India

USA

Japan

Germany

Brazil
```

Should everyone

download

images

from

Virginia?

No.

Instead,

build servers

around the world.

These are called

**Edge Servers.**

---

# 7. How a CDN Works

Suppose

Netflix stores

its videos

in Object Storage.

```text id="9wft2k"
S3

↓

Video
```

A user in Delhi

requests

the movie.

The nearest CDN edge

checks

whether

it already has the video.

### Cache Hit

```text id="kof9l5"
Delhi Edge

↓

Video Found

↓

Return
```

Very fast.

---

### Cache Miss

```text id="5mrb79"
Delhi Edge

↓

No Video

↓

Fetch from Origin

↓

Store

↓

Return
```

Future users

benefit

from the cached copy.

---

# 8. Origin Server

The original server

that owns

the content

is called

the

**Origin.**

Example

```text id="b5gk9v"
S3 Bucket

↓

Origin

↓

CDN Edge

↓

User
```

The CDN stores

copies.

The origin

remains

the source of truth.

---

# 9. Edge Servers

Think of

CDN

as

hundreds

or

thousands

of mini caches

distributed globally.

Example

```text id="2ubjlwm"
Delhi

Singapore

London

New York

Sydney
```

Each location

is called

an

**Edge Location.**

---

# 10. Why Is This Powerful?

Without CDN

```text id="8rfd7j"
India

↓

USA

↓

India
```

Every request.

---

With CDN

```text id="yjlwmu"
India

↓

Delhi Edge
```

Huge reduction

in latency.

---

# 11. What Should Be Cached?

Excellent candidates:

- Images
- Videos
- CSS
- JavaScript
- Fonts
- PDFs
- Software downloads
- Product images

---

Poor candidates:

- Bank balances
- Personalized dashboards
- Live chat messages
- Frequently changing data

Although modern CDNs can cache some dynamic content under carefully controlled conditions, the classic use case is static or infrequently changing content.

---

# 12. Browser Cache vs CDN vs Redis

This is a favorite interview comparison.

| Browser Cache            | CDN                             | Redis                         |
| ------------------------ | ------------------------------- | ----------------------------- |
| On the user's device     | Near the user                   | Near the application          |
| Static assets            | Static & some cacheable content | Application data              |
| Fastest for that user    | Reduces geographic latency      | Reduces backend/database load |
| Managed by browser rules | Managed by CDN                  | Managed by application        |

Notice

they solve

different problems.

---

# 13. CDN vs Object Storage

Another common confusion.

Object Storage

stores

the original file.

CDN

stores

copies

closer

to users.

Example

```text id="6r1i3a"
S3

↓

Origin

↓

CloudFront

↓

User
```

One stores.

The other distributes.

---

# 14. Real Request Flow

Imagine opening YouTube.

```text id="tjlwm9"
Browser Cache

↓

CDN

↓

Reverse Proxy

↓

Redis

↓

Database

↓

Object Storage
```

Most requests

never reach

the database.

Many don't even reach

the application.

That's why YouTube can scale to billions of requests.

---

# 15. Mental Model

Imagine a popular textbook.

The publisher

owns

the master copy.

Libraries

around the world

keep

copies.

Students borrow

from

their nearest library,

not from

the publisher's warehouse.

Publisher

↓

Origin

Libraries

↓

CDN Edge Servers

Students

↓

Users

---

# 16. Tradeoffs

### Advantages

- Lower latency
- Lower bandwidth costs
- Reduced backend load
- Better user experience
- Improved scalability

---

### Disadvantages

- Cache invalidation across edge locations
- Extra infrastructure
- Cost
- Not all content is cacheable
- Dynamic content requires careful handling

---

# 17. Popular Technologies

### CDNs

- Cloudflare
- Amazon CloudFront
- Akamai
- Fastly

---

### Reverse Proxy

- NGINX
- Varnish

---

### Browser Cache

Built into every modern web browser using HTTP caching headers.

We'll study these technologies individually later.

---

# 18. Common Interview Questions

- What is a CDN?
- Why does Netflix use CDNs?
- Browser Cache vs CDN?
- CDN vs Redis?
- CDN vs Object Storage?
- What is an origin server?
- What is an edge server?
- Why aren't bank balances cached at the CDN?

---

# 19. Before vs After

### Without CDN

```text id="c2n7xe"
User (India)
      │
      ▼
Origin (USA)
```

Every request crosses continents.

---

### With CDN

```text id="cnd8qe"
User (India)
      │
      ▼
CDN Edge (Delhi)
      │
      ▼
Origin (USA)
```

Only cache misses travel to the origin.

Most requests stay local.

---

# 20. Connections

We've now completed

the entire

Caching Module.

Notice the progression:

- **Why cache?**
- **How to cache?**
- **How to evict?**
- **How to keep it fresh?**
- **How to scale the cache?**
- **How to move the cache closer to the user?**

That covers the major caching concepts you'll encounter in real systems and interviews.

---

# Key Takeaways

- Caching exists at multiple layers, not just inside your application.
- **Browser Cache** serves assets directly from the user's device.
- **Reverse Proxy Cache** reduces load on application servers.
- **CDNs** cache content at edge locations close to users.
- **Object Storage** stores the original content, while CDNs distribute cached copies globally.
- A modern web application typically uses multiple caching layers together for the best performance.

---

# 🎉 Module 3 Complete

I think this is actually one of the strongest modules in the handbook.

By now, you've gone from thinking:

> "Redis is a cache."

to understanding an entire caching ecosystem:

```
Browser Cache
        ↓
CDN
        ↓
Reverse Proxy
        ↓
Application Cache (Redis)
        ↓
Database
        ↓
Object Storage
```

Each layer has a distinct purpose, and together they dramatically reduce latency and backend load.

---
