# Module 4 — Traffic Management & Request Routing

# Chapter 1 — DNS (Domain Name System)

---

# Goal

Understand how a user's request finds the correct server somewhere on the Internet.

---

# The Problem

Imagine you want to watch a movie on Netflix.

You open your browser and type:

```
https://www.netflix.com
```

A few seconds later, the homepage appears.

Seems simple.

But think about what just happened.

How did your computer know where Netflix's servers are?

Netflix owns thousands of servers spread across hundreds of data centers around the world.

```
USA
Europe
India
Japan
Australia
Brazil
...
```

Which one should your laptop connect to?

Even more importantly—

How did your laptop even know the location of any Netflix server?

---

Suppose instead of typing:

```
www.netflix.com
```

you had to type

```
44.236.72.91
```

every time.

Now imagine remembering IP addresses for:

- Google
- Amazon
- Instagram
- YouTube
- GitHub
- Reddit
- StackOverflow

Impossible.

Humans are good at remembering names.

Computers communicate using IP addresses.

We need something that translates between the two worlds.

---

# Why Existing Solutions Fail

So far we've learned:

- Databases
- Replication
- Sharding
- Caching

These solve problems **inside backend systems**.

But before any backend receives a request...

the user's device must first locate that backend.

Without knowing where the server lives, nothing else matters.

A browser cannot send a request to:

```
amazon.com
```

because network routers do not understand domain names.

Routers only understand IP addresses.

We need a giant directory that answers questions like:

```
Where is google.com?

Where is youtube.com?

Where is amazon.com?
```

---

# The Big Idea

**DNS is the Internet's phonebook.**

It converts human-friendly domain names into machine-friendly IP addresses.

```
google.com
        ↓
142.250.183.206
```

Once the IP address is known, the browser can contact the server.

---

# Understanding IP Addresses

Every machine connected to the Internet has an address.

For example:

```
142.250.183.206
```

Think of it like a home address.

```
House Name:
Google

Address:
142.250.183.206
```

When you send a package, you need the destination address.

Similarly...

When your browser sends a request...

it needs the server's IP address.

Without it, the request has nowhere to go.

---

# Why Can't We Just Memorize IP Addresses?

There are several problems.

## Problem 1 — Humans hate numbers

Which is easier?

```
google.com
```

or

```
142.250.183.206
```

Names win.

---

## Problem 2 — Servers change

Suppose Netflix moves their application to a new data center.

Yesterday:

```
44.10.2.5
```

Today:

```
93.44.10.18
```

If everyone remembered IPs...

every user would have to learn the new one.

Instead...

Netflix only updates DNS.

Users keep typing:

```
netflix.com
```

No changes required.

---

## Problem 3 — Multiple Servers

Google doesn't have one server.

It has millions.

Depending on where you live...

```
google.com
```

may resolve to different IP addresses.

India

↓

```
142.250.x.x
```

USA

↓

```
172.217.x.x
```

Japan

↓

```
216.58.x.x
```

DNS makes this possible.

---

# What Happens When You Enter a Website?

Let's walk through the entire process.

You type:

```
www.amazon.com
```

into your browser.

---

## Step 1

Browser asks:

```
Do I already know the IP?
```

If yes...

No DNS lookup needed.

This is called **DNS caching**.

```
Browser Cache

amazon.com

↓

54.239.28.85
```

If found...

Done.

---

Otherwise...

---

## Step 2

Ask the Operating System.

```
Browser

↓

Operating System Cache
```

Maybe another application recently visited Amazon.

If yes...

Reuse that answer.

---

Otherwise...

---

## Step 3

Ask the Local DNS Resolver.

Usually this belongs to:

- ISP
- Google DNS
- Cloudflare DNS
- Enterprise DNS server

Example:

```
Browser

↓

DNS Resolver
```

The resolver now tries to find the answer.

---

# Where Does the Resolver Look?

