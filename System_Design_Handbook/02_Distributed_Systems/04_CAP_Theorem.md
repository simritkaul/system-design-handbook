# Module 2 — Chapter 4 — CAP Theorem

> **Goal:** Understand why CAP exists, what a network partition is, why distributed systems cannot guarantee all three properties simultaneously, and how this influences system design.

---

# 1. The Story

Imagine we have only one database.

```text
Application

↓

Database
```

Everything is simple.

No replication.

No distributed system.

No CAP theorem.

Why?

Because there's only one machine.

CAP only becomes relevant when you have **multiple machines that store the same data**.

---

# 2. The Problem

Let's add replication.

```text
          Primary

          /     \

Replica A      Replica B
```

Now we have multiple copies.

This is great.

More reads.

Higher availability.

Fault tolerance.

Life is good.

---

One day...

the network cable between the databases breaks.

```text
Primary   X   Replica
```

The machines are still alive.

They simply

cannot communicate.

This situation is called a

**Network Partition.**

---

# 3. What Is a Network Partition?

Imagine two friends.

One lives in Delhi.

One lives in Mumbai.

Their phones stop working.

Both people are alive.

Neither can talk to the other.

That's exactly what happens in distributed systems.

The machines haven't crashed.

Communication has.

---

# 4. Now Comes the Difficult Question

Suppose a user changes their password.

The request reaches

Primary.

Primary updates

the password.

But...

Replica never receives the update

because the network is broken.

Now another user logs in.

They read from

Replica.

Replica still has

the old password.

What should the system do?

---

# 5. The Three Choices

Every distributed system wants these three things.

---

## Consistency (C)

Every user should see the same data.

If I update my profile picture,

everyone should immediately see the new one.

No stale data.

---

## Availability (A)

Every request should receive a response.

Even if some machines are down.

The system should continue serving users.

---

## Partition Tolerance (P)

The system should continue operating even when machines cannot communicate because of network failures.

---

# 6. The Impossible Situation

Suppose

the network is broken.

```text
Primary   X   Replica
```

A write reaches

Primary.

Replica never receives it.

Now another user sends

a read request

to Replica.

What should Replica do?

There are only two options.

---

### Option 1

Replica answers.

```text
Old Password
```

The system is

Available.

But...

the data is inconsistent.

---

### Option 2

Replica refuses.

```text
Error

Try Again Later
```

Now users receive no stale data.

Consistency is preserved.

But

Availability is lost.

---

There is no third option.

---

# 7. The Big Idea

Eric Brewer's CAP Theorem states:

> When a network partition occurs, a distributed system can choose either:

- Consistency

or

- Availability

But not both simultaneously.

Notice something important.

It doesn't say

you choose two out of three

all the time.

It says

**when a partition happens**, you must choose between consistency and availability.

Since real distributed systems cannot prevent network partitions entirely, **Partition Tolerance is generally a requirement**, leaving the practical choice between Consistency and Availability during a partition.

---

# 8. CP Systems

Choose

Consistency

-

Partition Tolerance.

Example

```text
Write

↓

Primary

↓

Replica unreachable

↓

Reject request
```

The system prefers

correct data

over

serving every request.

---

Good for

Banking

Payments

Inventory

---

# 9. AP Systems

Choose

Availability

-

Partition Tolerance.

Example

```text
Primary updated

↓

Replica unavailable

↓

Replica still answers

(old data)
```

Users always receive a response.

Eventually,

the replica catches up.

---

Good for

Social Media

News Feeds

Recommendations

---

# 10. CA Systems?

People often ask

> "Can I build a CA system?"

If there is **no possibility of a network partition** (for example, a single-machine system), then yes, you can have consistency and availability.

But once you build a truly distributed system across multiple machines, you must assume that partitions can occur.

That's why distributed systems typically focus on **CP** or **AP** behavior during failures.

---

# 11. Real Examples

### Banking

Suppose your account says

₹1000.

Another ATM says

₹500.

Impossible.

The bank would rather reject your request

than show incorrect balances.

This leans toward

CP.

---

### Instagram

You change your profile picture.

Your friend sees

the old picture

for

2 seconds.

Nobody cares.

Availability matters more.

This leans toward

AP.

---

### Amazon Reviews

A review appears

5 seconds later.

Perfectly acceptable.

---

### Flight Booking

Seat 21A

cannot be sold

to two people.

Consistency matters.

---

# 12. Why Eventual Consistency Exists

Remember replication lag?

Replica

↓

Old Data

↓

Eventually

↓

Updated

That's

**Eventual Consistency.**

Many AP systems rely on this idea.

We'll study it in detail in the next chapter.

---

# 13. Mental Model

Imagine two teachers grading the same exam.

Normally,

they compare answers.

One day,

the phone line breaks.

Now one teacher changes a score.

The other never hears about it.

A student asks

their grade.

Should the second teacher

answer with the old score?

Or say

"I can't verify the latest grade right now"?

That's exactly the CAP dilemma.

---

# 14. Tradeoffs

### CP Systems

Advantages

- Correct data
- No stale reads
- Better for financial systems

Disadvantages

- Some requests may fail during partitions
- Reduced availability

---

### AP Systems

Advantages

- Always responds
- Better user experience
- High availability

Disadvantages

- Temporary stale data
- Eventual synchronization required

---

# 15. Popular Technologies

Examples (simplified)

### CP-oriented

- ZooKeeper
- etcd

---

### AP-oriented

- Cassandra (configurable, but often deployed with eventual consistency)
- Riak

---

Many modern databases allow you to tune consistency levels rather than fitting neatly into one category.

---

# 16. Common Interview Questions

- What is CAP Theorem?
- What is a network partition?
- Why can't distributed systems guarantee all three properties?
- Why is partition tolerance usually considered mandatory?
- CP vs AP?
- Is MySQL a CA system?
- Why is Instagram okay with eventual consistency?
- Why do banks prioritize consistency?

---

# 17. Connections

We now understand

why

distributed databases

behave differently.

The next question becomes

> If I choose availability,

how long can inconsistent data exist?

That leads us directly to

**Consistency Models**.

Strong Consistency.

Eventual Consistency.

Read-after-Write.

Monotonic Reads.

And more.

---

# Key Takeaways

- CAP Theorem applies only to **distributed systems**.
- A **network partition** means machines are alive but cannot communicate.
- During a partition, a system must choose between **consistency** and **availability**.
- Systems requiring correctness (e.g., banking) often favor consistency.
- Systems prioritizing user experience (e.g., social media) often favor availability and accept eventual consistency.
- CAP is about behavior **during network partitions**, not about choosing any two properties under normal operation.

---

## One improvement I'd make to the handbook

I'd add a small note after this chapter titled **"Common Interview Misconceptions"**, because CAP is frequently oversimplified.

For example:

❌ "Every database is either CP or AP."

A better understanding is:

- Many databases let you configure different consistency levels.
- The same database may behave differently depending on how it's deployed and configured.
- CAP describes the tradeoff **during a partition**, not the database's behavior under normal operating conditions.

That nuance is often what separates someone who has memorized CAP from someone who truly understands it.
