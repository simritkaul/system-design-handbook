# Chapter 4 — API Gateway

---

# Goal

Understand why microservice-based systems need a dedicated entry point for APIs, and how an API Gateway simplifies security, routing, authentication, rate limiting, and other API-specific concerns.

---

# The Problem

Let's go back to our Instagram example.

Initially, Instagram was a relatively simple application.

```text
                Users
                   │
                   ▼
            +-------------+
            | Application |
            +-------------+
```

Life was easy.

There was only one backend.

---

As Instagram grew, engineers started splitting the application into multiple services.

```text
User Service

Post Service

Comment Service

Notification Service

Media Service

Search Service
```

Now the architecture looks like this:

```text
                    Users
                       │
                       ▼
                Reverse Proxy
                       │
                       ▼
      ┌─────────┬─────────┬─────────┐
      ▼         ▼         ▼
   User      Post      Search
  Service    Service    Service
```

This is much easier to scale.

Each service can evolve independently.

---

Now imagine opening Instagram.

The home screen needs:

- User profile
- Feed
- Stories
- Notifications
- Friend suggestions
- Advertisements

Where does this data come from?

Different services.

```text
User Service

Feed Service

Story Service

Notification Service

Ads Service
```

If the mobile app communicates directly with every service:

```text
Mobile App

 ├── User Service
 ├── Feed Service
 ├── Story Service
 ├── Notification Service
 ├── Ads Service
```

The client suddenly has to know:

- Every service address
- Every API endpoint
- Authentication for each service
- Which service owns which data

This creates a very tightly coupled client.

---

Now imagine we rename:

```text
Feed Service
```

to

```text
Timeline Service
```

Every mobile application must now be updated.

Old app versions may even stop working.

---

Another problem.

Suppose every service independently performs authentication.

```text
User Service

↓

Validate JWT

Feed Service

↓

Validate JWT

Comment Service

↓

Validate JWT

Search Service

↓

Validate JWT
```

The same logic is duplicated everywhere.

---

One more problem.

Imagine a user refreshes their feed rapidly.

```text
100 requests

in

10 seconds
```

Every service now has to implement:

- Rate limiting
- Request validation
- Logging
- Monitoring

Again...

the same logic appears in every service.

There has to be a better approach.

---

# Why Existing Solutions Fail

## DNS

DNS only tells users:

> "Here's the IP address."

It doesn't understand APIs.

---

## Load Balancer

A load balancer decides:

> "Which backend server receives this request?"

It doesn't know:

- Authentication
- API versions
- User permissions
- Rate limits
- Request transformation

---

## Reverse Proxy

Reverse proxies understand HTTP and perform:

- TLS termination
- Compression
- Caching
- URL routing

Very useful.

But as API ecosystems grow, we need much richer API-specific capabilities.

---

# The Big Idea

An **API Gateway** is a single entry point for all client API requests.

Instead of clients talking directly to many services, they talk to the API Gateway.

The gateway communicates with the appropriate backend services on their behalf.

---

# Before API Gateway

```text
              Mobile App

      ├── User Service
      ├── Feed Service
      ├── Search Service
      ├── Comment Service
      ├── Notification Service
```

Problems:

- Client knows every service.
- Authentication repeated everywhere.
- Rate limiting repeated everywhere.
- Difficult to evolve services.

---

# After API Gateway

```text
                  Mobile App
                       │
                       ▼
              +----------------+
              | API Gateway    |
              +----------------+
                │   │   │   │
                ▼   ▼   ▼   ▼
             User Feed Search Comment
```

The client knows only one endpoint.

Everything else becomes an internal implementation detail.

---

# Why is it called a Gateway?

Think about entering an airport.

You don't directly enter:

- Security
- Immigration
- Boarding

Instead:

You first pass through the main entrance.

The entrance decides:

- Who can enter.
- Where they should go.
- Which checks are required.

An API Gateway plays the same role.

---

# Responsibilities of an API Gateway

