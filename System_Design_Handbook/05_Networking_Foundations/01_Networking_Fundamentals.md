# Module 5 — Networking Foundations

# Chapter 1 — Networking Fundamentals

## Goal

Understand what a computer network is, why it exists, and the core networking concepts every system designer needs before learning about HTTP, TCP, TLS, and other protocols.

---

# 1. The Problem

Imagine you're designing Instagram.

A user opens the app and taps on a photo.

Within a fraction of a second, the app receives:

- User profile
- Caption
- Comments
- Likes
- Images
- Suggested posts
- Advertisements

But here's the question:

**How does your phone actually get this data?**

Your phone isn't inside Meta's data center.

The servers may be thousands of kilometers away.

Somewhere between your phone and Instagram's servers, the request travels through:

- Wi-Fi
- Your home router
- Your ISP
- Multiple internet routers
- Fiber optic cables
- Internet exchanges
- Data center networks
- Load balancers
- Application servers

None of this happens magically.

It happens because billions of devices are connected together into one enormous network.

Before we can understand HTTP requests or APIs, we first need to understand **what a network actually is.**

---

# 2. Why Existing Solutions Fail

Until now, every module quietly assumed something.

Whenever we drew diagrams like this:

```text
User
   │
Load Balancer
   │
App Servers
   │
Database
```

We simply drew lines.

We never asked:

- What are those lines?
- How does data travel across them?
- Why doesn't data get mixed up with everyone else's?
- How does a server on another continent receive my request?

Our previous modules focused on **what systems do.**

Now we need to understand **how systems talk to each other.**

Without networking, there is no distributed system.

There is no caching.

There is no API Gateway.

There is no cloud.

Everything we've learned depends on computers being able to communicate.

---

# 3. The Big Idea

> **A network is simply a way for computers to exchange data with each other.**

Everything else in networking exists to make that communication reliable, efficient, and scalable.

---

# 4. Detailed Explanation

## Imagine Only Two Computers

Let's start as simply as possible.

Suppose you own two laptops.

```text
Laptop A

Laptop B
```

You connect them using a cable.

Now they can exchange files.

Congratulations.

You've built the smallest possible network.

```text
Laptop A ───────── Laptop B
```

Networking begins with just two devices exchanging information.

---

## Adding More Computers

Now suppose you add three more laptops.

```text
A
B
C
D
E
```

Should every computer connect directly to every other computer?

```text
A────B
│\  /│
│ \/ │
│ /\ │
│/  \│
C────D
 \  /
  E
```

This quickly becomes a mess.

The number of cables grows rapidly.

For **N** computers:

```text
Connections = N(N-1)/2
```

For just:

- 10 computers → 45 cables
- 100 computers → 4,950 cables
- 1,000 computers → almost half a million cables

Clearly, this doesn't scale.

---

## Introducing a Switch

Instead, every computer connects to one central device.

```text
      Switch
   / / | \ \
  A  B C  D E
```

Now:

- Each computer needs only one cable.
- The switch forwards data to the correct destination.

Instead of every device talking directly, they communicate through shared infrastructure.

This is the first example of a recurring system design principle:

> Central coordination often simplifies communication.

---

## Expanding Beyond One Building

Now imagine your company opens another office.

```text
Office A

Office B
```

Each office has its own network.

How do they communicate?

You connect the two networks together.

```text
Office A
   │
Router
   │
Internet
   │
Router
   │
Office B
```

Instead of connecting individual computers, we connect **entire networks**.

That's how the Internet works.

---

# What Is the Internet?

The Internet is **not one giant computer.**

It is not owned by one company.

It isn't even one network.

Instead, it is:

> **A network made up of millions of smaller networks connected together.**

Hence the name:

**Inter-network**

↓

**Internet**

Every ISP, cloud provider, university, company, and government runs its own network.

Together they form one global network.

```text
Office Network
        │
ISP
        │
Internet
        │
Cloud Provider
        │
Data Center
```

When your phone accesses Netflix, the request travels across many independent networks before reaching Netflix's servers.

---

# Data Doesn't Travel as One Giant File

Suppose you're downloading a 500 MB video.

Does the entire file travel as one enormous block?

Imagine one truck carrying every package.

If the truck crashes halfway, everything is lost.

Instead, networking breaks data into many small pieces.

These pieces are called **packets**.

