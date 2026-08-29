# Module 8 — Chapter 11: Event Sourcing

## 1. Goal

**Store the history of state changes as the source of truth, rather than storing only the latest state.**

---

# 2. The Problem

Let's take a simple order.

In a traditional system, we might store:

```text
Order
----------------
id       = 123
status   = SHIPPED
total    = ₹80,000
```

That's enough to answer:

> "What is the order's current state?"

But now imagine someone asks:

> "How did this order become SHIPPED?"

The database only tells us:

```text
status = SHIPPED
```

It doesn't necessarily tell us:

```text
Order created
Payment completed
Inventory reserved
Order packed
Order shipped
```

The history has been compressed into the current state.

---

# 3. A More Interesting Problem

Consider a bank account.

Today:

```text
Balance = ₹40,000
```

But how did we get there?

Perhaps:

```text
Initial deposit       +₹50,000
Purchase              -₹5,000
Salary                +₹30,000
Rent                  -₹20,000
Transfer               -₹15,000
```

Current balance:

```text
₹40,000
```

If we only store:

```text
balance = ₹40,000
```

we lose the detailed history of what happened.

But sometimes that history is extremely valuable.

We may need to answer:

- Why is the balance ₹40,000?
- What transactions happened?
- When did the account change?
- What was the balance yesterday?
- Can we reconstruct the state at a particular point in time?
- Can we audit every change?

This leads to Event Sourcing.

---

# 4. Why Existing Solutions Fail

We've just learned CQRS.

CQRS separates:

```text
Write Model
     |
     v
Read Model
```

But we haven't decided what the write model actually stores.

A traditional write model stores **current state**:

```text
Order
status = SHIPPED
```

Event Sourcing instead asks:

> **What if the source of truth were the sequence of events that changed the state?**

So instead of:

```text id="h0g5fl"
Current State
-------------
status = SHIPPED
```

we store:

```text id="hkgx3k"
Event 1 → OrderCreated
Event 2 → PaymentCompleted
Event 3 → InventoryReserved
Event 4 → OrderShipped
```

The current state is derived from the events.

---

# 5. The Big Idea

> **Event Sourcing stores state changes as an append-only sequence of events, and the current state is derived by replaying those events.**

Traditional model:

```text
Change
  ↓
Update current state
```

Event-sourced model:

```text
Change
  ↓
Append event
  ↓
Event history
  ↓
Replay events
  ↓
Current state
```

That's the entire idea.

---

# 6. Traditional State Storage

Let's say an order goes through:

```text
CREATED
   ↓
PAID
   ↓
SHIPPED
```

A traditional database might eventually contain:

```text
Order
---------
status = SHIPPED
```

The previous states may no longer be directly available.

---

# 7. Event-Sourced Storage

Instead, we store:

```text
Order Events
------------------------
1. OrderCreated
2. PaymentCompleted
3. OrderShipped
```

The events are generally append-only.

We don't replace:

```text
PaymentCompleted
```

with:

```text
OrderShipped
```

We append a new event.

---

# 8. Reconstructing State

Suppose we start with:

```text
Order = {}
```

Apply:

```text
OrderCreated
```

State becomes:

```text
Order = CREATED
```

Apply:

```text
PaymentCompleted
```

State becomes:

```text
Order = PAID
```

Apply:

```text
OrderShipped
```

State becomes:

```text
Order = SHIPPED
```

So:

```text
Events
  |
  +--> OrderCreated
  |
  +--> PaymentCompleted
  |
  +--> OrderShipped
              |
              v
        Current State
          SHIPPED
```

The events are the source of truth.

---

# 9. Events Are Facts

This is an important distinction.

An event represents something that **already happened**.

For example:

```text
OrderCreated
PaymentCompleted
InventoryReserved
OrderShipped
```

These are facts.

Compare that with a command:

```text
CreateOrder
ChargePayment
ReserveInventory
ShipOrder
```

Commands are requests:

> "Please do this."

Events are facts:

> "This happened."

So:

```text
Command
   ↓
perform operation
   ↓
Event
```

---

# 10. Event vs State

Suppose we have:

```text
Balance = ₹40,000
```

That's state.

Events might be:

```text
AccountOpened       +₹0
Deposit             +₹50,000
Purchase            -₹5,000
Salary              +₹30,000
Rent                -₹20,000
Transfer            -₹15,000
```

We can calculate:

```text
0
+ 50,000
- 5,000
+ 30,000
- 20,000
- 15,000
= 40,000
```

So:

```text
Events
  ↓
State
```

rather than:

```text
State
  ↓
discard history
```

---

# 11. Why Append-Only?

Event stores are generally conceptually append-only.

