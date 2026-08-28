# Chapter 6 — Idempotency

## Goal

Understand how distributed systems remain correct when the same request or message is processed more than once, and why **idempotency is one of the most important tools for handling retries and at-least-once delivery**.

---

# 1. The Problem

In the previous chapter, we learned that failures create uncertainty.

Suppose a producer sends a message:

```text
ProcessPayment
```

The consumer processes it:

```text
Payment processed ✓
```

But before the consumer can confirm completion:

```text
Consumer crashes ✕
```

The messaging system sees:

```text
No acknowledgment
```

So, to avoid losing the message, it delivers it again.

```text
ProcessPayment
        ↓
ProcessPayment
```

Now imagine the consumer simply executes the operation every time.

```text
First message
      ↓
Charge $100 ✓

Duplicate message
      ↓
Charge $100 again ✕
```

The customer has now been charged twice.

The messaging system behaved according to an at-least-once guarantee.

It did not want to lose the message.

But the application was not prepared for duplicates.

This is where we need a new property.

We want:

```text
Do operation once
        ↓
Correct result

Do the same operation again
        ↓
Still the same correct result
```

That property is called **idempotency**.

---

# 2. Why Existing Solutions Fail

Our first instinct might be:

> Let's guarantee that every message is delivered exactly once.

But as we learned in the previous chapter, this is difficult across distributed systems.

Consider:

```text
Messaging System
       │
       ▼
Consumer
       │
       ▼
Database
```

The consumer might:

1. Update the database.
2. Crash before acknowledging the message.

```text
Update database ✓
       │
       ▼
Crash ✕
```

The messaging system retries.

The consumer sees the same message again.

Trying to eliminate every duplicate at the messaging layer can require increasingly complicated coordination.

Instead, we can ask a different question:

> What if duplicate messages were safe?

Instead of trying to guarantee:

```text
Message arrives exactly once
```

we design the consumer so that:

```text
Message arrives one or many times
            ↓
Correct business result
```

That is often much more practical.

---

# 3. The Big Idea

An operation is **idempotent** if performing it multiple times produces the same final result as performing it once.

Conceptually:

```text
Operation once

State A
   │
   ▼
State B
```

Now perform the same operation again:

```text
State B
   │
   ▼
State B
```

Nothing changes.

So:

```text
Operation
     │
     ├── Once
     ├── Twice
     ├── Ten times
     └── Hundred times

            ↓

      Same final result
```

The important word is:

> **Final result.**

Idempotency does not necessarily mean that the system literally does no work on repeated attempts.

It means repeated attempts do not produce an incorrect additional effect.

---

# 4. A Simple Example

Consider updating a user's profile.

```text
Set user's city = Delhi
```

First execution:

```text
City = Mumbai

Set city = Delhi

City = Delhi
```

Second execution:

```text
City = Delhi

Set city = Delhi

City = Delhi
```

Third execution:

```text
City = Delhi

Set city = Delhi

City = Delhi
```

The final state remains:

```text
Delhi
```

This operation is naturally idempotent.

---

Now compare it with:

```text
Increase account balance by $100
```

First execution:

```text
Balance = $500
        +
      $100
        ↓
Balance = $600
```

Second execution:

```text
Balance = $600
        +
      $100
        ↓
Balance = $700
```

The result changes every time.

This operation is **not naturally idempotent**.

---

# 5. Idempotent vs Non-Idempotent Operations

A useful comparison:

## Idempotent

```text
Set status = SHIPPED
```

Repeated execution:

```text
SHIPPED
   ↓
SHIPPED
   ↓
SHIPPED
```

Still correct.

---

## Non-Idempotent

```text
Add $100
```

Repeated execution:

```text
$100
  ↓
$200
  ↓
$300
```

Every execution creates another effect.

---

More examples:

| Operation                   | Naturally Idempotent?             |
| --------------------------- | --------------------------------- |
| Set user name to "John"     | Usually yes                       |
| Set order status to SHIPPED | Usually yes                       |
| Delete file                 | Often yes, depending on semantics |
| Create a new user           | Usually no                        |
| Send email                  | Usually no                        |
| Charge credit card          | No                                |
| Add loyalty points          | No                                |
| Increment view counter      | No                                |
| Replace configuration       | Usually yes                       |

