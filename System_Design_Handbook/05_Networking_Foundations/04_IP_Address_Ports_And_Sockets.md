# Chapter 4 — IP Addresses, Ports & Sockets

---

# Goal

Understand how data reaches not just the correct **machine**, but also the correct **application** running on that machine.

---

# 1. The Problem

Imagine you're using your laptop.

At this exact moment, you might have:

- Chrome open with YouTube
- Slack running
- Spotify playing music
- VS Code connected to GitHub
- Docker containers running locally
- A PostgreSQL database
- A Redis server

All of these applications are communicating over the network simultaneously.

Now imagine a packet arrives at your laptop.

How does your operating system know whether that packet is for:

- Chrome?
- Spotify?
- PostgreSQL?
- Redis?
- Docker?

The packet reached the correct computer.

But that's only half the problem.

It still has to reach the correct application.

---

# 2. Why Existing Solutions Fail

In the previous chapter, we learned that:

- IP helps packets reach the correct machine.
- TCP and UDP transport data between machines.

But an IP address only identifies a **device**.

Suppose your laptop has this IP:

```text
192.168.1.20
```

That tells the Internet:

> "Deliver this packet to this laptop."

But your laptop is running dozens of programs.

IP cannot distinguish between them.

Without another level of addressing:

```text
Packet

↓

Laptop

↓

???
```

The operating system has no idea where to deliver the data.

We need a way to identify **applications**, not just computers.

---

# 3. The Big Idea

> **IP addresses identify machines. Ports identify applications. Sockets identify a specific communication between two applications.**

Think of it as adding another level of addressing.

---

# 4. Detailed Explanation

## Part 1 — IP Addresses

Let's begin with something familiar.

Suppose you want to send a letter.

You write:

```text
221B Baker Street
London
```

That address identifies a house.

Networking works similarly.

Every device participating in a network has an address.

That address is called an **IP Address**.

Example:

```text
203.0.113.25
```

Its purpose is simple:

> "Deliver this packet to this machine."

Nothing more.

It does **not** specify which application should receive the packet.

---

# Why Do We Need IP Addresses?

Imagine the Internet without addresses.

```text
Packet

↓

Internet

↓

Which computer?
```

Routers would have no idea where to send data.

IP addresses provide a unique destination that routers can use to forward packets.

Think of an IP address as the street address of a building.

---

# Public vs Private IP (Revisited)

We briefly introduced this in the previous chapter.

Now let's understand it a little better.

Imagine an apartment building.

The building has one postal address:

```text
221B Baker Street
```

Inside the building are apartments:

```text
Flat 101
Flat 102
Flat 103
```

Someone outside only needs the building's address first.

The building itself handles delivery to the correct apartment.

Networking follows the same idea.

```text
Internet
      │
Public IP
      │
Home Router
      │
Private Network
```

Inside your home:

```text
Phone

192.168.1.2

Laptop

192.168.1.5

TV

192.168.1.9
```

Outside:

Your ISP assigns one public IP.

```text
49.x.x.x
```

The rest of the Internet only sees this public IP.

---

# Why Private IPs Exist

Imagine every light bulb, TV, and smart speaker in the world required its own globally unique address.

We would quickly exhaust the available address space (particularly with IPv4).

Instead:

Homes, offices, and schools reuse private IP ranges internally.

The router connects that private network to the public Internet.

This is both efficient and easier to manage.

---

# NAT Revisited

We introduced NAT (Network Address Translation) earlier.

Let's now see it in action.

Suppose your laptop sends a request.

```text
Laptop

192.168.1.5

↓

Router

↓

Internet
```

The router changes the source address.

```text
Before

192.168.1.5

↓

After

49.35.120.10
```

The outside world only sees the router's public IP.

When the response comes back, the router remembers which internal device initiated the request and forwards it appropriately.

You can think of NAT as a receptionist keeping track of outgoing conversations and directing replies to the correct employee.

---

# Part 2 — Ports

Now imagine your packet has reached the correct laptop.

We're still not finished.

Suppose this laptop is running:

```text
Chrome

Spotify

Slack

Redis

PostgreSQL
```

Who should receive the packet?

This is where **ports** come in.

---

# What Is a Port?