Suppose:

```text
OrderCreated
```

happened.

We don't modify history to say:

```text
OrderCreated → OrderCancelled
```

Instead:

```text
OrderCreated
OrderCancelled
```

The second event explains what happened later.

This gives us an immutable history.

That has enormous value for:

- auditing
- debugging
- temporal analysis
- rebuilding state
- understanding how the system reached its current state

---

# 12. Event Sourcing + CQRS

Now we can connect the previous chapter.

CQRS:

```text
Commands
    ↓
Write Model
    ↓
Read Model
```

Event Sourcing changes the write side:

```text
Commands
    ↓
Write Model
    ↓
Events
    ↓
Read Model / Projections
```

A more complete picture:

```text
                    Commands
                       |
                       v
                  Write Model
                       |
                       v
                  Event Store
                       |
          +------------+------------+
          |            |            |
          v            v            v
      Projection   Projection   Projection
          |            |            |
          v            v            v
       Read DB      Read DB      Read DB
```

This combination is extremely powerful.

But remember:

> **CQRS and Event Sourcing are separate patterns.**

---

# 13. Event Sourcing Without CQRS

You can use Event Sourcing without having a sophisticated separate read architecture.

For example:

```text
Application
    |
    v
Event Store
    |
    v
Current State
```

The application reconstructs the entity state from its events.

You don't necessarily need separate read databases.

---

# 14. CQRS Without Event Sourcing

Similarly:

```text
Write DB
    |
    | changes
    v
Read DB
```

can implement CQRS while the write database stores ordinary current-state data.

So:

```text
CQRS ≠ Event Sourcing
```

But they pair naturally.

---

# 15. A Real Example: Order Lifecycle

Let's build one.

Initial:

```text
No order
```

Customer creates order:

```text
OrderCreated {
    orderId: 123,
    customerId: 42,
    total: 80000
}
```

Then payment succeeds:

```text
PaymentCompleted {
    orderId: 123,
    paymentId: 987
}
```

Then inventory is reserved:

```text
InventoryReserved {
    orderId: 123
}
```

Then shipping happens:

```text
OrderShipped {
    orderId: 123
}
```

The event stream is:

```text
OrderCreated
      ↓
PaymentCompleted
      ↓
InventoryReserved
      ↓
OrderShipped
```

Current state:

```text
Order
---------
status = SHIPPED
paid   = true
reserved = true
```

But unlike a traditional database, we still have the complete history.

---

# 16. Replaying Events

Suppose our read model becomes corrupted.

Traditional approach:

```text
Database corrupted
     ↓
Restore backup
```

With Event Sourcing:

```text
Event Store
    |
    | replay
    v
Projection
    |
    v
Rebuilt Read Model
```

We can reconstruct the current state from the authoritative event history.

This is one of the biggest benefits of Event Sourcing.

---

# 17. Building a New View

Suppose we've been storing events for years.

Originally, we built:

```text
Order History View
```

Later the business asks:

> "Can we create a dashboard showing revenue by month?"

If the necessary events exist, we can create a new projection:

```text
Event Store
    |
    +----→ Order History
    |
    +----→ Revenue Dashboard
    |
    +----→ Customer Analytics
```

We don't necessarily need to modify the original write model.

We can replay historical events into the new projection.

This is extremely powerful.

---

# 18. Time Travel

Because we have the event history, we can potentially reconstruct state at a particular point.

Suppose:

```text
Event 1
Event 2
Event 3
Event 4
Event 5
```

Current state:

```text
after Event 5
```

But we can replay only:

```text
Event 1
Event 2
Event 3
```

to determine:

> "What did the order look like at that point?"

Conceptually:

```text
Event 1 → Event 2 → Event 3 → Event 4 → Event 5
                       ↑
                  state at T3
```

This is extremely useful for auditing and debugging.

---

# 19. The Big Tradeoff: Storage Growth

Here's the obvious downside.

Traditional state storage:

```text
Order
status = SHIPPED
```

One current representation.

Event Sourcing:

```text
OrderCreated
PaymentCompleted
InventoryReserved
OrderShipped
...
```

The history grows continuously.

For a busy system:

```text
millions
   ↓
billions
   ↓
trillions
```

of events may eventually accumulate.

So we need strategies for efficiently reconstructing state.

---

# 20. Snapshots

Suppose an entity has:

```text
100,000 events
```

Do we really want to replay all 100,000 every time we need its current state?

Probably not.

We can periodically create a snapshot:

```text
Events 1...50,000
       ↓
    Snapshot
       ↓
Current State
```

Then:

```text
Snapshot
   +
Events 50,001...50,100
   ↓
Current State
```

So instead of replaying:

