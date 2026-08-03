# Chapter 3 — Reverse Proxy

---

# Goal

Understand how a reverse proxy acts as an intelligent gateway in front of application servers, providing features like request routing, caching, SSL termination, compression, and security.

---

# The Problem

In the previous chapter, we introduced a load balancer.

Our architecture now looks like this:

```text
                Users
                   │
                   ▼
          +------------------+
          | Load Balancer    |
          +------------------+
            │      │      │
            ▼      ▼      ▼
         App1    App2    App3
```

This solves one major problem:

- Traffic is distributed across multiple servers.

But as systems grow, new requirements begin to appear.

Let's imagine we're building Instagram.

Users request many different resources:

```text
GET /

GET /profile

GET /posts

GET /images/logo.png

GET /videos/123

GET /css/main.css
```

Should every one of these requests reach the application servers?

Not really.

Some requests don't even require the application.

For example:

```text
/images/logo.png
```

This image rarely changes.

Yet every request currently travels all the way to an application server.

```text
User

↓

Load Balancer

↓

Application

↓

Read image

↓

Return image
```

The application is doing unnecessary work.

---

Now imagine another problem.

Every request arrives over HTTPS.

```text
https://instagram.com
```

Decrypting HTTPS is computationally expensive.

Should every application server perform encryption and decryption independently?

That would waste CPU resources.

---

Another issue.

Our application servers currently have public IP addresses.

Anyone on the Internet can try to connect directly to them.

```text
Internet

↓

App Server
```

That isn't ideal from a security perspective.

We'd rather expose only one public endpoint.

---

One more problem.

Suppose we now have multiple services.

```text
Images

Videos

Search

API
```

How should requests be routed?

```text
/images/*

↓

Image Service

/api/*

↓

API Service

/search/*

↓

Search Service
```

The load balancer _can_ perform some routing, but that's not really its primary responsibility.

We need something smarter.

---

# Why Existing Solutions Fail

### DNS

DNS only answers:

> "What IP address should I connect to?"

Once it returns an IP address, its job is finished.

DNS cannot:

- Cache images
- Compress responses
- Terminate HTTPS
- Filter malicious requests
- Route based on URLs

---

### Load Balancer

A load balancer answers:

> "Which backend server should receive this request?"

That's its primary job.

It is not designed to become a full application gateway.

As systems become larger, we need another layer that understands HTTP itself.

---

# The Big Idea

A **Reverse Proxy** is an intelligent server that sits in front of your application servers and handles many common request-processing tasks before forwarding requests to the backend.

Think of it as a receptionist for your backend systems.

---

# Why is it called "Reverse" Proxy?

Most people have heard of a proxy server.

Let's understand the difference.

---

## Forward Proxy

A forward proxy represents the **client**.

```text
User

↓

Forward Proxy

↓

Internet
```

The destination server often doesn't even know who the original client was.

Forward proxies are commonly used for:

- Privacy
- Content filtering
- Corporate internet access
- VPN-like behavior

The proxy is helping the client.

---

## Reverse Proxy

A reverse proxy represents the **server**.

```text
Users

↓

Reverse Proxy

↓

Application Servers
```

Clients don't know which backend server actually handled the request.

The reverse proxy is helping the servers.

Hence the name:

**Forward Proxy → protects clients**

**Reverse Proxy → protects servers**

---

# Basic Architecture

Without a reverse proxy:

```text
Users

↓

Application Servers
```

Every application server is exposed publicly.

---

With a reverse proxy:

```text
                Users
                   │
                   ▼
          +----------------+
          | Reverse Proxy  |
          +----------------+
             │    │    │
             ▼    ▼    ▼
           App1 App2 App3
```

Users only know about the reverse proxy.

The application servers can now live in a private network.

---

# What Does a Reverse Proxy Actually Do?

A reverse proxy performs many different responsibilities.

Think of it as a collection of useful capabilities rather than a single feature.

---

# 1. Hide Backend Servers

Instead of exposing every application server:

```text
Internet

↓

App1

App2

App3
```

Only expose the reverse proxy.

