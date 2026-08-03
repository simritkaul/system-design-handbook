# Module 2 — Chapter 5 — Consistency Models

> **Goal:** Understand the different consistency guarantees a distributed system can provide, the tradeoffs between them, and when each model is appropriate.

---

# 1. The Problem

Imagine Instagram.

You update your profile picture.

```text
Old Picture

↓

New Picture
```

You immediately refresh the app.

Should you always see

the new picture?

Hopefully yes.

Now your friend refreshes.

Should they also see

the new picture?

Maybe.

Maybe not.

That depends on

the system's

**consistency model**.

---

# 2. The Big Idea

A consistency model answers one question:

> **"After data changes, what guarantees does the system make about what different users will observe?"**

Not every application needs the same guarantee.

A bank and Instagram have very different requirements.

---

# 3. Strong Consistency

This is the easiest model to understand.

Rule

> Once a write succeeds, every future read sees that latest value.

Example

Alice changes her password.

Immediately afterward,

every server returns

the new password.

No exceptions.

---

Timeline

```text
10:00

Password = ABC

↓

10:01

Update → XYZ

↓

10:01:01

Every read

↓

XYZ
```

Nobody ever sees

ABC

again.

---

### Where It Is Used

- Banking
- Payments
- Inventory
- Airline seat booking

Anywhere stale data could cause serious problems.

---

### Tradeoff

Achieving this often means coordinating multiple machines before confirming a write.

That increases latency and can reduce availability during failures.

---

# 4. Eventual Consistency

Now imagine Instagram.

Alice uploads a new photo.

Replica A updates immediately.

Replica B updates

two seconds later.

Users connected to Replica B

still see

the old feed

for a short time.

Eventually

every replica agrees.

Hence

**Eventual Consistency.**

---

Timeline

```text
10:00

Old Picture

↓

10:01

Write

↓

Replica A ✔

↓

Replica B (still old)

↓

Replica B updates

↓

Everyone agrees
```

---

### Where It Is Used

- Social media
- Product reviews
- News feeds
- Analytics
- Recommendations

---

### Tradeoff

Fast and highly available.

But temporary stale reads are possible.

---

# 5. Read-After-Write Consistency

Imagine you edit your own profile.

You press Save.

Then immediately refresh.

Seeing the old profile would feel broken.

Read-after-write consistency guarantees:

> **After _you_ successfully write data, _you_ will always see your latest write.**

Other users may still see stale data for a short period.

This model improves user experience without requiring full strong consistency.

---

### Example

You post on LinkedIn.

You immediately see your post.

Your colleague may not see it for another second.

That's acceptable.

---

# 6. Monotonic Read Consistency

Imagine checking your bank balance.

First read:

```text
₹1,500
```

A minute later,

you read again.

You should never see

```text
₹1,200
```

That would feel like time moved backwards.

Monotonic reads guarantee:

> Once you've seen a newer value, you won't later observe an older one.

---

### Example

Feed Version 10

↓

Feed Version 12

↓

Never again

↓

Feed Version 10

---

# 7. Monotonic Write Consistency

Suppose you edit a document.

First write

```text
Draft V1
```

Then

```text
Draft V2
```

The system shouldn't apply

V2

before

V1.

Writes from the same client must be processed in order.

---

### Example

Chat messages.

If you send

```text
Hello

↓

How are you?
```

Nobody should receive

"How are you?"

before

"Hello."

---

# 8. Causal Consistency

Some operations depend on others.

Example

Alice posts

```text
I got promoted!
```

Bob comments

```text
Congratulations!
```

It would be strange if someone saw

Bob's comment

before

Alice's post.

Causal consistency preserves

cause-and-effect relationships.

Independent operations may still appear in different orders.

---

# 9. Linearizability vs Sequential Consistency

You don't need to master these for every interview, but you should know the difference.

### Linearizability

Acts as if every operation happened instantly at one precise point in real time.

Very strong guarantee.

---

### Sequential Consistency

Everyone observes the same order of operations,

but that order doesn't have to match real-world time exactly.

