# Chapter 5 — TLS / SSL (HTTPS)

---

# Goal

Understand why encryption is necessary on the Internet, how TLS enables secure communication, and why HTTPS has become the standard for modern web applications.

---

# 1. The Problem

Imagine you're logging into your bank.

You enter:

```text
Username: alice@example.com
Password: MySecretPassword123
```

Then you click **Login**.

Your browser sends the request over the Internet.

Remember what we've learned so far:

```text
Browser
    │
Wi-Fi
    │
Home Router
    │
ISP
    │
Internet
    │
Bank Server
```

Your request passes through many devices before reaching the bank.

Now imagine one of those devices can read everything passing through it.

If your password is sent in plain text, anyone monitoring the network could see:

```text
Username: alice@example.com
Password: MySecretPassword123
```

That would make online banking impossible.

The same applies to:

- Credit card payments
- Emails
- WhatsApp Web
- GitHub logins
- Amazon purchases

The Internet itself is **not a trusted network**.

So how do we safely send sensitive information across it?

---

# 2. Why Existing Solutions Fail

So far we've learned:

- Networks move packets.
- TCP delivers them reliably.
- IP finds the destination machine.
- Ports find the destination application.

But none of these answer an important question:

> **Can someone read or modify my data while it's traveling?**

TCP guarantees delivery.

It does **not** guarantee secrecy.

IP guarantees routing.

It does **not** guarantee privacy.

Anyone controlling part of the network could potentially inspect or tamper with unencrypted traffic.

We need a way to make the data unreadable to everyone except the intended recipient.

---

# 3. The Big Idea

> **TLS creates a secure, encrypted communication channel between two applications over an untrusted network.**

Even if someone intercepts the traffic, they should not be able to understand or alter it.

---

# 4. Detailed Explanation

## Imagine Sending a Letter

Suppose you send your friend a postcard.

```text
Hi!
See you tomorrow.
```

Everyone who handles the postcard can read it.

- Postal workers
- Anyone who finds it
- Anyone who intercepts it

Now imagine instead you place the letter inside a locked box.

Only someone with the correct key can open it.

Even if someone steals the box, they cannot read the contents.

TLS works like that locked box.

---

# HTTP vs HTTPS

Without encryption:

```text
Browser
      │
HTTP
      │
Internet
      │
Server
```

Data travels in plain text.

Anyone on the path can inspect it.

---

With TLS:

```text
Browser
      │
HTTPS
      │
Internet
      │
Server
```

The data is encrypted before leaving your computer.

Only the destination server can decrypt it.

---

# Wait... What Is HTTPS?

Many people think HTTPS is a completely different protocol.

It isn't.

Think of it like this:

```text
HTTPS

=

HTTP

+

TLS
```

HTTP still defines:

- GET
- POST
- Headers
- Status codes

TLS simply protects that communication.

---

# What Does TLS Actually Provide?

TLS provides three major guarantees.

## 1. Confidentiality

Only the sender and receiver can read the data.

Example:

Without TLS:

```text
Password=Secret123
```

With TLS:

```text
A9X7LQ82J...F1P0K...
```

Anyone intercepting the packets only sees encrypted data.

---

## 2. Integrity

Suppose an attacker intercepts a payment request.

Original request:

```text
Transfer ₹100
```

The attacker changes it to:

```text
Transfer ₹100000
```

TLS includes mechanisms that allow the receiver to detect if the encrypted data was modified during transmission.

If anything changes, the communication is rejected.

The receiver knows:

> "This message has been tampered with."

---

## 3. Authentication

Encryption alone is not enough.

Imagine an attacker creates a fake banking website.

You connect.

The connection is encrypted.

Great.

But...

You're securely talking to the attacker.

TLS also helps verify **who you're communicating with**.

Before sending sensitive information, your browser checks whether the server is actually the one it claims to be.

This is where certificates come in.

---

# Certificates

Suppose someone claims:

> "I'm Amazon."

How do you verify that?

In real life:

You might ask for an ID card.

On the Internet, servers present a **digital certificate**.

Think of it as an identity card.

```text
Website

↓

Certificate

↓

"I'm amazon.com"
```

Your browser verifies whether that certificate is trustworthy before establishing a secure connection.

---

# Certificate Authorities (CAs)

Who issues these certificates?

Not the websites themselves.

Imagine if everyone printed their own passport.

Nobody would trust it.

Instead, trusted organizations called **Certificate Authorities (CAs)** verify website ownership and issue certificates.

Your browser already contains a list of trusted Certificate Authorities.

When you visit:

```text
https://amazon.com
```

Your browser asks:

> "Is this certificate signed by a CA I trust?"

If yes:

Continue.

If not:

Show a warning.

This is why browsers sometimes display messages like:

```text
Your connection is not private.
```

Usually, the certificate cannot be trusted or has another security issue.

---

# What Happens When You Visit an HTTPS Website?

At a high level:

```text
Browser

↓

Hello

↓

Server

↓

Certificate

↓

Browser Verifies Identity

↓

Agree on Encryption

↓

Secure Communication Begins
```

This initial negotiation is called the **TLS Handshake**.

The handshake itself is a fascinating topic, but for high-level design, it's enough to understand its purpose:

- Verify identities.
- Agree on how to encrypt communication.
- Establish a shared secure session.

After that, normal HTTP requests flow through the encrypted channel.

---

# Why Not Encrypt Everything with One Secret Key?

Suppose the browser and server both use the same secret password.

Question:

How do they agree on that password in the first place?

If they simply send it across the Internet...

An attacker can steal it.

This is known as the **key exchange problem**.

Modern TLS solves this using public-key cryptography during the handshake.

Once both sides have securely established a shared session key, they switch to much faster symmetric encryption for the rest of the communication.