The important phrase is **usually**.

The actual answer depends on business semantics.

For example:

```text
Delete user
```

may be technically safe to repeat.

But if the second request should produce an error because the user no longer exists, then the API semantics need to define what repeated execution means.

Idempotency is not just a database property.

It is a **business operation property**.

---

# 6. The Most Important Distinction: Same Request vs Same Intent

Imagine a user clicks:

```text
Pay $100
```

The browser sends:

```text
POST /payments
```

The network times out.

The user sees:

```text
Something went wrong.
```

They click again.

Now we have:

```text
Request 1 ─────► ?
Request 2 ─────► ?
```

The server receives two requests.

Are they:

```text
Two different payments?
```

Or:

```text
The same payment retried?
```

The server cannot always know from the request alone.

Both might look like:

```text
POST /payments

{
    amount: 100
}
```

This is the central problem.

The server needs a way to understand:

> Are these two requests representing the same logical operation?

That leads us to the concept of an **idempotency key**.

---

# 7. Idempotency Keys

Suppose the client generates a unique key for the logical operation.

```text
Payment Request

Idempotency-Key: abc-123
Amount: $100
```

The server processes the request.

```text
abc-123
    │
    ▼
Has this operation already been processed?
```

If no:

```text
No
 │
 ▼
Process payment
 │
 ▼
Store result for abc-123
```

If the same request arrives again:

```text
abc-123
    │
    ▼
Already processed?
    │
    ▼
Yes
    │
    ▼
Return previous result
```

Conceptually:

```text
Client
   │
   │ Payment, Key = abc-123
   ▼
Server
   │
   ├── First time?
   │       │
   │       ▼
   │    Process
   │       │
   │       ▼
   │    Store result
   │
   └── Duplicate?
           │
           ▼
      Return existing result
```

Now retries become safer.

---

# 8. Idempotency Is About the Logical Operation

This is crucial.

Suppose a user genuinely wants to make two separate payments.

```text
Payment 1 → $100
Payment 2 → $100
```

They should not use:

```text
Same idempotency key
```

Otherwise, the system may think the second payment is merely a retry.

Instead:

```text
Payment 1
Key: abc-123

Payment 2
Key: xyz-789
```

Different logical operations need different identities.

A retry of the same operation keeps the same key.

```text
Original request
Key: abc-123

Retry
Key: abc-123

Another retry
Key: abc-123
```

This allows the system to distinguish:

```text
Same operation retried
```

from:

```text
New operation
```

---

# 9. Idempotency in Message Consumers

The same principle applies to asynchronous systems.

Suppose a consumer receives:

```text
PaymentCompleted

Event ID: event-456
```

The consumer processes it.

```text
event-456
    │
    ▼
Update order status
```

Later, due to at-least-once delivery:

```text
event-456
```

arrives again.

The consumer can maintain a record:

```text
Processed Events

event-123
event-234
event-456
event-789
```

When a message arrives:

```text
Event ID
   │
   ▼
Already processed?
   │
 ┌─┴──┐
 │    │
Yes   No
 │     │
Skip  Process
```

This is called **deduplication**.

It is one way to implement idempotent processing.

---

# 10. Idempotency vs Deduplication

These concepts are related but not identical.

## Deduplication

The system detects:

```text
I've already seen this message.
```

So it skips processing.

```text
Message ID = 123
       │
       ▼
Already processed?
       │
      Yes
       │
       ▼
Skip
```

---

## Idempotency

The operation itself is safe to repeat.

```text
Set status = SHIPPED
```

Run it once:

```text
SHIPPED
```

Run it again:

```text
SHIPPED
```

The operation still produces the correct result.

So:

```text
Deduplication
      ↓
Try not to process duplicates

Idempotency
      ↓
Duplicates are safe if processed
```

Both are useful.

In practice, systems may combine them.

---

# 11. Database Constraints as an Idempotency Tool

Suppose a service creates an order.

The request contains:

```text
Order Request ID: order-request-123
```