```text
Internet

↓

Reverse Proxy

↓

Private Servers
```

This provides:

- Better security
- Easier infrastructure changes
- Cleaner architecture

Backend IP addresses can change without affecting clients.

---

# 2. Request Routing

Suppose our company has multiple services.

```text
User Requests

/api/users

/images/logo.png

/videos/movie.mp4
```

Instead of every request going to the same application:

```text
                Reverse Proxy

                     │

     ┌───────────────┼───────────────┐

     ▼               ▼               ▼

Image Service   API Service   Video Service
```

Routing rules might look like:

```text
/images/*

↓

Image Service

/api/*

↓

API Service

/videos/*

↓

Video Service
```

This is called **path-based routing**.

---

It can also route based on:

- Domain name

```text
api.company.com

↓

API Servers

admin.company.com

↓

Admin Servers
```

called **host-based routing**.

---

# 3. SSL/TLS Termination

Earlier we mentioned HTTPS.

When a browser connects:

```text
HTTPS
```

the data is encrypted.

Someone has to decrypt it.

Instead of every backend server doing this:

```text
Users

↓

HTTPS

↓

App1

App2

App3
```

The reverse proxy handles encryption once.

```text
Users

↓

HTTPS

↓

Reverse Proxy

↓

HTTP

↓

Application Servers
```

This is called **SSL/TLS Termination**.

Benefits:

- Simpler backend servers
- Lower CPU usage
- Easier certificate management
- Centralized HTTPS configuration

> **Note:** In highly secure environments, traffic between the reverse proxy and backend servers may also remain encrypted (HTTPS). The key idea is that the reverse proxy centralizes certificate handling, even if encryption continues internally.

---

# 4. Compression

Suppose an API returns:

```json
{
  "posts": ...
}
```

The response is:

```text
2 MB
```

The reverse proxy can compress it before sending it.

```text
2 MB

↓

Compression

↓

250 KB
```

Benefits:

- Faster downloads
- Lower bandwidth usage
- Better mobile performance

Application servers don't need to implement compression themselves.

---

# 5. Caching

Suppose millions of users request:

```text
/logo.png
```

Without caching:

```text
Every Request

↓

Application

↓

Disk

↓

Return Image
```

The application repeats identical work.

Instead:

```text
Users

↓

Reverse Proxy Cache

↓

Application (only on cache miss)
```

Future requests are served directly from the proxy.

We already studied caching extensively.

A reverse proxy is simply another place where caching can happen.

---

# 6. Security

Since every request passes through the reverse proxy first, it can reject suspicious traffic.

Examples:

- Block malicious IPs
- Limit request rates
- Filter invalid requests
- Restrict access to internal endpoints

Instead of every application implementing these checks, they are centralized.

---

# 7. Header Management

Applications often need useful information like:

- Original client IP
- Protocol (HTTP vs HTTPS)
- Request ID
- Authentication headers

A reverse proxy can add standardized headers before forwarding the request.

This makes backend services simpler and more consistent.

---

# Reverse Proxy vs Load Balancer

This is one of the most common interview questions.

The two have overlapping capabilities, which causes confusion.

The easiest way to understand them is through their primary responsibilities.

---

## Load Balancer

Primary goal:

> Distribute traffic across multiple backend servers.

Questions it answers:

- Which server should receive this request?
- Is a server healthy?
- Which routing algorithm should I use?

---

## Reverse Proxy

Primary goal:

> Process and manage incoming HTTP requests before they reach backend services.

Questions it answers:

- Should this response be cached?
- Should HTTPS be terminated here?
- Which service matches this URL?
- Should this request be compressed?
- Should this request be blocked?

---

Many modern products (such as NGINX, HAProxy, Envoy, and cloud load balancers) can perform **both** roles.

That doesn't mean the concepts are identical.

Think of it this way:

- **Load Balancing** is a responsibility.
- **Reverse Proxying** is a responsibility.

One piece of software may implement one, the other, or both.

---

# Real-World Usage

## Netflix

Uses reverse proxies to terminate TLS, route traffic, and protect backend services.

---

## YouTube

