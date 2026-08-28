# Chapter 5 — Delivery Guarantees

## Goal

Understand what can happen to a message when failures occur, why distributed systems cannot simply assume that a message was delivered successfully, and the tradeoffs between **at-most-once, at-least-once, and exactly-once delivery**.

---

# 1. The Problem

So far, messaging has looked relatively simple.

A producer sends a message:

```text
Producer
   │
   │ Message
   ▼
Queue / Topic / Stream
   │
   ▼
Consumer
```

The consumer processes it.

```text
Receive
   ↓
Process
   ↓
Done
```

But real distributed systems fail.

Suppose an Order Service publishes:

```text
OrderCreated
```

The message travels toward the messaging system.

Then:

```text
Order Service
      │
      │ OrderCreated
      ▼
   Network ✕
```

What happened?

Did the message reach the messaging system?

Maybe yes.

Maybe no.

The producer may not know.

Or suppose the message reaches the consumer.

```text
Consumer
   │
   ▼
Process Payment
   │
   ▼
✓ Payment processed
   │
   ▼
Send acknowledgment
   │
   ✕ Crash
```

The payment may have been processed successfully.

But the acknowledgment never arrived.

The messaging system may think:

> "The consumer failed. I should deliver the message again."

Now the payment might be processed twice.

So the real question is not simply:

> Did we send the message?

It is:

> **After failures, how many times might the message be delivered and processed?**

This is the problem of **delivery guarantees**.

---

# 2. Why Existing Solutions Fail

Imagine we use the simplest possible approach.

The producer sends a message.

```text
Producer
   │
   ▼
Consumer
```

If there is no response, the producer assumes failure.

```text
Send message
      │
      ▼
No response
      │
      ▼
Try again
```

But what if the first message actually arrived?

```text
Attempt 1 ✓ Delivered
Attempt 2 ✓ Delivered again
```

Now:

```text
Consumer receives:

Message A
Message A
```

We created a duplicate.

So we might decide:

> Never retry.

```text
Send once
   │
   └── If something goes wrong, give up.
```

Now we avoid duplicates.

But:

```text
Network failure
      │
      ▼
Message lost
```

We have traded duplication for message loss.

This is the central tradeoff.

When failures create uncertainty, we usually have to choose between risks.

---

# 3. The Big Idea

There are three major conceptual delivery guarantees.

```text
At-most-once
At-least-once
Exactly-once
```

They answer the question:

> What can happen to a message during delivery and processing?

Conceptually:

```text
At-most-once
────────────
0 or 1 times

At-least-once
─────────────
1 or more times

Exactly-once
────────────
Exactly 1 time
```

Each guarantee makes a different tradeoff between:

- Message loss.
- Duplicate processing.
- Complexity.
- Performance.
- Cost.

---

# 4. At-Most-Once Delivery

The simplest approach is:

> Deliver the message once and do not retry.

```text
Producer
   │
   │ Message
   ▼
Messaging System
```

If something fails:

```text
Message lost
```

No retry.

The message may therefore be processed:

```text
0 times
```

or:

```text
1 time
```

But never intentionally delivered again.

Hence:

> **At-most-once.**

---

## Example

Suppose a system sends a metric:

```text
CPU Usage = 72%
```

The message is lost.

Five seconds later:

```text
CPU Usage = 74%
```

A new metric arrives.

The system may not care about the missing measurement.

So:

```text
Some data loss
     ↓
Acceptable
```

In this situation, avoiding duplicate processing may be more important than guaranteeing every single event.

---

## Architecture

```text
Producer
   │
   │ Send once
   ▼
Queue / Consumer
   │
   ├── Success ✓
   │
   └── Failure ✕
          │
          ▼
       Message lost
```

---

# 5. At-Least-Once Delivery

Now imagine we care much more about losing messages.

For example:

```text
PaymentCompleted
```

We do not want this event to disappear because of a temporary network failure.

So we introduce retries.

```text
Send message
      │
      ├── Confirmed ✓
      │
      └── No confirmation
              │
              ▼
            Retry
```

Now the message should eventually be delivered as long as the system recovers and keeps retrying under its configured policy.

But there is a problem.

Suppose the first attempt actually succeeded.

```text
Attempt 1

Producer ───► Messaging System ✓
```

But the confirmation is lost.

```text
Messaging System
      │
      │ ACK
      ▼
Network ✕
```

The producer thinks:

```text
"I don't know if it arrived."
```

So it retries.

```text
Attempt 2

Producer ───► Messaging System
```

Now the same message may exist twice.

```text
Message A
Message A
```

Therefore:

> **At-least-once delivery reduces the risk of loss by accepting the possibility of duplicates.**

The message should be processed:

```text
1 or more times
```

---

# 6. The Fundamental Distributed Systems Problem

Why can't the producer simply know what happened?