An API Gateway provides many API-focused capabilities.

---

# 1. Single Entry Point

Instead of:

```text
api-users.company.com

api-orders.company.com

api-payments.company.com

api-search.company.com
```

Clients simply call:

```text
api.company.com
```

Much simpler.

---

# 2. Request Routing

The gateway decides where requests belong.

Example:

```text
GET /users

↓

User Service
```

```text
GET /orders

↓

Order Service
```

```text
GET /payments

↓

Payment Service
```

The client doesn't need to know where these services actually live.

---

# 3. Authentication

Suppose a request arrives.

```text
Authorization:

Bearer JWT
```

Instead of every service validating the JWT:

```text
Gateway

↓

Validate Token

↓

Forward Request
```

Backend services can trust that authenticated requests have already passed through the gateway.

This removes duplicated work.

> **Note:** In practice, many services still perform lightweight authorization checks (such as verifying user roles or scopes) because they own their own business rules. The gateway centralizes authentication, but services remain responsible for protecting sensitive operations.

---

# 4. Authorization

Authentication answers:

> Who are you?

Authorization answers:

> What are you allowed to do?

Suppose:

```text
DELETE /users/123
```

Only administrators should perform this action.

The gateway can reject unauthorized requests before they reach backend services.

---

# 5. Rate Limiting

Imagine one client sends:

```text
10,000 requests

per minute
```

Without protection:

Backend services become overloaded.

Instead:

```text
Gateway

↓

100 requests/minute

Allowed

↓

Reject everything else
```

The gateway protects the entire system.

---

# 6. API Versioning

Suppose we release:

```text
API V2
```

Old mobile apps still use:

```text
API V1
```

The gateway can route:

```text
/api/v1

↓

Old Service
```

```text
/api/v2

↓

New Service
```

Clients upgrade gradually.

---

# 7. Request Aggregation

One screen often requires data from multiple services.

Suppose the mobile app needs:

- User profile
- Recent posts
- Notifications

Without aggregation:

```text
Mobile App

↓

User Service

↓

Feed Service

↓

Notification Service
```

Three network requests.

Instead:

```text
Mobile App

↓

Gateway

↓

User Service

Feed Service

Notification Service

↓

One Combined Response
```

The gateway performs multiple backend calls and returns one response.

This reduces network overhead for clients, especially mobile applications.

---

# 8. Request Transformation

Sometimes internal APIs differ from public APIs.

Example:

External request:

```json
{
  "userId": 15
}
```

Internal service expects:

```json
{
  "id": 15
}
```

The gateway can transform requests and responses without requiring every client to change.

---

# 9. Logging and Monitoring

Every request passes through the gateway.

This makes it an excellent place to collect:

- Request counts
- Latency
- Error rates
- User activity
- API usage statistics

Instead of collecting logs separately from every service.

---

# 10. API Keys and Client Management

Public APIs often need to distinguish between different clients.

For example:

```text
Weather API

↓

Client A

Client B

Client C
```

Each receives:

- Different API keys
- Different rate limits
- Different usage quotas

The gateway manages these policies centrally.

---

# API Gateway vs Reverse Proxy

Another common interview question.

They are closely related, but not identical.

---

## Reverse Proxy

General HTTP infrastructure.

Focuses on:

- TLS termination
- Caching
- Compression
- URL routing
- Security

It can serve websites, images, APIs, and more.

---

## API Gateway

Specialized for APIs.

Focuses on:

- Authentication
- Authorization
- API versioning
- Rate limiting
- Request aggregation
- API keys
- Request transformation
- Monitoring

Think of it as a reverse proxy with API-specific intelligence.

---

# Real-World Usage

## Amazon

Uses API gateways for many internal microservices and public APIs.

---

## Uber

Mobile applications communicate through API gateways rather than directly with hundreds of backend services.

---

## Netflix

Uses gateway layers that handle authentication, routing, monitoring, and request aggregation before forwarding requests to backend services.

---

## Stripe