Routes requests for videos, thumbnails, APIs, and static assets to different backend systems.

---

## Amazon

Caches static content, manages HTTPS, and forwards requests to different internal services.

---

## GitHub

Uses reverse proxies to securely expose its web infrastructure while hiding internal services.

---

# Where It Helps

Reverse proxies are valuable because they:

- Hide backend infrastructure.
- Centralize HTTPS management.
- Cache static content.
- Compress responses.
- Route requests to different services.
- Improve security by filtering traffic.
- Simplify backend applications.

---

# Where It Doesn't Help

A reverse proxy cannot:

- Replace databases.
- Replace application business logic.
- Replace asynchronous messaging.
- Fix slow SQL queries.
- Scale CPU-intensive application code by itself.

It improves request handling but does not replace the application.

---

# Mental Model

Imagine visiting a large corporate office.

You don't walk directly into different departments.

Instead, you first meet the receptionist.

```text
Visitors

↓

Reception Desk

↓

Engineering

Finance

HR

Legal
```

The receptionist:

- Checks who you are.
- Gives you a visitor badge.
- Directs you to the correct department.
- Blocks unauthorized visitors.
- Answers simple questions directly.

The departments focus only on their actual work.

The receptionist is the reverse proxy.

---

# Tradeoffs

## Advantages

- Hides internal infrastructure from the public.
- Centralizes HTTPS certificate management.
- Enables caching and compression.
- Supports flexible routing based on paths or hostnames.
- Improves security through request filtering and rate limiting.
- Reduces duplicated infrastructure logic across applications.

## Disadvantages

- Introduces another component that must be deployed and monitored.
- Adds a small amount of latency because every request passes through it.
- Can become a bottleneck if not scaled properly.
- Misconfiguration can affect all backend services simultaneously.

---

# Common Interview Questions

### Why do we need a reverse proxy if we already have a load balancer?

A load balancer's primary job is distributing traffic across servers. A reverse proxy focuses on processing HTTP requests—such as TLS termination, caching, compression, security, and URL-based routing. Many modern systems combine both roles in one product.

---

### Why terminate HTTPS at the reverse proxy?

It centralizes certificate management and reduces the amount of TLS-related work each backend server must perform.

---

### Why cache static files at the reverse proxy?

Static assets change infrequently. Serving them directly from the proxy reduces application load and improves response times.

---

### What is path-based routing?

Routing requests based on the URL path.

Example:

```text
/images/* → Image Service

/api/* → API Service
```

---

### Why hide backend servers?

It improves security, allows infrastructure changes without affecting clients, and keeps internal networks private.

---

# Before vs After Architecture

## Before

```text
                 Users
                    │
                    ▼
            +----------------+
            | Load Balancer  |
            +----------------+
              │      │      │
              ▼      ▼      ▼
            App1   App2   App3
```

Every request reaches an application server, even for static files or common infrastructure tasks.

---

## After

```text
                 Users
                    │
                    ▼
            +----------------+
            | Reverse Proxy  |
            +----------------+
                    │
                    ▼
            +----------------+
            | Load Balancer  |
            +----------------+
              │      │      │
              ▼      ▼      ▼
            App1   App2   App3
```

The reverse proxy handles cross-cutting concerns like TLS, caching, compression, security, and routing, while the load balancer focuses on distributing traffic among healthy backend servers.

> **Note:** In many real-world deployments, the reverse proxy and load balancer are implemented by the same software or cloud service. The diagram separates them to illustrate their conceptual responsibilities.

---

# Connections

Our architecture is becoming increasingly sophisticated:

- DNS tells users **where** to connect.
- The Load Balancer decides **which server** should handle the request.
- The Reverse Proxy prepares, filters, secures, and routes the request before it reaches backend services.

But modern applications often consist of **dozens or hundreds of microservices**.

Soon, new challenges emerge:

- Authentication and authorization
- API versioning
- Rate limiting
- Request transformation
- Aggregating responses from multiple services
- Developer-facing APIs

These responsibilities go beyond a traditional reverse proxy.

They are handled by a specialized component built specifically for APIs:

> **Chapter 4 — API Gateway**