Imagine:

```text
Producer ──────► Messaging System
```

The producer waits.

Then:

```text
No response
```

What does that mean?

There are multiple possibilities.

### Case 1

The message never arrived.

```text
Producer ──X──► Messaging System
```

### Case 2

The message arrived, but the acknowledgment was lost.

```text
Producer ───► Messaging System ✓
                  │
                  │ ACK
                  ▼
                 ✕
```

### Case 3

The messaging system processed the message, then crashed before responding.

```text
Message stored ✓
       │
       ▼
Messaging System crashes ✕
```

From the producer's perspective:

```text
No response
```

The producer cannot always distinguish these situations.

This is one of the fundamental realities of distributed systems.

> **A timeout tells you that you don't know what happened. It does not tell you that the operation definitely failed.**

That single idea explains why retries can create duplicates.

---

# 7. Consumer-Side Delivery

The same problem happens when delivering messages to consumers.

Imagine:

```text
Queue
   │
   │ Message A
   ▼
Consumer
```

The consumer processes it successfully.

```text
Message A
    │
    ▼
Update database ✓
```

Then it sends:

```text
ACK
```

But crashes before the acknowledgment reaches the queue.

```text
Consumer
   │
   │ ACK
   ✕ Crash
```

The queue sees:

```text
No ACK
```

What should it do?

If it assumes success:

```text
Message removed
```

But perhaps the consumer crashed before processing it.

Now we could lose the message.

If it assumes failure:

```text
Deliver again
```

But perhaps processing already completed.

Now we can create a duplicate.

Again:

```text
Avoid loss
   ↓
Accept duplicates

Avoid duplicates
   ↓
Risk loss
```

This is why at-least-once delivery is so common.

---

# 8. Exactly-Once Delivery

At first, the obvious answer seems to be:

> Why not simply guarantee exactly once?

Conceptually:

```text
Message
   │
   ▼
Consumer
   │
   ▼
Processed exactly once
```

No loss.

No duplicates.

Perfect.

Unfortunately, distributed systems make this much harder than it sounds.

Consider:

```text
Consumer receives message
        │
        ▼
Updates database ✓
        │
        ▼
Crashes before ACK ✕
```

The messaging system does not know whether the database update happened.

If it retries:

```text
Duplicate database update
```

If it does not retry:

```text
Potential lost processing
```

To truly guarantee exactly-once processing across multiple systems, we need coordination between:

```text
Messaging System
       │
       ▼
Consumer
       │
       ▼
Database
```

They somehow need to agree:

> Did this message produce its effect exactly once?

That coordination can be expensive and complicated.

So we need an important clarification.

---

# 9. Delivery Exactly Once vs Effect Exactly Once

These are not always the same thing.

Suppose a consumer receives a message exactly once.

```text
Message
   │
   ▼
Consumer
```

The consumer might still internally perform an action twice due to a bug.

Conversely, a consumer might receive a message twice:

```text
Message A
Message A
```

but produce the correct effect only once.

For example:

```text
Update Order #123 status → SHIPPED
```

First attempt:

```text
Order #123 = SHIPPED
```

Second attempt:

```text
Order #123 = SHIPPED
```

The final result is still correct.

This gives us an important insight:

> **In practice, it is often easier to tolerate duplicate delivery and design processing so the effect happens only once.**

This is where **idempotency** becomes important.

And that is the next chapter.

But before we get there, let's understand the three guarantees clearly.

---

# 10. Comparing the Guarantees

## At-Most-Once

```text
Message
   │
   ├── Delivered once ✓
   │
   └── Lost ✕
```

Possible processing count:

```text
0 or 1
```

Tradeoff:

> Avoid duplicates, accept possible loss.

---

## At-Least-Once

```text
Message
   │
   ▼
Try delivery
   │
   ├── Confirmed ✓
   │
   └── Uncertain
         │
         ▼
       Retry
```

Possible processing count:

```text
1 or more
```

Tradeoff:

> Avoid loss, accept possible duplicates.

---

## Exactly-Once

Conceptually:

```text
Message
   │
   ▼
Exactly one successful effect
```

Desired count:

```text
Exactly 1
```

Tradeoff:

> Strong guarantee, often requiring significant coordination or carefully designed semantics.

---

# 11. A Visual Comparison

```text
AT-MOST-ONCE

Message
   │
   ▼
Attempt once
   │
   ├── Success ✓
   │
   └── Failure → Lost
```

```text
AT-LEAST-ONCE

Message
   │
   ▼
Attempt
   │
   ├── Confirmed ✓
   │
   └── Uncertain
         │
         ▼
       Retry
         │
         ▼
    Possible duplicate
```

```text
EXACTLY-ONCE

Message
   │
   ▼
Process
   │
   ▼
System must ensure
one logical effect
```

The third one is significantly harder when multiple independent systems are involved.