The database can enforce uniqueness.

```text
Orders

request_id
──────────
UNIQUE
```

First request:

```text
order-request-123
        ↓
INSERT ✓
```

Duplicate:

```text
order-request-123
        ↓
INSERT ✕ Already exists
```

The application can then return the existing order.

Conceptually:

```text
Request
   │
   ▼
Insert using unique key
   │
 ┌─┴──────────────┐
 │                 │
Success       Already exists
 │                 │
 ▼                 ▼
Create          Return existing
```

The database itself helps protect against duplicate effects.

This is often more reliable than:

```text
Check first
     ↓
Then insert
```

Why?

Because two requests might arrive at nearly the same time.

---

# 12. The Race Condition Problem

Imagine two duplicate requests arrive simultaneously.

```text
Request A ──┐
            ├──► Server
Request B ──┘
```

A naive implementation:

```text
1. Check if key exists.
2. If not, process request.
```

Both requests might do this:

```text
Request A → Key doesn't exist ✓
Request B → Key doesn't exist ✓
```

Then both process.

```text
Payment A ✓
Payment B ✓
```

We have duplicated the effect.

This is why the critical decision often needs to happen atomically.

Conceptually:

```text
Check + Reserve key
```

must behave as one indivisible operation.

For example:

```text
Try to create:

Idempotency-Key = abc-123
```

Only one request succeeds.

```text
Request A → Success
Request B → Already exists
```

Now the second request knows:

> Someone is already handling this operation.

---

# 13. The In-Progress Problem

Now consider another scenario.

Request A arrives.

```text
Key = abc-123
```

The server reserves it.

```text
abc-123 → PROCESSING
```

Before completing:

```text
Server crashes ✕
```

Now Request B retries.

```text
abc-123
```

The system finds:

```text
Status = PROCESSING
```

What should it do?

There is no universally correct answer.

Possible approaches include:

```text
Wait
```

or:

```text
Return "still processing"
```

or:

```text
Use a timeout and recover the operation
```

or:

```text
Allow another worker to continue processing
```

This shows that idempotency is not simply:

```text
Store a request ID.
```

You must also design the lifecycle.

For example:

```text
NEW
 │
 ▼
PROCESSING
 │
 ├── SUCCESS
 │
 └── FAILED
```

And decide what happens when the system crashes in every state.

---

# 14. Idempotency and Side Effects

Now suppose processing a request involves multiple effects.

```text
Create order
     │
     ▼
Charge payment
     │
     ▼
Send confirmation email
```

The server crashes here:

```text
Create order ✓
     │
     ▼
Charge payment ✓
     │
     ▼
Crash ✕
```

A retry occurs.

Now the system must not simply start blindly from the beginning.

Otherwise:

```text
Create duplicate order
Charge twice
```

A robust design needs to understand:

```text
What has already happened?
```

This is why idempotency becomes increasingly important as workflows cross multiple systems.

A simple operation:

```text
Set status = SHIPPED
```

is easy.

A workflow:

```text
Database
    +
Payment provider
    +
Inventory service
    +
Email provider
```

is much harder.

This naturally connects to concepts we will later study, such as:

- Saga.
- Outbox Pattern.
- Distributed transactions.

---

# 15. HTTP Methods and Idempotency

HTTP itself provides a useful conceptual example.

Consider:

```text
GET /users/123
```

You can usually perform it repeatedly.

```text
GET
GET
GET
```

The intended server state does not change.

So it is generally considered idempotent.

---

Now:

```text
PUT /users/123

{
    "name": "Alice"
}
```

Repeated execution:

```text
Set name = Alice
Set name = Alice
Set name = Alice
```

Final result:

```text
Alice
```

Conceptually idempotent.

---

Now:

```text
POST /payments
```

Repeated execution may create:

```text
Payment #1
Payment #2
Payment #3
```

So POST is not inherently idempotent.

But an API can **make a POST operation idempotent** by introducing an idempotency key.

This is an important distinction:

> HTTP method semantics and application-level idempotency are related, but they are not the same thing.

---

# 16. Real-World Usage

## Payments

A customer clicks:

```text
Pay ₹5,000
```

Network failure occurs.

The client retries.

```text
Attempt 1
Key: payment-abc

Attempt 2
Key: payment-abc
```

The payment service should understand:

```text
Same logical payment
```

not:

```text
Two independent payments
```

---

## Order Creation

A mobile application loses connectivity.

```text
Create Order
      │
      ▼
Timeout
      │
      ▼
Retry
```

The server receives two requests.

An idempotency key or unique request identifier can ensure:

```text
One logical order
```

rather than:

```text
Order #101
Order #102
```

---

## Event Processing

An inventory consumer receives:

```text
OrderCancelled
Event ID: 123
```

The message is delivered twice.

The consumer must avoid:

```text
Restore stock +1
Restore stock +1
```

It can either deduplicate:

```text
Event 123 already processed
```

or design the state transition so repeating it is harmless.

---

# 17. Where Idempotency Helps

Idempotency is particularly useful when:

### Retries Exist

```text
Failure
   ↓
Retry
```

Retries create the possibility of duplicate execution.

---

### At-Least-Once Delivery Exists

```text
Message
   ↓
Process
   ↓
Uncertain acknowledgment
   ↓
Deliver again
```

Duplicates are part of the expected system behavior.

---

### Network Failures Create Uncertainty

```text
Request sent
      ↓
Timeout
      ↓
Did it succeed?
```

Idempotency allows safe retry.

---

### Operations Have Expensive or Dangerous Side Effects

Examples:

```text
Charge money
Create order
Reserve inventory
Send reward
```

These operations should not blindly execute multiple times.

---

# 18. Where Idempotency Doesn't Magically Solve the Problem

Idempotency is powerful, but it is not magic.

## External Systems

Suppose:

```text
Your Service
     │
     ▼
External Payment Provider
```

Your service may record:

```text
Payment = PROCESSING
```

Then call the provider.

The provider charges the customer.

```text
Charge ✓
```

Before your service records success:

```text
Your Service crashes ✕
```

Now you still need a strategy to determine what happened.

Idempotency must often extend across system boundaries.

For example:

```text
Your idempotency key
        │
        ▼
External provider also supports the same logical request identity
```

Otherwise, retrying the external call may still create duplicate effects.

---

## Storage Has Limits

If you store every processed key forever:

```text
Processed IDs

1
2
3
4
...
10 billion
```

storage keeps growing.

So you need retention.

```text
Keep idempotency records for:

1 hour?
24 hours?
7 days?
30 days?
```

This depends on how long retries can realistically occur.

---

## Different Requests Can Represent the Same Intent

Two requests may accidentally have different IDs but represent the same business action.

```text
Request A
Key = abc

Request B
Key = xyz
```

Technically different.

But perhaps both are:

```text
"Charge this exact invoice."
```

Idempotency keys alone may not solve every business-level duplication problem.

Sometimes business constraints are also required.

---

# 19. Mental Model

Imagine an elevator button.

You press:

```text
Floor 10
```

Once:

```text
Elevator goes to Floor 10
```

Press it five more times:

```text
Floor 10
Floor 10
Floor 10
Floor 10
Floor 10
```

The elevator should still go to Floor 10 once.

The repeated action does not create:

```text
Go to Floor 10
Go to Floor 10
Go to Floor 10
```

That is idempotency.

The request means:

> Make the state become this.

Once the state has already reached that condition, repeating the request should not create additional effects.

Compare that with pressing a button that means:

```text
Add ₹100 to my account.
```

Pressing it five times should normally add ₹500.

That operation is fundamentally different.

---

# 20. Tradeoffs

## Advantages

### Safe Retries

Clients and consumers can retry after uncertain failures.

### Works Well With At-Least-Once Delivery

Duplicate messages no longer necessarily produce duplicate business effects.

### Improved Reliability

Systems can prefer retrying over silently losing operations.

### Clear Business Semantics

A logical operation can have a stable identity.

### Simpler Recovery

Crashes become easier to recover from when repeated execution is safe.

---

## Disadvantages

### Additional Storage

Processed message IDs or idempotency records may need to be stored.