Slightly weaker.

---

# 10. Visual Comparison

| Consistency Model | Guarantee                                             |
| ----------------- | ----------------------------------------------------- |
| Strong            | Everyone immediately sees the latest write            |
| Eventual          | Everyone will agree eventually                        |
| Read-After-Write  | The writer always sees their own latest write         |
| Monotonic Reads   | Once you see new data, you never see older data later |
| Monotonic Writes  | Writes from one client are applied in order           |
| Causal            | Cause-and-effect ordering is preserved                |

---

# 11. Which One Should I Choose?

Think about the application.

### Banking

Money.

Strong consistency.

---

### Flight Booking

Seats.

Strong consistency.

---

### Instagram Feed

Eventual consistency.

---

### WhatsApp

Messages should appear in order.

Monotonic writes and causal consistency are important.

---

### Google Docs

Users expect edits to make logical sense.

Ordering and causality matter.

---

# 12. Mental Model

Imagine newspapers.

One printing press updates the headline.

Some cities receive the new edition immediately.

Others receive yesterday's edition for a few hours.

Eventually,

every city has today's newspaper.

That's eventual consistency.

Strong consistency would require nobody receiving any newspaper until every city could get today's edition simultaneously.

---

# 13. Tradeoffs

### Strong Consistency

Advantages

- Always correct
- Easy to reason about
- No stale reads

Disadvantages

- Higher latency
- Lower availability during failures
- More coordination

---

### Eventual Consistency

Advantages

- Fast
- Highly available
- Scales well

Disadvantages

- Temporary stale data
- More complex application logic in some cases

---

# 14. Real-World Examples

### Bank Balance

Strong consistency.

---

### Amazon Product Reviews

Eventual consistency.

---

### Instagram Likes

Eventual consistency.

A like count being off by one for a second is acceptable.

---

### Gmail

Read-after-write.

After sending an email,

you expect to immediately see it in your "Sent" folder.

---

### Slack / WhatsApp

Messages should preserve order and causality.

---

# 15. Common Interview Questions

- What is a consistency model?
- Strong vs eventual consistency?
- What is read-after-write consistency?
- What are monotonic reads?
- What is causal consistency?
- Why doesn't every application use strong consistency?
- Which consistency model would you choose for a social network? For a banking system?

---

# 16. Connections

Notice how the last three chapters fit together.

**Replication** introduced multiple copies of data.

↓

**CAP Theorem** explained that network partitions force tradeoffs.

↓

**Consistency Models** describe exactly what guarantees users receive despite those tradeoffs.

Now we can finally answer questions like:

> "Why did my friend see my Instagram post two seconds later than I did?"

or

> "Why can two users temporarily see different like counts?"

Those are consistency model decisions, not bugs.

---

# Key Takeaways

- A consistency model defines **what guarantees a distributed system makes about reads after writes**.
- **Strong consistency** prioritizes correctness but often requires more coordination.
- **Eventual consistency** prioritizes availability and performance while accepting temporary stale data.
- Specialized guarantees like **read-after-write**, **monotonic reads**, **monotonic writes**, and **causal consistency** improve user experience without always requiring full strong consistency.
- Choosing the right consistency model depends on the application's requirements, not on a universally "best" option.

---

## Before we move to the next chapter

I think we've now completed the **core distributed systems theory**:

- ✅ Replication
- ✅ Sharding
- ✅ Consistent Hashing
- ✅ CAP Theorem
- ✅ Consistency Models

The next chapter—**Caching**—is where theory starts turning into architecture.

And unlike the previous chapters, caching is something you'll use in almost every HLD interview:

- Twitter
- Instagram
- Netflix
- YouTube
- DoorDash
- Uber
- Amazon
- TinyURL
- Rate Limiter

In my opinion, **Caching** deserves to be one of the longest chapters in the entire handbook because it's not just one concept. It includes cache-aside, write-through, write-back, write-around, TTL, invalidation, cache stampedes, hot keys, cache warming, eviction policies, and more. It's essentially a module by itself rather than a single topic.