---

# 12. Real-World Examples

## Analytics

Suppose:

```text
User viewed product
```

One event is lost.

The analytics dashboard says:

```text
999,999 views
```

instead of:

```text
1,000,000 views
```

Depending on the use case, this may be acceptable.

At-most-once may be sufficient.

---

## Email Notifications

Suppose:

```text
Send promotional email
```

What is worse?

```text
Customer misses email
```

or:

```text
Customer receives it twice
```

The answer depends on the product.

Often, duplicates are undesirable but manageable.

A system might use at-least-once delivery with deduplication.

---

## Payments

Suppose:

```text
Charge customer $500
```

Duplicates are dangerous.

But losing the payment request may also be unacceptable.

This often requires:

```text
Reliable delivery
       +
Idempotent processing
       +
Deduplication
```

The goal is not necessarily magical "exactly-once delivery" everywhere.

The goal is:

> **The customer should not be charged twice.**

That is an effect-level business guarantee.

---

# 13. Why "Exactly Once" Is Often Misunderstood

Imagine someone says:

> "Our messaging system guarantees exactly-once."

The first question should be:

> Exactly once where?

For example:

```text
Producer
   │
   ▼
Messaging System
```

Maybe the message is written exactly once to the messaging system.

But what about:

```text
Messaging System
   │
   ▼
Consumer
```

And then:

```text
Consumer
   │
   ▼
Database
```

And then:

```text
Consumer
   │
   ▼
External Payment Provider
```

The guarantee may not automatically extend across every system.

Consider:

```text
Consumer
   │
   ├── Database update ✓
   │
   └── External API call ✓
          │
          ▼
       Crash before checkpoint ✕
```

If the consumer retries, the database and API may behave differently.

Therefore:

> **Always ask what boundary the guarantee applies to.**

Exactly-once within one controlled system is a different problem from exactly-once across databases, queues, external APIs, and third-party services.

---

# 14. Message Identity

If duplicates are possible, how can we recognize them?

Suppose every message has an ID.

```text
Message {
    id: "abc-123",
    type: "PaymentCompleted"
}
```

The consumer receives:

```text
abc-123
```

It processes it.

Later:

```text
abc-123
```

again.

The consumer can ask:

> Have I already processed this message?

Conceptually:

```text
Received Message ID
       │
       ▼
Already processed?
       │
   ┌───┴────┐
   │        │
  Yes       No
   │        │
Ignore   Process
```

This is one approach to handling duplicates.

But this also creates questions.

Where do we store processed IDs?

For how long?

What happens if the deduplication store fails?

This is why reliability always creates more engineering decisions.

---

# 15. Retries Are Not Free

Suppose a consumer temporarily fails.

We retry immediately.

```text
Failure
   │
   ▼
Retry immediately
   │
   ▼
Failure
   │
   ▼
Retry immediately
   │
   ▼
Failure
```

Thousands of consumers may do the same thing.

Now the already failing service receives even more traffic.

```text
Service overloaded
        │
        ▼
Clients retry
        │
        ▼
More overload
        │
        ▼
More failures
```

This can create a retry storm.

So reliability requires us to think not just:

> Should we retry?

but:

> **When should we retry, how often, and when should we stop?**

This will connect to:

- Retry.
- Exponential backoff.
- Dead-letter queues.

For now, the important lesson is:

> Retries improve reliability, but they can also amplify failures.

---

# 16. Where Each Guarantee Helps

## At-Most-Once

Useful when occasional loss is acceptable.

Examples:

- Metrics.
- Telemetry.
- Some monitoring data.
- High-frequency state updates where newer data quickly replaces older data.

---

## At-Least-Once

Useful when losing work is unacceptable but duplicate processing can be handled.

Examples:

- Order events.
- Background jobs.
- Notifications.
- Data pipelines.
- Many business workflows.

This is often the most practical model.

---

## Exactly-Once Semantics

Useful when duplicate effects would be unacceptable.

Examples might include:

- Financial operations.
- Inventory changes.
- Critical state transitions.

But the engineering solution may actually be:

```text
At-least-once delivery
          +
Idempotent processing
          =
Effectively once business effect
```

rather than one magical end-to-end delivery mechanism.

---

# 17. Where Stronger Guarantees Don't Help

A stronger guarantee is not automatically better.

Suppose you are recording:

```text
Mouse moved to X=438, Y=291
```

millions of times per second.

Building expensive exactly-once coordination may be absurd.

If one event disappears:

```text
Mouse moved to X=437, Y=290
Mouse moved to X=439, Y=292
```

The system still understands what happened.

Engineering is about matching guarantees to requirements.

The question is not:

> What is the strongest possible guarantee?

It is:

> **What is the weakest guarantee that still keeps the business correct?**

That often produces simpler, faster, and more reliable systems.

---