```text
100,000 events
```

we replay:

```text
snapshot + 100 events
```

Snapshots are an optimization.

They don't replace the event history.

---

# 21. Eventual Consistency

When Event Sourcing is combined with CQRS:

```text
Command
   ↓
Event Store
   ↓
Event
   ↓
Projection
   ↓
Read Model
```

the read model may lag.

For example:

```text
Event Store:
OrderShipped ✓

Read Model:
Order = PROCESSING
```

for a short period.

Then:

```text
Projection catches up
```

and:

```text
Order = SHIPPED
```

So again, we have eventual consistency.

---

# 22. Events Become a Contract

There's another subtle issue.

Once events are stored and consumed by many systems:

```text
OrderCreated
   |
   +→ Order View
   +→ Analytics
   +→ Notifications
   +→ Billing
```

the event structure becomes important.

Changing:

```text
OrderCreated
```

carelessly can break consumers.

This means event schemas need careful evolution.

For example, if an event originally contains:

```text
{
    orderId,
    total
}
```

and later we want:

```text
{
    orderId,
    total,
    currency
}
```

existing consumers need to continue functioning.

So Event Sourcing introduces a new architectural concern:

> **How do we evolve historical events without breaking the systems that depend on them?**

---

# 23. Events Should Represent Business Meaning

Compare:

```text
UPDATE orders SET status = 'SHIPPED'
```

with:

```text
OrderShipped
```

The second carries business meaning.

Why is that useful?

Because different consumers can interpret it:

```text
OrderShipped
   |
   +→ update customer order view
   |
   +→ send notification
   |
   +→ update analytics
   |
   +→ trigger loyalty points
```

The event is meaningful beyond the database operation that generated it.

---

# 24. Event Sourcing vs Traditional CRUD

Let's compare.

### Traditional

```text
Command
   ↓
Update DB
   ↓
Current State
```

The database primarily stores:

```text
"What is true now?"
```

---

### Event Sourcing

```text
Command
   ↓
Event
   ↓
Event Store
   ↓
Derived State
```

The event store primarily records:

```text
"What happened?"
```

and state is derived from that history.

This is perhaps the cleanest way to remember the difference.

---

# 25. Where It Helps

### 1. Strong audit requirements

Financial systems, compliance-heavy systems, or other domains where knowing exactly what happened matters.

### 2. Complex domain history

When the sequence of changes itself is valuable.

### 3. Rebuilding projections

If you need to create new read models from historical information.

### 4. Debugging

You can inspect the sequence of events that produced a state.

### 5. Temporal queries

Questions such as:

> "What did the state look like at this point in time?"

become much easier conceptually.

### 6. Multiple read models

One event history can produce many projections.

---

# 26. Where It Doesn't Help

### Simple CRUD applications

If your application is:

```text
Create user
Update user
Read user
Delete user
```

Event Sourcing may add huge complexity without much value.

---

### Very high-frequency state changes

If an entity changes constantly:

```text
100,000 changes/sec
```

event storage can become enormous.

It may still be appropriate, but the architecture needs to account for it.

---

### Domains without meaningful history

If nobody cares about:

```text
how we got here
```

storing every state transition may not provide enough value to justify the cost.

---

### Teams unfamiliar with the model

Event Sourcing changes how developers reason about:

- updates
- deletes
- schema evolution
- debugging
- data migrations
- projections

It isn't just swapping one database for another.

---

# 27. What About Deletes?

This is an interesting consequence.

In a traditional database:

```text
DELETE FROM Order
```

In Event Sourcing, deleting the event would destroy history.

Instead, we might record:

```text
OrderDeleted
```

or:

```text
OrderCancelled
```

The event history remains intact.

So:

```text
Delete state
```

often becomes:

```text
Record that the deletion/cancellation happened
```

This preserves the historical truth.

---

# 28. What About Corrections?

Suppose someone entered:

```text
Price = ₹10,000
```

but it should have been:

```text
₹12,000
```

We don't generally rewrite the old event:

```text
PriceSet(10000)
```

Instead, append:

```text
PriceCorrected(
    oldPrice = 10000,
    newPrice = 12000
)
```

Now the history tells us:

```text
10,000
   ↓
correction
   ↓
12,000
```

This is one of the defining properties of event-based systems.

---

# 29. Tradeoffs

## Advantages

### Complete history

You know what happened, not merely the current state.

### Auditability

Changes can be traced to events.

### Rebuildable state

Projections can be recreated from the event history.

### Multiple projections

Different consumers can derive different views.

### Time-based reconstruction

Historical state can be reconstructed by replaying events up to a point.

---

## Disadvantages

### More storage

Every state transition is retained.