### Lifecycle Complexity

The system must handle:

```text
PROCESSING
SUCCESS
FAILED
```

and crashes between transitions.

### Concurrency Problems

Multiple copies of the same request may arrive simultaneously.

### Retention Decisions

How long should idempotency records remain?

### Cross-System Complexity

Guaranteeing idempotent effects across databases and external services is harder.

---

# 21. Common Interview Questions

## What is idempotency?

An operation is idempotent if executing it multiple times produces the same correct final effect as executing it once.

---

## Why is idempotency important in distributed systems?

Because failures create uncertainty.

When a request times out:

```text
Did it fail?
```

or:

```text
Did it succeed but the response get lost?
```

We often cannot know.

Retrying is safer if the operation is idempotent.

---

## How do you make an API idempotent?

A common approach is:

```text
Client generates idempotency key
          │
          ▼
Server atomically records key
          │
          ▼
Process operation
          │
          ▼
Store result
```

Repeated requests with the same key return or reproduce the original logical result rather than creating another effect.

---

## How do you make a message consumer idempotent?

Common approaches:

```text
Message ID
    +
Processed-message store
```

or:

```text
Database unique constraint
```

or:

```text
Naturally idempotent state transition
```

Often, systems combine these approaches.

---

## Is deduplication the same as idempotency?

No.

```text
Deduplication
→ Detect duplicate and skip it.

Idempotency
→ Repeated processing is safe.
```

Deduplication can help implement idempotency, but the concepts are different.

---

## Why not just use exactly-once delivery?

Because exactly-once guarantees can be difficult across distributed boundaries.

A more practical design is often:

```text
At-least-once delivery
        +
Idempotent consumer
        ↓
Effectively-once business result
```

---

# 22. Before vs After Architecture

## Before: Retry Can Create Duplicate Effects

```text
Client
   │
   │ Payment Request
   ▼
Payment Service
   │
   ▼
Charge Customer ✓
   │
   ✕ Response lost

Client retries
   │
   ▼
Payment Service
   │
   ▼
Charge Customer again ✕
```

---

## After: Idempotent Operation

```text
Client
   │
   │ Payment
   │ Key = abc-123
   ▼
Payment Service
   │
   ▼
Is abc-123 known?
   │
 ┌─┴─────────────┐
 │               │
No              Yes
 │               │
 ▼               ▼
Process       Return previous
 │             result
 ▼
Store result
```

Now:

```text
Retry
  ≠
New payment
```

provided the same logical operation uses the same key.

---

# 23. Connections

Our messaging story has now reached an important point.

We discovered:

```text
Failures
   ↓
Uncertainty
   ↓
Retries
   ↓
Possible duplicates
```

Then we learned:

```text
Idempotency
   ↓
Duplicates become safe
```

But now another problem appears.

Suppose a message fails because the consumer is temporarily unavailable.

```text
Message
   │
   ▼
Consumer ✕
```

We retry.

```text
Retry
```

It fails again.

```text
Retry
```

Again.

If we retry forever and immediately:

```text
Failure
   ↓
Retry
   ↓
Failure
   ↓
Retry
   ↓
Failure
```

we may overload the failing system and create a retry storm.

So the next question becomes:

> **How should a distributed system retry work without making the failure worse?**

That takes us to:

# Chapter 7 — Retry

---

# 24. Key Takeaways

- Distributed systems must assume that requests and messages can be duplicated.
- Idempotency means repeated execution produces the same correct final effect as executing once.
- Some operations are naturally idempotent, while others need explicit protection.
- Idempotency keys give a logical operation a stable identity across retries.
- Message consumers can use event IDs and deduplication records to handle duplicate delivery.
- Database uniqueness constraints can help enforce idempotent behavior.
- Concurrency and in-progress requests must be handled carefully.
- Idempotency is especially important for payments, orders, inventory, and other operations with dangerous side effects.
- A practical pattern is often:

```text
At-least-once delivery
        +
Idempotent processing
        ↓
Correct business effect
```

- Idempotency makes retries safer.

But retries themselves can create new failures if handled badly.

**Next: Chapter 7 — Retry.**