The important engineering idea is:

- Public-key cryptography solves the initial trust problem.
- Symmetric encryption handles the bulk of the data efficiently.

This combination provides both security and performance.

---

# What Does Encryption Protect?

TLS encrypts the **contents** of the communication.

For example:

```text
POST /login

Username

Password

Cookies

Headers

Response Body
```

These are protected.

However, some metadata must remain visible so the network can deliver packets, such as IP addresses and routing information.

Routers still need to know where the packet is going.

TLS protects application data, not the underlying network infrastructure.

---

# Why HTTPS Matters in System Design

Nearly every modern distributed system relies on HTTPS.

Consider this architecture:

```text
User
   │
API Gateway
   │
Microservices
   │
Databases
```

Communication may occur:

- User → API Gateway
- API Gateway → Service A
- Service A → Service B
- Service B → Payment Service

Organizations decide which links require encryption.

Communication crossing public networks almost always uses TLS.

Many organizations also encrypt internal service-to-service communication, especially in zero-trust architectures.

---

# SSL vs TLS

You'll often hear both terms.

Historically:

```text
SSL

↓

TLS
```

TLS is the modern, more secure successor.

SSL versions are obsolete and should not be used.

However, people still casually say:

> "SSL Certificate"

Even though they usually mean a TLS certificate.

This terminology persists for historical reasons.

---

# 5. Types / Variations

## HTTP

```text
Application Data

↓

Plain Text

↓

Internet
```

Fast, but insecure.

---

## HTTPS

```text
Application Data

↓

TLS Encryption

↓

Internet
```

Secure communication over an untrusted network.

---

# 6. Real-World Usage

### Banking

Every login, balance check, and money transfer is protected by TLS to ensure confidentiality, integrity, and authentication.

---

### Amazon

Customer accounts, shopping carts, payments, and order histories all rely on HTTPS for secure communication.

---

### GitHub

Source code, authentication tokens, and Git operations are transmitted over HTTPS (or SSH for some workflows) to prevent interception and tampering.

---

### Google

Services like Gmail, Google Drive, and Search all use HTTPS by default. Modern browsers increasingly mark plain HTTP websites as "Not Secure."

---

### Internal Microservices

Many companies encrypt communication between internal services using mutual TLS (mTLS), where both the client and the server authenticate each other.

---

# 7. Where It Helps

TLS provides:

- Secure logins.
- Secure payments.
- Protection against eavesdropping.
- Detection of tampered data.
- Verification that you're communicating with the intended server.

It enables users to trust Internet services despite communicating over networks they do not control.

---

# 8. Where It Doesn't Help

TLS does **not** protect against every security threat.

For example, it cannot prevent:

- Bugs in application code.
- SQL Injection.
- Cross-Site Scripting (XSS).
- Stolen passwords from phishing attacks.
- Malware on the user's device.

TLS secures communication—not the application itself.

---

# 9. Mental Model

Imagine sending valuables through the mail.

Without TLS:

```text
Money

↓

Transparent Envelope

↓

Everyone Can See It
```

With TLS:

```text
Money

↓

Locked Safe

↓

Courier Delivers Safe

↓

Only Recipient Has the Key
```

The postal system still delivers the package.

It simply cannot see what's inside.

The Internet works the same way.

---

# 10. Tradeoffs

## Advantages

- Protects sensitive data from eavesdropping.
- Detects tampering during transmission.
- Verifies the identity of servers.
- Builds user trust.
- Required for many modern browser features and security standards.

## Disadvantages

- The TLS handshake adds some latency before communication begins.
- Encryption and decryption consume CPU resources.
- Certificates must be issued, renewed, and managed.
- Misconfigured certificates can cause connection failures.

In practice, these costs are usually far outweighed by the security benefits.

---

# 11. Common Interview Questions

### Why isn't HTTP secure?

HTTP sends application data in plain text. Anyone able to observe the traffic can potentially read or modify it.

---

### What is HTTPS?

HTTPS is HTTP running over a TLS-secured connection. HTTP defines the application protocol, while TLS provides encryption, integrity, and authentication.

---

### What are the three primary goals of TLS?

- **Confidentiality:** Keep data secret.
- **Integrity:** Detect tampering.
- **Authentication:** Verify the identity of the communicating party.

---

### Why do browsers trust some certificates but not others?

Browsers include a list of trusted Certificate Authorities. Certificates signed by those trusted authorities are generally accepted; others may trigger warnings.

---

### Why doesn't TLS use public-key encryption for all communication?

Public-key cryptography is computationally expensive. TLS uses it mainly during the handshake to establish a shared secret, then switches to faster symmetric encryption for the actual data transfer.

---

# 12. Before vs After Architecture

### Without TLS

```text
User
   │
HTTP
   │
Internet
   │
Server
```

Anyone on the network may be able to inspect or alter the application data.

↓

### With TLS

```text
User
   │
HTTPS (HTTP + TLS)
   │
Internet
   │
Server
```

The network still transports the packets, but the application data remains encrypted end-to-end between the communicating applications.

---

# 13. Connections

So far, we've learned how a request:

- Travels across networks.
- Uses TCP or UDP for transport.
- Reaches the correct machine using IP.
- Reaches the correct application using ports.
- Is protected using TLS.

But we've treated an HTTP request as a single abstract action.

What actually happens from the moment you type:

```text
https://www.amazon.com
```

until the homepage appears?

The browser performs dozens of steps:

- Resolving the domain name.
- Establishing network connections.
- Performing the TLS handshake.
- Sending HTTP requests.
- Receiving responses.
- Downloading CSS, JavaScript, and images.
- Rendering the page.

In the next chapter, we'll trace the **complete HTTP Request Lifecycle**, tying together everything we've learned so far into one end-to-end journey.