A port is simply a logical identifier for a network service or application.

Think of it as an apartment number inside a building.

The IP address gets you to the building.

The port gets you to the correct apartment.

```text
Street Address

↓

Apartment Number
```

Networking works exactly the same way.

```text
IP Address

↓

Port
```

---

# Example

Suppose your server has this IP:

```text
18.210.45.100
```

Multiple applications are running.

```text
18.210.45.100

├── Port 80
├── Port 443
├── Port 5432
├── Port 6379
```

Each application listens on a different port.

Incoming packets specify both:

```text
Destination IP

18.210.45.100

Destination Port

443
```

The operating system now knows exactly where to deliver the data.

---

# Common Port Numbers

Some ports have become standard conventions.

| Port  | Common Service |
| ----- | -------------- |
| 80    | HTTP           |
| 443   | HTTPS          |
| 22    | SSH            |
| 25    | SMTP (Email)   |
| 53    | DNS            |
| 3306  | MySQL          |
| 5432  | PostgreSQL     |
| 6379  | Redis          |
| 27017 | MongoDB        |

These aren't mandatory, but using the standard ports makes systems easier to understand and configure.

---

# Can Multiple Applications Use the Same Port?

No.

Not on the same IP address and transport protocol at the same time.

For example:

```text
Redis

Port 6379
```

Another Redis instance cannot also bind to:

```text
6379
```

unless it uses:

- A different IP address, or
- A different transport protocol (TCP vs UDP), or
- The first process releases the port.

The operating system must know exactly which application owns a given listening port.

---

# Part 3 — Sockets

Now we know:

- IP identifies machines.
- Ports identify applications.

But communication happens between **two** applications.

Consider your browser requesting a webpage.

```text
Browser

↓

Web Server
```

Both sides have:

- An IP address.
- A port.

Together, they uniquely identify the communication.

This is where sockets come in.

---

# What Is a Socket?

A socket represents one endpoint of a network communication.

You can think of it as:

> **An IP address + a port + a transport protocol.**

For example:

```text
192.168.1.10:54321 (TCP)
```

This identifies one communication endpoint on your machine.

---

# Client and Server Sockets

Suppose your browser visits:

```text
https://example.com
```

The browser chooses a temporary (ephemeral) port.

Example:

```text
Client

IP: 192.168.1.5

Port: 52134
```

The server listens on:

```text
Server

IP: 203.0.113.20

Port: 443
```

The communication can be represented as:

```text
192.168.1.5:52134

↓

203.0.113.20:443
```

This combination uniquely identifies that conversation.

Another browser tab might use:

```text
192.168.1.5:52135
```

Even though both tabs connect to the same server, the operating system can distinguish between them because they use different client-side ports.

---

# Why Does the Client Need a Port?

A common misconception is that only servers use ports.

Actually, clients need ports too.

Imagine Chrome opens three websites simultaneously.

```text
google.com

youtube.com

github.com
```

The operating system might assign:

```text
Chrome

52130

52131

52132
```

Each outgoing connection gets its own temporary port.

This allows responses to be matched to the correct browser tab or network connection.

---

# Listening Ports vs Ephemeral Ports

There are two broad categories of ports you'll encounter.

### Listening Ports

Servers wait for incoming connections on well-known or configured ports.

Examples:

```text
HTTP

80

HTTPS

443

Redis

6379
```

These stay open, waiting for clients.

---

### Ephemeral Ports

Clients usually don't choose fixed ports.

The operating system automatically assigns a temporary, available port for each outgoing connection.

Once the communication ends, that port can be reused for future connections.

---

# Putting Everything Together

Suppose you open YouTube.

Here's what happens at a high level.

```text
Browser

↓

DNS finds YouTube's IP

↓

TCP connection established

↓

Client

192.168.1.5:52134

↓

Server

142.250.x.x:443

↓

HTTPS Request

↓

Video Response
```

Notice how every layer contributes something:

- IP gets the packet to the correct machine.
- Port gets it to the correct application.
- Socket identifies the communication endpoint.
- TCP or UDP determines how the data is transported.

---

# 5. Types / Variations

## IP Addresses

| Type       | Purpose                     |
| ---------- | --------------------------- |
| Private IP | Used inside local networks  |
| Public IP  | Reachable from the Internet |