# 18. Mental Model

## Registered Mail

Imagine sending a letter.

### At-Most-Once

You send it once.

```text
Send
  │
  └── If lost, it's gone.
```

No duplicate letters.

But it might never arrive.

---

### At-Least-Once

You keep sending until you receive confirmation.

```text
Send
  │
  ▼
No confirmation
  │
  ▼
Send again
```

Now the recipient may receive two copies.

---

### Exactly-Once

You want the recipient to receive the information exactly once.

But imagine:

```text
Recipient receives letter ✓
      │
      ▼
Signature confirmation gets lost ✕
```

You cannot easily know whether to send another copy.

The problem is not just sending the letter.

The problem is **knowing what happened when communication itself can fail**.

That is the core challenge behind delivery guarantees.

---

# 19. Tradeoffs

## At-Most-Once

### Advantages

- Simple.
- Low overhead.
- No retry-driven duplicates.
- Often lower latency.

### Disadvantages

- Messages can be lost.

---

## At-Least-Once

### Advantages

- Strong protection against message loss.
- Handles temporary failures well.
- Often practical to implement.

### Disadvantages

- Duplicate delivery is possible.
- Consumers must be designed to handle duplicates.
- Retries increase system load.

---

## Exactly-Once

### Advantages

- Simplifies certain business correctness requirements when the guarantee genuinely covers the required boundary.
- Reduces duplicate effects.

### Disadvantages

- Difficult across distributed systems.
- Often requires coordination.
- Can reduce throughput or increase latency.
- Guarantees may apply only within specific boundaries.
- Frequently misunderstood or overstated.

---

# 20. Common Interview Questions

## What is the difference between at-most-once and at-least-once delivery?

```text
At-most-once
────────────
Message may be lost.
Duplicates are avoided.

At-least-once
─────────────
Message should eventually arrive.
Duplicates are possible.
```

The difference comes from retries.

---

## Why can retries create duplicates?

Because failure may create uncertainty.

```text
Send message
     │
     ▼
No response
```

The sender cannot always know whether:

```text
Message was lost
```

or:

```text
Message succeeded but confirmation was lost
```

Retrying handles the first case but creates duplicates in the second.

---

## Is exactly-once delivery always possible?

It depends heavily on the system boundary and what "exactly once" means.

Guaranteeing exactly-once effects across independent databases, services, and external APIs is much harder than guaranteeing controlled processing within one coordinated system.

The correct interview answer is not simply:

> Yes.

or:

> No.

It is:

> **Exactly once at which boundary, and exactly once delivery or exactly once effect?**

---

## Why is at-least-once so common?

Because many systems prefer:

```text
Possible duplicate
```

over:

```text
Permanent data loss
```

Duplicates can often be handled through:

- Idempotency.
- Deduplication.
- Unique constraints.
- State-aware processing.

---

## How do you prevent duplicate processing?

Typical approaches include:

```text
Message ID
    +
Deduplication Store
```

or designing operations to be idempotent.

For example:

```text
Set Order Status = SHIPPED
```

can safely be repeated.

Whereas:

```text
Add $100 to account balance
```

is not naturally idempotent.

That distinction takes us directly to the next chapter.

---

# 21. Before vs After Architecture

## Naive Delivery

```text
Producer
   │
   │ Message
   ▼
Consumer
   │
   ▼
Process
```

Failure:

```text
Producer ───► Consumer
                  │
                  ✕
```

We don't know what happened.

---

## Retry-Based Delivery

```text
Producer
   │
   ▼
Send
   │
   ├── Confirmed ✓
   │
   └── Uncertain
         │
         ▼
       Retry
         │
         ▼
Possible duplicate
```

We reduce the chance of message loss.

But duplicates become part of the system design.

---

# 22. Connections

Our story is becoming increasingly realistic.

We began with:

```text
Producer
   │
   ▼
Consumer
```

Then introduced asynchronous communication.

Then:

```text
Queue
```

for buffering work.

Then:

```text
Pub/Sub
```

for fan-out.

Then:

```text
Event Streams
```

for durable history and replay.

Now we have discovered the uncomfortable reality underneath all of them.

Failures create uncertainty.

```text
Message sent
     │
     ▼
Something failed
     │
     ▼
Did it arrive?
```

Often, we cannot know for certain.

So we choose a delivery model.

```text
At-most-once
     │
     ▼
Possible loss

At-least-once
     │
     ▼
Possible duplicates

Exactly-once
     │
     ▼
Stronger coordination and carefully defined boundaries
```

At-least-once delivery is frequently the practical choice.

But that means every consumer must be prepared for this:

```text
Message #123
Message #123
```

The same message may arrive again.

So our next question becomes:

> How can we design an operation so that performing it multiple times produces the same correct result as performing it once?

That concept is:

# Chapter 6 — Idempotency
