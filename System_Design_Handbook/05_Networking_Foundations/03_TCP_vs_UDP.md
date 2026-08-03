# Chapter 3 — TCP vs UDP

---

# Goal

Understand why the Internet has two major transport protocols, what problems each solves, and how engineers decide which one to use in real systems.

---

# 1. The Problem

Imagine you're watching a movie on Netflix.

Everything is going smoothly.

Suddenly, your Wi-Fi drops for half a second.

When the connection returns, you expect the movie to continue from where it left off.

You **don't** want:

- Random missing scenes
- Scrambled subtitles
- Video frames arriving in the wrong order

Now consider a completely different application.

You're playing an online multiplayer game.

Your character moves left.

One network packet is lost.

Should the game stop and wait for that missing packet?

Absolutely not.

By the time the packet arrives, your character has already moved somewhere else.

Waiting would make the game feel laggy.

Now consider a third application.

You're on a Zoom call.

If one word is lost, you'd rather miss that word than freeze the entire conversation waiting for it.

These three applications have very different priorities:

| Application   | Most Important Requirement  |
| ------------- | --------------------------- |
| Netflix       | Correct and complete data   |
| Online Gaming | Lowest possible latency     |
| Video Calls   | Smooth real-time experience |

One transport protocol cannot optimize for all three simultaneously.

---

# 2. Why Existing Solutions Fail

In the previous chapter, we learned about the **Transport Layer**.

We said it is responsible for end-to-end communication between applications.

But we never answered:

- What if packets are lost?
- What if packets arrive out of order?
- What if packets are duplicated?
- What if the receiver is slower than the sender?
- Should missing packets be retransmitted?
- Is waiting for retransmission always a good idea?

Different applications answer these questions differently.

A banking transaction and a live football stream have very different needs.

So instead of one transport protocol, the Internet provides two.

---

# 3. The Big Idea

> **TCP prioritizes reliability. UDP prioritizes speed.**

Everything else is a consequence of that design decision.

---

# 4. Detailed Explanation

## First, Remember Packets

In the previous chapter, we learned that data is broken into packets.

Suppose your application sends this message:

```text
Hello World
```

Instead of one large block:

```text
Hello World
```

The transport layer may send:

```text
Packet 1 → Hello

Packet 2 → World
```

Now imagine the network behaves unexpectedly.

Possible problems include:

```text
Packet 1 ✓

Packet 2 Lost
```

or

```text
Packet 2 arrives first

Packet 1 arrives later
```

or

```text
Packet 2 arrives twice
```

The transport layer must decide what to do.

Different protocols make different decisions.

---

# TCP

TCP stands for:

> **Transmission Control Protocol**

TCP assumes:

> **Every byte of data matters.**

Its goal is:

- Deliver everything
- Deliver it once
- Deliver it in order

Even if that means waiting.

---

## How TCP Thinks

Imagine sending ten packets.

```text
1
2
3
4
5
6
7
8
9
10
```

Suppose packet 5 is lost.

The receiver gets:

```text
1
2
3
4
6
7
8
9
10
```

Instead of saying:

"Close enough."

TCP says:

"I'm missing packet 5."

It requests packet 5 again.

Only after packet 5 arrives does it deliver the complete data to the application.

```text
1
2
3
4
5
6
7
8
9
10
```

This guarantees correctness.

But it also introduces delay.

---

# Analogy: Registered Courier

Imagine you're sending legal documents.

The courier:

- Tracks every package.
- Confirms delivery.
- Resends lost packages.
- Ensures pages stay in order.

This takes longer.

But you can trust the delivery.

That's TCP.

---

# Features of TCP

## Reliable Delivery

Packets are retransmitted if lost.

---

## Ordered Delivery

Even if packets arrive:

```text
3
1
2
```

The application receives:

```text
1
2
3
```

---

## Duplicate Detection

If packet 4 arrives twice:

```text
4
4
```

TCP removes the duplicate before passing data to the application.

---

## Flow Control

Imagine:

Sender:

```text
1000 MB/s
```

Receiver:

```text
20 MB/s
```

Without coordination:

The receiver becomes overwhelmed.

TCP allows the receiver to say:

> "Slow down."

This prevents buffer overflow and excessive packet loss.

---

## Congestion Control

Now imagine millions of users sending traffic simultaneously.

The network itself becomes overloaded.