---

## Ports

| Type                   | Purpose                             |
| ---------------------- | ----------------------------------- |
| Well-known / Listening | Servers accept incoming connections |
| Ephemeral              | Temporary ports assigned to clients |

---

## Transport

Sockets exist for both:

- TCP
- UDP

The transport protocol is part of the communication endpoint.

---

# 6. Real-World Usage

### Web Browsers

When you open multiple tabs, each connection typically uses a different ephemeral client port, allowing the operating system to distinguish between concurrent network conversations.

---

### Database Servers

Databases expose listening ports.

For example:

- PostgreSQL commonly listens on **5432**.
- MySQL commonly listens on **3306**.

Applications connect by specifying both the server's IP address and its listening port.

---

### Redis

Redis commonly listens on **6379**. Many backend services communicate with Redis by connecting to that IP/port combination.

---

### Kubernetes

Even though containers may have private IPs internally, Kubernetes Services expose stable network endpoints and ports so applications can communicate without worrying about the underlying container lifecycle.

---

# 7. Where It Helps

Understanding IPs, ports, and sockets helps you:

- Understand how multiple services run on one machine.
- Configure servers correctly.
- Debug connectivity issues.
- Read system architecture diagrams.
- Understand APIs, databases, caches, and microservices.

---

# 8. Where It Doesn't Help

Knowing addresses and ports tells you **where** data should go.

It does not explain:

- How the data is encrypted.
- What the request contains.
- How APIs are structured.
- Whether communication is reliable.

Those concerns belong to higher layers and protocols.

---

# 9. Mental Model

Think of a large office building.

```text
Building Address
        │
     Reception
        │
Department Number
        │
Specific Employee
```

In networking:

- **IP address** = Building address.
- **Port** = Department or apartment number.
- **Socket** = One employee talking to another employee through a dedicated phone line.

The address gets you to the building.

The department number gets you to the right room.

The phone call represents the active communication.

---

# 10. Tradeoffs

## Advantages

- Separates machine identification from application identification.
- Allows many applications to share one network connection.
- Enables multiple simultaneous communications.
- Provides a standardized addressing scheme across the Internet.

## Disadvantages

- Port conflicts can prevent applications from starting.
- NAT adds complexity because private devices aren't directly reachable from the public Internet.
- Firewalls often need explicit rules allowing traffic to specific ports.
- IP addresses can change, making stable naming systems like DNS necessary.

---

# 11. Common Interview Questions

### Why isn't an IP address enough?

An IP address identifies a machine, but modern machines run many applications simultaneously. Ports identify which application should receive incoming data.

---

### What is the difference between an IP address and a port?

An IP address identifies a network interface on a device. A port identifies a specific service or application running on that device.

---

### What is a socket?

A socket is one endpoint of a network communication, typically identified by an IP address, a port number, and a transport protocol. A network connection is commonly described by the combination of the client and server socket endpoints.

---

### Why do clients use ports?

Clients often maintain multiple simultaneous network connections. Temporary (ephemeral) ports allow the operating system to distinguish between them and route responses correctly.

---

### Why can't two applications listen on the same port?

The operating system needs a unique owner for each listening port (for a given IP address and transport protocol). Otherwise, it wouldn't know which application should receive incoming traffic.

---

# 12. Before vs After Architecture

### Using Only IP Addresses

```text
Client
    │
192.168.1.5
    │
Server
```

The packet reaches the server, but the operating system doesn't know which application should receive it.

↓

### IP + Port

```text
Client
192.168.1.5:52134
        │
        │
Server
203.0.113.20:443
```

The packet reaches both the correct machine **and** the correct application.

---

# 13. Connections

We've now answered an important question:

- **IP addresses** locate machines.
- **Ports** locate applications.
- **Sockets** identify communication endpoints.

But another question naturally arises.

So far, every packet has traveled across the network **in plain form**.

Imagine you're logging into your bank.

Your password travels from your laptop to the server.

> **What stops someone on the network from reading or modifying that data?**

To solve this problem, the Internet relies on **TLS (formerly SSL)**—the technology that powers **HTTPS** and enables secure communication over untrusted networks.

That's the focus of our next chapter.