The resolver itself also maintains a huge cache.

Maybe someone nearby recently searched:

```
amazon.com
```

If yes...

Return immediately.

No Internet-wide search required.

---

If not...

the resolver begins asking the DNS hierarchy.

---

# DNS is Hierarchical

Instead of one gigantic server containing every website...

DNS is split into levels.

```
                Root
                  |
         ----------------
         |              |
       .com           .org
         |
     amazon.com
         |
     www.amazon.com
```

This hierarchy keeps DNS scalable for billions of domains.

Let's see each level.

---

## Root Name Servers

At the very top are the **Root Name Servers**.

Think of them as reception desks.

You ask:

```
Where is amazon.com?
```

They answer:

```
I don't know.

But I know who manages .com.
```

Notice...

They don't know Amazon's IP.

They only know where to go next.

---

## TLD Servers

TLD means:

**Top-Level Domain**

Examples:

```
.com

.org

.net

.edu

.io
```

The resolver now asks the `.com` servers:

```
Where is amazon.com?
```

The TLD server replies:

```
Ask Amazon's authoritative DNS server.
```

Again...

Still no IP.

Just another direction.

---

## Authoritative Name Server

This server belongs to Amazon (or whoever manages its DNS).

Now the resolver asks:

```
What is the IP of www.amazon.com?
```

Finally...

The authoritative server replies:

```
54.239.28.85
```

Success.

The resolver caches the result and returns it to your browser.

---

# Complete Lookup Flow

```
User

↓

Browser Cache

↓

OS Cache

↓

DNS Resolver

↓

Root Server

↓

.com Server

↓

Amazon Authoritative DNS

↓

IP Address Returned

↓

Browser connects to server
```

---

# Why So Many Levels?

Imagine a single server storing every domain on Earth.

```
google.com

youtube.com

amazon.com

...

500+ million domains
```

Problems:

- Too much data
- Single point of failure
- Impossible to scale
- Huge latency

Instead...

DNS distributes responsibility.

```
Root

knows TLDs

↓

TLD

knows domains

↓

Domain

knows subdomains
```

Each layer only manages a small part of the namespace.

---

# DNS Caching

Earlier we studied caching extensively.

DNS is another great example.

Without caching...

Every website visit would require:

```
Browser

↓

Resolver

↓

Root

↓

TLD

↓

Authoritative Server
```

for every single request.

That would make browsing painfully slow.

Instead...

everyone caches DNS answers.

```
Browser

↓

OS

↓

Resolver

↓

ISP
```

Often, the entire lookup finishes locally.

---

# Time To Live (TTL)

Earlier we learned about TTL in caching.

DNS uses exactly the same concept.

Suppose Amazon says:

```
Cache this answer for

300 seconds.
```

That means:

```
amazon.com

↓

54.239.28.85

↓

Expires after 5 minutes
```

After TTL expires...

the resolver performs a fresh lookup.

---

## Why Not Cache Forever?

Imagine Amazon migrates to a new infrastructure.

Old IP:

```
54.239.28.85
```

New IP:

```
18.220.15.9
```

If everyone cached forever...

users would continue contacting the old server.

TTL ensures outdated mappings eventually disappear.

---

# Recursive vs Iterative Lookup

This is one area that often confuses beginners.

Let's simplify it.

---

## Recursive Lookup

You ask one person:

```
Find Amazon's IP for me.
```

That person does all the work.

```
You

↓

Resolver

↓

Root

↓

TLD

↓

Authoritative

↓

Answer comes back
```

You only wait for the final answer.

This is how your browser interacts with a DNS resolver.

---

## Iterative Lookup

Instead of doing everything...

each server simply points you to the next one.

Example:

```
Root

↓

Ask .com

↓

.com

↓

Ask Amazon

↓

Amazon

↓

Here is the IP
```

The resolver performs these iterative steps behind the scenes.

---

# Real-World Usage