TCP doesn't simply keep sending faster.

Instead, it reduces its sending rate when it detects congestion, helping prevent network collapse.

This is one of the reasons the Internet remains stable under heavy load.

We'll keep the details of congestion algorithms out of this chapter—they deserve their own discussion.

---

# TCP Has a Cost

Every reliability feature requires extra work.

TCP performs:

- A connection setup before data transfer.
- Acknowledgments (ACKs) for received data.
- Retransmissions for lost packets.
- Sequence tracking.
- Congestion management.
- Flow control.

All of this increases latency.

---

# UDP

UDP stands for:

> **User Datagram Protocol**

UDP takes a very different approach.

It says:

> "I'll send the packet. What happens afterward is not my concern."

There are:

- No acknowledgments.
- No retransmissions.
- No ordering guarantees.
- No built-in congestion control.
- No flow control.

Just send.

---

# Analogy: Postcards

Imagine sending postcards.

You drop them into a mailbox.

You don't know:

- Whether they arrived.
- In what order they arrived.
- Whether one was lost.

But they're fast and inexpensive.

That's UDP.

---

# What Happens If a Packet Is Lost?

Suppose:

```text
Packet 1 ✓

Packet 2 Lost

Packet 3 ✓
```

TCP says:

```text
Wait.

Resend Packet 2.
```

UDP says:

```text
Continue.
```

No waiting.

No retransmission.

The application decides whether that missing packet matters.

---

# Why Would Anyone Want UDP?

Because sometimes waiting is worse than losing data.

Imagine a live football match.

Suppose one video frame is lost.

Would you rather:

Option A:

Skip one frame.

or

Option B:

Freeze the entire video for two seconds waiting for that frame.

Most people prefer Option A.

Similarly:

During a video call,

missing one syllable is less noticeable than freezing the conversation.

Real-time applications value freshness over perfection.

---

# TCP vs UDP Example

Imagine saying:

```text
"The meeting starts at 10."
```

---

### TCP

If one word is lost:

```text
The meeting _____ at 10
```

TCP waits until the missing word arrives.

Only then does it deliver the sentence.

---

### UDP

UDP immediately delivers:

```text
The meeting at 10
```

Not perfect.

But immediate.

---

# Connection-Oriented vs Connectionless

Another major difference is **how communication begins**.

## TCP

Before exchanging data, both sides establish a connection.

Conceptually:

```text
Client

↓

Can we talk?

↓

Server

↓

Yes

↓

Start sending data
```

This setup allows both sides to agree on communication parameters before exchanging information.

We'll study the famous **three-way handshake** in a later chapter.

---

## UDP

UDP doesn't establish a connection.

It simply sends packets.

```text
Client

↓

Packet Sent
```

No negotiation.

No setup.

This reduces latency.

---

# When Should You Use TCP?

Whenever correctness is critical.

Examples:

- Banking
- Payments
- Email
- File downloads
- Database replication
- REST APIs
- Login systems
- E-commerce orders

Imagine Amazon.

You click:

> Buy Now

You definitely don't want the order packet to disappear.

---

# When Should You Use UDP?

Whenever timeliness is more important than perfection.

Examples:

- Video calls
- Voice calls
- Live streaming (in many scenarios)
- Online multiplayer games
- IoT sensor data
- DNS lookups (we'll learn why in Module 6)

Here, a small amount of data loss is often acceptable if it keeps communication fast and responsive.

---

# Can Applications Add Reliability to UDP?

Yes.

This is an important engineering decision.

UDP intentionally stays simple.

If an application needs:

- Retries
- Ordering
- Error correction

it can implement only the features it needs.

This avoids paying the full cost of TCP when complete reliability isn't necessary.

A modern example is **QUIC**, which powers HTTP/3. It uses UDP as its foundation while implementing reliability, congestion control, and other features in user space. We'll revisit this idea in Module 6.

---

# 5. Types / Variations

There are two major transport protocols you'll encounter most often.

| Protocol | Philosophy                      |
| -------- | ------------------------------- |
| TCP      | Reliable, ordered communication |
| UDP      | Fast, best-effort communication |

The choice depends on the application's priorities, not on which protocol is "better."

---

# 6. Real-World Usage

### Web Browsing

Traditional HTTP/1.1 and HTTP/2 use TCP because webpages, JavaScript, CSS, and images must arrive correctly.

---

### Netflix

Video streaming often favors smooth playback over perfect delivery. Modern streaming techniques may tolerate some packet loss or recover in application-specific ways rather than freezing playback.

---

### Zoom and Google Meet

Real-time audio and video typically use UDP-based transport because delayed packets are often less useful than simply continuing the conversation.

---

### Online Gaming

Games such as Fortnite or Valorant use UDP for frequent position updates. If one position update is lost, the next update quickly replaces it.

---

### Databases

Replication, transactions, and client communication commonly use TCP because every piece of data must be accurate and in order.

---

# 7. Where It Helps

## TCP

Best when you need:

- Reliable delivery.
- Ordered data.
- Guaranteed completeness.
- Error recovery.
- Consistent application behavior.

---

## UDP

Best when you need:

- Low latency.
- Minimal overhead.
- Continuous real-time updates.
- High throughput for time-sensitive data.

---

# 8. Where It Doesn't Help

## TCP Limitations

- Higher latency due to acknowledgments and retransmissions.
- Connection setup adds overhead.
- Can delay delivery while waiting for missing packets.

---

## UDP Limitations

- No guarantee that packets arrive.
- No guarantee of ordering.
- No built-in recovery for lost packets.
- Applications must handle reliability themselves if needed.

---

# 9. Mental Model

Imagine sending information in two different ways.

### TCP

Registered courier.

```text
Package

↓

Tracking

↓

Signature Required

↓

Delivered Correctly
```

Reliable, but slower.

---

### UDP

Throwing flyers from a helicopter.

```text
Flyers

↓

Dropped

↓

Some Reach People

↓

Some Blow Away
```

Fast, but with no guarantees.

The best choice depends on whether every message matters or whether getting the latest information quickly matters more.

---

# 10. Tradeoffs

## TCP

### Advantages

- Reliable delivery.
- Ordered communication.
- Handles packet loss automatically.
- Prevents overwhelming the receiver.
- Adapts to network congestion.

### Disadvantages

- Higher latency.
- More protocol overhead.
- Connection setup before communication.
- Retransmissions can slow time-sensitive applications.

---

## UDP

### Advantages

- Extremely low latency.
- Minimal overhead.
- No connection setup.
- Well suited for real-time communication.

### Disadvantages

- No delivery guarantees.
- No ordering guarantees.
- Lost packets are not recovered automatically.
- Applications bear the responsibility for any required reliability.

---

# 11. Common Interview Questions

### Why can't everything simply use TCP?

Because many applications prioritize low latency over perfect reliability. Waiting for retransmissions would degrade experiences like gaming or live video.

---

### Is UDP always faster than TCP?

UDP has less protocol overhead, but overall application performance depends on the workload. If your application repeatedly reimplements reliability on top of UDP, the difference may shrink.

---

### Why do databases use TCP?

Databases require every byte of data to arrive correctly and in order. Losing or reordering data could corrupt state or produce incorrect results.

---

### Why do video calls typically use UDP?

Late audio or video packets are often useless. It's usually better to continue with fresh data than to pause the conversation waiting for missing packets.

---

### Can UDP be made reliable?

Yes. Applications can implement acknowledgments, retransmissions, ordering, or forward error correction themselves when they need those features. QUIC is a prominent example of this design approach.

---

# 12. Before vs After Architecture

### One Protocol for Everything (Hypothetical)

```text
Applications
      │
Single Transport Protocol
      │
Internet
```

Every application would inherit the same tradeoffs, whether appropriate or not.

↓

### Two Specialized Protocols

```text
                Application
                     │
          ┌──────────┴──────────┐
          │                     │
         TCP                   UDP
          │                     │
          └──────────┬──────────┘
                     │
                  Internet
```

Applications choose the transport protocol that best matches their requirements.

---

# 13. Connections

Now we know **how applications choose to transport data**:

- TCP provides reliable, ordered communication.
- UDP provides fast, best-effort communication.

But another question naturally follows:

> **When a packet finally reaches the destination machine, how does the operating system know which application should receive it?**

If your laptop is simultaneously running:

- Chrome
- Spotify
- Slack
- VS Code
- Zoom

all using the same network connection, how are incoming packets delivered to the correct application?

To answer that, we need to understand **IP addresses, ports, and sockets**—the addressing system that allows billions of applications to communicate without interfering with one another.

That's the focus of our next chapter.