### More complexity

You now need:

```text
Event Store
Projections
Replay
Snapshots
Event schema evolution
```

### Eventual consistency

Especially when projections are asynchronous.

### Difficult event evolution

Historical events cannot casually be rewritten.

### Debugging requires a different mindset

You reason about:

```text
sequence of events
```

rather than simply:

```text
current database row
```

---

# 30. Common Interview Questions

## Q1. What is Event Sourcing?

Event Sourcing stores the sequence of state-changing events as the source of truth, with current state derived by replaying those events.

---

## Q2. Event Sourcing vs traditional database?

Traditional:

```text
Store current state.
```

Event Sourcing:

```text
Store changes/events.
Derive current state.
```

---

## Q3. Does Event Sourcing require CQRS?

**No.**

They are separate patterns, although they are commonly combined.

---

## Q4. Why are events append-only?

Because events represent historical facts.

Changing or deleting past events destroys the history from which state and audit information are derived.

---

## Q5. How do you rebuild a projection?

Replay historical events:

```text
Event Store
    ↓
Replay
    ↓
Projection
    ↓
New Read Model
```

---

## Q6. What if there are millions of events?

Use mechanisms such as snapshots so that state reconstruction doesn't require replaying the entire history every time.

---

## Q7. What is the biggest disadvantage?

**Complexity.**

Event Sourcing affects the entire architecture:

- data modeling
- event design
- projections
- schema evolution
- consistency
- storage
- recovery

It shouldn't be introduced merely because "event-driven architecture is modern."

---

# 31. Before vs After Architecture

### Traditional state-based system

```text
                  Command
                     |
                     v
                  Service
                     |
                     v
                 Database
                     |
                     v
               Current State
```

The database contains primarily:

```text
Current State
```

---

### Event-Sourced system

```text
                  Command
                     |
                     v
                Write Model
                     |
                     v
                Event Store
                     |
          +----------+----------+
          |          |          |
          v          v          v
      Projection Projection Projection
          |          |          |
          v          v          v
       Read DB    Search View Analytics
```

The event store contains:

```text
Event 1
Event 2
Event 3
Event 4
...
```

and the current state is derived from them.

---

# 32. The Deeper Connection

Now look at the last three chapters together.

### Saga

```text
How do we coordinate
a business operation across services?
```

### CQRS

```text
How do we separate
write requirements from read requirements?
```

### Event Sourcing

```text
What if the history of state changes
is more valuable than just current state?
```

So:

```text
Saga
  ↓
distributed business workflow

CQRS
  ↓
separate command/query models

Event Sourcing
  ↓
store the history of changes
```

And when CQRS + Event Sourcing are combined:

```text
                 Command
                    |
                    v
               Write Model
                    |
                    v
               Event Store
                    |
                    v
                 Events
                /  |   \
               /   |    \
              v    v     v
           View  Search  Analytics
```

This is a powerful architecture pattern—but also a significantly more complex one.

---

# 33. Connections

We have **one final chapter in Module 8**:

## Chapter 12 — Outbox Pattern

And this one solves a very important problem that naturally emerges from everything we've just learned.

Suppose a service needs to:

```text
1. Update its database
2. Publish an event
```

For example:

```text
Create Order
    |
    +----→ Save to DB
    |
    +----→ Publish OrderCreated
```

What if:

```text
DB write ✓
Event publish ✗
```

Now the database says:

```text
Order exists
```

but other services never receive:

```text
OrderCreated
```

Or the reverse:

```text
Event published ✓
DB write ✗
```

Now consumers believe something happened that the database doesn't contain.

We have a classic **dual-write problem**.

The next chapter asks:

> **How can we make a database update and event publication reliable without requiring a distributed transaction between our database and messaging system?**

That is exactly what the **Outbox Pattern** addresses.

---

# 34. Key Takeaways

1. **Event Sourcing stores state changes as events rather than only storing current state.**
2. The event history becomes the source of truth.
3. Current state is derived by replaying events.
4. Events represent **facts about what happened**; commands represent requests to perform something.
5. Events are generally append-only and should not be casually modified or deleted.
6. Event Sourcing provides powerful auditing, debugging, replay, and historical reconstruction capabilities.
7. **Snapshots** can make state reconstruction efficient when event histories become large.
8. Event Sourcing does **not** require CQRS, although the two are commonly combined.
9. Combining Event Sourcing with CQRS allows one event history to drive multiple specialized read models.
10. Event Sourcing introduces significant complexity around storage, projections, eventual consistency, and event schema evolution.

### One sentence to remember

> **Traditional systems ask "What is the state now?"; Event Sourcing asks "What happened?" and derives the current state from that history.**