Almost every Internet service relies on DNS.

### Google

Maps users to nearby Google servers.

### Netflix

Routes users toward regional infrastructure.

### Amazon

Uses DNS to direct traffic to healthy endpoints.

### GitHub

Maps requests to global infrastructure.

### Cloudflare

Provides globally distributed authoritative DNS services.

---

# Where It Helps

DNS is useful because it:

- Hides complex IP addresses behind readable names.
- Allows server IPs to change without affecting users.
- Supports large-scale distributed infrastructure.
- Enables caching for faster lookups.
- Forms the entry point for nearly every web request.

---

# Where It Doesn't Help

DNS is **not** responsible for:

- Choosing which application server handles your request.
- Distributing traffic between backend servers.
- Authenticating users.
- Encrypting communication.
- Speeding up application processing.

Once DNS returns an IP address, its job is done.

The next stages (load balancers, reverse proxies, API gateways, etc.) take over.

---

# Mental Model

Imagine you want to visit a friend in another city.

You know their name:

```
John
```

But not their address.

You first check:

```
Your contacts
```

If not found...

you call a directory service.

The directory tells you:

```
John lives at:

221 Baker Street
```

Now you can drive there.

DNS works exactly the same way.

```
Domain Name

↓

Address Lookup

↓

IP Address

↓

Visit Server
```

---

# Tradeoffs

## Advantages

- Human-friendly names instead of IP addresses.
- Decentralized and highly scalable hierarchy.
- Heavy caching reduces lookup latency.
- Allows infrastructure changes without changing public URLs.
- Enables global services to present a single domain while serving users from many locations.

## Disadvantages

- DNS lookup adds latency (though caching usually minimizes it).
- Cached records can become temporarily stale until TTL expires.
- DNS outages can make services unreachable even if application servers are healthy.
- Misconfigured DNS records can direct users to the wrong destination.

---

# Common Interview Questions

### Why do we need DNS?

Because humans use names while networks route using IP addresses. DNS translates between them.

---

### Why is DNS hierarchical?

A single global database for every domain would be difficult to scale and would create a single point of failure. The hierarchy distributes responsibility.

---

### Why does DNS use caching?

To reduce latency and avoid repeatedly querying root, TLD, and authoritative servers.

---

### What is TTL in DNS?

TTL (Time To Live) specifies how long a DNS record may be cached before it should be refreshed.

---

### Does DNS choose the backend server that handles my request?

No. DNS only resolves a domain name to an IP address. Traffic distribution among backend servers is typically handled by load balancers, which we'll study next.

---

# Before vs After Architecture

### Before DNS

```
User

↓

Remember IP Address

↓

Connect to Server
```

Problems:

- Hard to remember
- IP changes break clients
- Poor scalability

---

### After DNS

```
User

↓

www.amazon.com

↓

DNS

↓

IP Address

↓

Application Server
```

Users always remember the same domain, while the underlying infrastructure can evolve independently.

---

# Connections

Now that the browser knows **where** to send the request, another question immediately appears:

Suppose DNS returns the IP address of Netflix.

Behind that IP might be:

- 10 servers
- 100 servers
- 10,000 servers

How does the request get routed to **one specific healthy server**?

How do we prevent a single machine from becoming overloaded while others sit idle?

That is the problem solved by the next chapter:

> **Chapter 2 — Load Balancers**

---

# Key Takeaways

- DNS translates domain names into IP addresses.
- Computers communicate using IP addresses, while humans prefer memorable names.
- DNS is hierarchical: Root → TLD → Authoritative Name Server.
- DNS responses are heavily cached by browsers, operating systems, and resolvers to reduce latency.
- TTL controls how long a DNS record can be cached before it must be refreshed.
- DNS's responsibility ends after providing the destination IP address.
- Once the destination is known, the next challenge is deciding **which server behind that IP** should receive the request—this leads naturally to load balancing.