Its public payment APIs are exposed through gateway infrastructure that enforces authentication, rate limits, and request validation.

---

# Where It Helps

API Gateways are valuable because they:

- Hide backend services.
- Simplify client development.
- Centralize authentication.
- Enforce rate limits.
- Support API versioning.
- Aggregate multiple backend calls.
- Improve observability.
- Decouple clients from backend architecture.

---

# Where It Doesn't Help

An API Gateway cannot:

- Replace backend business logic.
- Replace databases.
- Replace asynchronous messaging.
- Eliminate the need for service-to-service security.
- Automatically improve poorly designed APIs.

It coordinates access to services; it doesn't implement the services themselves.

---

# Mental Model

Imagine a luxury hotel.

Guests don't wander through the building asking every department for help.

Instead, they visit the concierge.

```text
Guest

↓

Concierge

↓

Restaurant

Housekeeping

Spa

Room Service

Taxi
```

The concierge:

- Verifies reservations.
- Directs requests.
- Coordinates multiple departments.
- Provides one consistent experience.

Guests interact with one person.

The concierge coordinates the rest.

The concierge is the API Gateway.

---

# Tradeoffs

## Advantages

- Provides a single, stable API endpoint.
- Centralizes authentication and rate limiting.
- Reduces duplicated infrastructure code.
- Shields clients from backend changes.
- Supports request aggregation and transformation.
- Simplifies monitoring and analytics.

## Disadvantages

- Adds another infrastructure component to manage.
- Can become a bottleneck if not scaled properly.
- Overloading the gateway with too much business logic can make it difficult to maintain.
- Gateway failures can impact many APIs unless deployed with high availability.

---

# Common Interview Questions

### Why not let clients call microservices directly?

It tightly couples clients to backend architecture, duplicates authentication logic, and makes backend changes difficult without updating clients.

---

### What is request aggregation?

The gateway calls multiple backend services and combines their responses into a single response for the client.

---

### Why perform rate limiting at the gateway?

Because every request passes through it, allowing the entire system to be protected with one centralized policy.

---

### What is the difference between a Reverse Proxy and an API Gateway?

A reverse proxy is a general-purpose HTTP infrastructure component. An API Gateway builds on similar ideas but adds API-specific capabilities like authentication, authorization, versioning, API keys, request aggregation, and rate limiting.

---

### Should backend services trust the gateway completely?

The gateway usually handles client authentication and common policies, but backend services should still enforce their own authorization and business rules. This follows the principle of defense in depth.

---

# Before vs After Architecture

## Before

```text
                 Mobile App

      ├── User Service
      ├── Feed Service
      ├── Search Service
      ├── Payment Service
      └── Notification Service
```

Problems:

- Clients know internal services.
- Authentication is duplicated.
- Backend changes affect clients directly.

---

## After

```text
                 Mobile App
                      │
                      ▼
             +----------------+
             |  API Gateway   |
             +----------------+
                │   │   │   │
                ▼   ▼   ▼   ▼
             User Feed Search Payment
              Svc   Svc   Svc    Svc
```

Benefits:

- One stable API endpoint.
- Centralized API management.
- Clients are decoupled from internal architecture.
- Services can evolve independently.

---

# Connections

So far, we've built a robust request path:

- **DNS** tells clients where to connect.
- **Load Balancer** distributes requests across healthy servers.
- **Reverse Proxy** handles infrastructure concerns like TLS, caching, and routing.
- **API Gateway** manages API-specific concerns for clients.

But we've quietly made an assumption throughout these chapters:

The gateway, reverse proxy, and load balancer all somehow know where the backend services are.

For example:

```text
Gateway

↓

User Service
```

But what if:

- A new User Service instance starts?
- One instance crashes?
- Kubernetes launches 20 new instances?
- Auto Scaling removes 15 old instances?

How does the gateway know the current addresses of these constantly changing services?

That leads us to the final chapter of this module:

> **Chapter 5 — Service Discovery**