```text
Video

↓

Packet 1
Packet 2
Packet 3
Packet 4
Packet 5
...
```

Each packet travels independently.

Some may even take different routes.

The destination reassembles them into the original file.

This improves:

- Reliability
- Efficiency
- Error recovery
- Network utilization

We'll study packets in much greater detail in later chapters.

For now, think of them as the basic unit of data sent across a network.

---

# Networks Can Be Different Sizes

Not every network spans the entire globe.

Different networks exist for different purposes.

---

## Local Area Network (LAN)

A LAN covers a relatively small geographic area.

Examples:

- Home Wi-Fi
- Office network
- School campus
- Coffee shop Wi-Fi

```text
Laptop
   │
Wi-Fi Router
   │
Desktop
   │
Printer
```

Characteristics:

- Short distance
- High speed
- Privately managed
- Low latency

---

## Wide Area Network (WAN)

A WAN connects networks over large distances.

Examples:

- Connecting offices across countries
- The Internet
- Corporate global networks

```text
Delhi Office
      │
Internet
      │
London Office
      │
New York Office
```

Characteristics:

- Long distance
- Multiple network providers
- Higher latency
- More complex routing

---

### Why Does This Matter in System Design?

Suppose your application servers are all inside one AWS region.

Communication happens over a fast internal network.

```text
App Server
      │
Database
```

Latency may be less than a millisecond.

Now suppose your database is in another continent.

```text
India
   │
Internet
   │
US Database
```

Even if both machines are incredibly powerful, the physical distance introduces delay.

Sometimes, network latency dominates computation time.

This is why distributed systems often place services close to each other.

---

# Public vs Private IP (High Level)

Imagine an apartment building.

Inside the building:

- Flat 101
- Flat 102
- Flat 103

These numbers only make sense inside that building.

Someone outside can't deliver a package using only "Flat 102."

They first need the building's street address.

Networking works similarly.

Devices inside your home have **private addresses**.

Your home itself has one **public address** visible to the Internet.

```text
Internet
      │
Public Address
      │
Router
   ┌──┴───┐
Phone Laptop TV
```

We'll study IP addresses properly in Chapter 4.

For now, simply remember:

- **Private IP:** Used within a local network.
- **Public IP:** Visible to the rest of the Internet.

---

# What Is NAT? (High Level)

Now another question appears.

If your phone, laptop, TV, and tablet all share one public address...

How does the Internet know which device should receive the response?

This is where **Network Address Translation (NAT)** comes in.

Think of NAT as a receptionist.

Everyone in the office shares the company's mailing address.

When mail arrives, the receptionist knows which employee it belongs to.

```text
Internet
      │
Router (NAT)
   ┌──┴────┐
Phone Laptop TV
```

NAT keeps track of outgoing requests and ensures the responses are delivered to the correct device.

We'll revisit NAT in greater depth later.

---

# Routers vs Switches (High Level)

These two devices are often confused.

They solve different problems.

## Switch

A switch connects devices **within the same network**.

Example:

```text
PC
 │
Switch
 │
Printer
 │
Server
```

Think of a switch as managing communication inside one office.

---

## Router

A router connects **different networks**.

Example:

```text
Home Network
      │
Router
      │
Internet
      │
Cloud Network
```

Think of a router as deciding how to travel between cities.

We'll avoid implementation details for now because routing algorithms deserve their own discussion.

---

# Why Does Networking Use Layers?

Imagine building a postal system.

One team designs roads.

Another designs trucks.

Another designs envelopes.

Another writes addresses.

Each team focuses on one responsibility.

Networking works the same way.

Instead of building one enormous protocol that handles everything, networking divides responsibilities into layers.

Each layer solves one problem.

```text
Application
Transport
Internet
Physical Network
```

This separation makes networking:

- Easier to improve
- Easier to debug
- Easier to replace individual technologies
- Easier to standardize

The next chapter is entirely about these layers.

---

# 5. Types / Variations

## Network Sizes

| Type | Covers                | Example                             |
| ---- | --------------------- | ----------------------------------- |
| LAN  | Small area            | Home, office, school                |
| WAN  | Large geographic area | Internet, global enterprise network |

---

## Network Devices

| Device | Primary Responsibility              |
| ------ | ----------------------------------- |
| Switch | Connect devices within one network  |
| Router | Connect different networks together |

---

## Address Types

| Type       | Scope                   |
| ---------- | ----------------------- |
| Private IP | Inside local network    |
| Public IP  | Visible on the Internet |

---

# 6. Real-World Usage

### Google

Google operates one of the world's largest private global networks. User requests often travel through Google's own backbone network instead of relying solely on the public Internet, reducing latency and improving reliability.

---

### Netflix

When you stream a movie, packets travel from Netflix's servers through multiple ISPs before reaching your home network. Netflix also places content closer to users using CDNs to reduce long-distance network travel.

---

### Amazon

Amazon's AWS regions contain massive internal networks connecting compute instances, databases, storage, and load balancers. Communication inside a region is much faster than communication across continents.

---

### Uber

When you request a ride, your phone communicates over mobile networks, your ISP, the Internet, and Uber's backend infrastructure before a driver is matched—all within seconds.

---

# 7. Where It Helps

Understanding networking fundamentals helps you:

- Reason about latency between services.
- Understand why geographically distributed systems behave differently.
- Appreciate why data centers and cloud regions matter.
- Build scalable distributed architectures.
- Understand later topics like HTTP, TCP, TLS, WebSockets, and gRPC.

---

# 8. Where It Doesn't Help

Networking fundamentals explain **how devices communicate**, but they do not explain:

- How communication stays reliable (TCP).
- How applications structure requests (HTTP).
- How data is encrypted (TLS).
- How servers discover each other (Service Discovery).
- How asynchronous messaging works.

Those require additional layers and protocols, which we'll build on in later chapters.

---

# 9. Mental Model

Imagine an international postal system.

- Houses are computers.
- Neighborhoods are LANs.
- Cities are separate networks.
- Highways connect cities (WAN).
- Routers choose which highways to take.
- Switches deliver mail within a neighborhood.
- Packets are individual envelopes.
- The Internet is the global postal network connecting every city.

No single organization owns the entire postal system, but they all cooperate using shared rules. The Internet works in much the same way.

---

# 10. Tradeoffs

## Advantages

- Enables communication across any distance.
- Supports billions of connected devices.
- Scales by connecting smaller networks together.
- Faults in one local network do not necessarily affect the entire Internet.
- Layering allows technologies to evolve independently.

## Disadvantages

- Longer physical distances increase latency.
- More network hops introduce additional delay and potential failures.
- Communication depends on many independent networks cooperating.
- Network failures can impact otherwise healthy applications.
- Complexity grows as networks become larger and more distributed.

---

# 11. Common Interview Questions

### Why do distributed systems rely so heavily on networking?

Because every service, database, cache, or message broker communicates over a network. The network is effectively the "glue" that connects all distributed components.

---

### Why don't computers connect directly to every other computer?

A fully connected network scales poorly because the number of required connections grows rapidly. Central devices like switches and routers make networks practical and manageable.

---

### Why is data broken into packets?

Small packets improve reliability and efficiency. If a packet is lost or corrupted, only that packet needs to be retransmitted instead of the entire file.

---

### What's the difference between a LAN and a WAN?

A LAN connects devices within a small area under a single administrative domain, while a WAN connects multiple networks across large geographic distances, often involving many different providers.

---

### What's the difference between a switch and a router?

A switch forwards traffic within the same network, while a router forwards traffic between different networks.

---

### Why is network latency such an important consideration in system design?

Even very fast servers cannot overcome the time required for data to travel across long distances. As systems become distributed, communication delays often become a dominant factor in overall response time.

---

# 12. Before vs After Architecture

### Before Networking

```text
Computer A

Computer B

(No communication)
```

↓

### Direct Connection

```text
Computer A ───────── Computer B
```

↓

### Small Local Network

```text
      Switch
    /   |   \
 PC1  PC2  PC3
```

↓

### Network of Networks

```text
Home LAN
     │
Router
     │
Internet
     │
Cloud Router
     │
Application Servers
```

---

# 13. Connections

So far, we've learned that computers can communicate by exchanging packets across interconnected networks.

But another question immediately appears:

> **How do billions of devices, cables, routers, and applications all cooperate without descending into chaos?**

The answer is **layering**.

Rather than making every device understand every networking detail, responsibilities are divided into well-defined layers. Each layer solves one specific problem and builds on the layer beneath it.

In the next chapter, we'll explore **the OSI Model and the TCP/IP Model**, which provide the conceptual blueprint for almost every modern network communication system.
