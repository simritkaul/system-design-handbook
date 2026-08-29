# Module 8 — Chapter 8: Saga Pattern

## 1. Goal

**Coordinate a business operation that spans multiple independent services without requiring one giant distributed database transaction.**

---

# 2. The Problem

Let's build an e-commerce system.

A customer places an order:

```text
"Buy iPhone for ₹80,000"
```

That single business operation might involve several services:

```text
              Order
                |
                v
           Payment Service
                |
                v
          Inventory Service
                |
                v
          Shipping Service
```

Each service may own its own data:

```text
Order Service      → Order DB
Payment Service    → Payment DB
Inventory Service  → Inventory DB
Shipping Service   → Shipping DB
```

Now imagine:

```text
Create Order       ✓
Charge Payment     ✓
Reserve Inventory  ✓
Create Shipment    ✗
```

We have a problem.

The customer has been charged.

Inventory has been reserved.

But the shipment wasn't created.

What should happen?

We can't simply say:

```text
ROLLBACK
```

because these are **different services with potentially different databases**.

There is no simple local transaction covering everything.

---

# 3. Why Existing Solutions Fail

Let's connect this to what we've already learned.

### Distributed Lock?

A lock can prevent two processes from modifying something simultaneously.

It doesn't provide:

```text
Order ✓
Payment ✓
Inventory ✓
Shipping ✗
```

and then automatically undo the previous successful operations.

---

### Retry?

We could retry Shipping:

```text
Shipping ✗
   ↓
Retry
   ↓
Shipping ✗
```

That may work for transient failures.

But what if Shipping is permanently unavailable?

Or what if the operation cannot be completed?

We still need to compensate for the work already performed.

---

### Database Transaction?

A normal database transaction gives us:

```text
BEGIN
  operation A
  operation B
  operation C
COMMIT
```

and:

```text
ROLLBACK
```

But now our operations live in separate services:

```text
DB A       DB B       DB C       DB D
 |           |          |          |
Order     Payment    Inventory  Shipping
```

A single traditional transaction doesn't naturally span all of them.

And even if we build a distributed transaction mechanism, it introduces significant coordination and availability costs.

That leads us to the Saga Pattern.

---

# 4. The Big Idea

> **A Saga breaks one distributed business transaction into a sequence of local transactions, with a compensating action for each step that may need to be undone.**

Instead of:

```text
ONE BIG TRANSACTION

Order
Payment
Inventory
Shipping
        ↓
      Commit
```

we do:

```text
Local transaction 1
        ↓
Local transaction 2
        ↓
Local transaction 3
        ↓
Local transaction 4
```

And if something fails:

```text
Failure
  ↓
Compensating actions
  ↓
Undo previously completed business effects
```

The critical idea is:

> **We don't literally roll back the old transaction. We perform a new action that semantically compensates for it.**

---

# 5. The Simple Example

Let's define our order process.

### Step 1 — Create Order

```text
Order Service
    ↓
Order = CREATED
```

### Step 2 — Charge Payment

```text
Payment Service
    ↓
Payment = SUCCESS
```

### Step 3 — Reserve Inventory

```text
Inventory Service
    ↓
Inventory = RESERVED
```

### Step 4 — Create Shipment

```text
Shipping Service
    ↓
Shipment = CREATED
```

Everything succeeds:

```text
Create Order       ✓
Charge Payment     ✓
Reserve Inventory  ✓
Create Shipment    ✓

        ↓

      SUCCESS
```

Great.

---

# 6. What Happens When Something Fails?

Suppose:

```text
Create Order       ✓
Charge Payment     ✓
Reserve Inventory  ✓
Create Shipment    ✗
```

We can't just leave things like this.

The Saga says:

> "Execute compensating actions for the successful steps."

So:

```text
Shipping ✗
   ↓
Compensate Inventory
   ↓
Release reservation
   ↓
Compensate Payment
   ↓
Refund customer
   ↓
Compensate Order
   ↓
Cancel order
```

The final state becomes:

```text
Order       = CANCELLED
Payment     = REFUNDED
Inventory   = RELEASED
Shipping    = NOT CREATED
```

This is **business-level rollback**.

---

# 7. A Crucial Difference: Rollback vs Compensation

This distinction is extremely important.

### Traditional rollback

Suppose:

```text
BEGIN TRANSACTION

UPDATE A
UPDATE B
UPDATE C

ROLLBACK
```

The database restores the previous state.

It's as though the operations never happened.

---

### Saga compensation

Suppose:

```text
Charge ₹80,000
```

You cannot make history disappear.

The payment system may have:

```text
Transaction #123 = ₹80,000 charged
```

To compensate:

```text
Refund ₹80,000
```

Now:

```text
Charge
  ↓
Refund
```

The original transaction still happened.

We performed a **new business operation** that counteracts its effect.

That's why:

> **Saga provides semantic rollback, not physical rollback.**

---

# 8. Compensation Isn't Always Perfectly Symmetric

Suppose we:

```text
Reserve inventory
```

Compensation:

```text
Release inventory
```

That's fairly straightforward.

But imagine:

```text
Send package
```

The compensation might be:

```text
Recall package
```

which isn't necessarily possible.

Or:

```text
Send email
```

What is the rollback?

You can't unsend an email.

So Saga design requires us to ask:

> **Can the business effect actually be compensated?**

Sometimes the answer is no.

In those cases, we may need a different business workflow.

For example:

```text
Email sent
    ↓
Can't undo
    ↓
Send correction email
```

That's not a true rollback.

It's another compensating business action.

---

# 9. Saga as a Sequence

We can represent the workflow like this:

```text
T1 → T2 → T3 → T4
```

where:

```text
T1 = Create Order
T2 = Charge Payment
T3 = Reserve Inventory
T4 = Create Shipment
```

Each transaction has a compensation:

```text
T1 → C1
T2 → C2
T3 → C3
T4 → C4
```

where:

```text
C1 = Cancel Order
C2 = Refund Payment
C3 = Release Inventory
C4 = Cancel Shipment
```

If T4 fails:

```text
T1 → T2 → T3 → T4 ✗
                |
                v
             C3 → C2 → C1
```

We compensate in reverse order.

Why reverse?

Because later steps may depend on earlier steps.

---

# 10. Saga Has Two Major Styles

There are two important ways to coordinate a Saga:

1. **Choreography**
2. **Orchestration**

These are conceptually different.

---

# 11. Choreography

In choreography, there is **no central coordinator**.

Each service reacts to events from other services.

For example:

```text
Order Service
     |
     | OrderCreated
     v
Payment Service
     |
     | PaymentCompleted
     v
Inventory Service
     |
     | InventoryReserved
     v
Shipping Service
```

Each service decides:

> "I received an event. What should I do next?"

---

## Example

Order Service:

```text
OrderCreated
```

Payment Service sees it:

```text
chargePayment()
```

Then publishes:

```text
PaymentCompleted
```

Inventory Service sees that:

```text
reserveInventory()
```

Then publishes:

```text
InventoryReserved
```

Shipping Service sees that:

```text
createShipment()
```

---

# 12. Choreography Failure

Suppose:

```text
OrderCreated
      ↓
PaymentCompleted
      ↓
InventoryReserved
      ↓
ShippingFailed
```

Who tells Payment to refund?

There is no central coordinator.

Shipping might publish:

```text
ShippingFailed
```

Inventory sees it:

```text
releaseInventory()
```

Then perhaps Inventory publishes:

```text
InventoryReleased
```

Payment sees the relevant failure event and refunds.

Conceptually:

```text
Shipping
   |
   | failure
   v
Inventory
   |
   | compensation event
   v
Payment
   |
   v
Refund
```

This can work.

But notice what happens as the workflow grows.

---

# 13. Choreography's Complexity Problem

Imagine 10 services.

Now you can get:

```text
Service A
  ↓
Service B
  ↓
Service C
  ↓
Service D
  ↓
Service E
  ↓
...
```

And each service needs to understand:

```text
Which events?
Which failures?
Which compensations?
Which next step?
```

The business workflow becomes distributed across many services.

This can become difficult to understand and debug.

That's the primary weakness of choreography.

---

# 14. Orchestration

In orchestration, we introduce a central coordinator:

```text
             Saga Orchestrator
              /      |      \
             /       |       \
            v        v        v
        Payment   Inventory  Shipping
```

The orchestrator knows the workflow.

For example:

```text
1. Create Order
2. Charge Payment
3. Reserve Inventory
4. Create Shipment
```

It explicitly tells services what to do.

---

# 15. Orchestrated Saga Example

The orchestrator starts:

```text
Orchestrator
     |
     | Create Order
     v
Order Service
     |
     | success
     v
Orchestrator
     |
     | Charge Payment
     v
Payment Service
```

Then:

```text
Payment ✓
   |
   v
Orchestrator
   |
   | Reserve Inventory
   v
Inventory
```

Then:

```text
Inventory ✓
   |
   v
Orchestrator
   |
   | Create Shipment
   v
Shipping
```

Everything succeeds:

```text
Saga = COMPLETED
```

---

# 16. Orchestrator Handling Failure

Suppose:

```text
Order ✓
Payment ✓
Inventory ✓
Shipping ✗
```

The orchestrator knows the history:

```text
Order       ✓
Payment     ✓
Inventory   ✓
Shipping    ✗
```

So it can execute:

```text
Release Inventory
       ↓
Refund Payment
       ↓
Cancel Order
```

Diagram:

```text
                 Orchestrator
                  /    |    \
                 /     |     \
                v      v      v
             Order  Payment Inventory
                ↑      ↑       ↑
                |      |       |
             Cancel  Refund  Release
```

This is often easier to reason about for complex workflows.

---

# 17. Choreography vs Orchestration

|                    | Choreography         | Orchestration                   |
| ------------------ | -------------------- | ------------------------------- |
| Coordinator        | None                 | Central orchestrator            |
| Communication      | Events               | Commands + responses/events     |
| Workflow location  | Distributed          | Centralized                     |
| Simple workflows   | Good                 | Good                            |
| Complex workflows  | Can become difficult | Easier to reason about          |
| Central dependency | No                   | Yes                             |
| Debugging          | Can be difficult     | Generally easier                |
| Coupling           | Event-based          | Orchestrator knows participants |

Neither is universally superior.

The right choice depends on the complexity and nature of the workflow.

---

# 18. A Subtle Problem: What If Compensation Fails?

Suppose:

```text
Shipping failed
```

We start compensating:

```text
Release Inventory ✓
Refund Payment    ✗
```

Now we're stuck again.

The customer may still have been charged.

So a Saga itself must handle failures during compensation.

We may need:

```text
Retry
   ↓
Retry
   ↓
Retry
   ↓
Dead Letter / Manual intervention
```

This connects directly to earlier chapters.

Remember:

```text
Retry + Backoff
```

and:

```text
Dead Letter Queue
```

The Saga isn't isolated from the rest of our reliability mechanisms.

It **uses them**.

---

# 19. Why Idempotency Matters

Suppose the orchestrator sends:

```text
Refund Payment
```

The payment service performs it.

But the response is lost:

```text
Payment Service
      |
      | refund successful
      X
      | response lost
      |
Orchestrator
```

The orchestrator doesn't know whether the refund succeeded.

So it retries:

```text
Refund Payment
```

If the payment service isn't idempotent:

```text
Refund #1 = ₹80,000
Refund #2 = ₹80,000
```

The customer could receive:

```text
₹160,000
```

for an ₹80,000 payment.

Therefore Saga implementations generally need **idempotent operations**.

This is a very important connection:

```text
Saga
  +
Retry
  +
Idempotency
```

work together.

---

# 20. Saga State

The system needs to know where the Saga currently is.

For example:

```text
Saga ID: 123

Order       = COMPLETED
Payment     = COMPLETED
Inventory   = COMPLETED
Shipping    = FAILED
Compensation:
Inventory   = COMPLETED
Payment     = PENDING
Order       = PENDING
```

This state is particularly important for an orchestrator.

If the orchestrator crashes:

```text
Orchestrator
     |
     X
```

we need another instance to resume.

So Saga state itself needs durable storage.

---

# 21. Saga Is Usually Eventually Consistent

This is an important conceptual consequence.

During the workflow, we may temporarily have:

```text
Order = CREATED
Payment = COMPLETED
Inventory = RESERVED
Shipping = PENDING
```

The entire system isn't necessarily in one globally consistent final state at every instant.

Eventually:

```text
Order = CONFIRMED
Payment = COMPLETED
Inventory = RESERVED
Shipping = CREATED
```

or:

```text
Order = CANCELLED
Payment = REFUNDED
Inventory = RELEASED
Shipping = NOT CREATED
```

So Saga embraces:

> **Eventual consistency across services in exchange for avoiding a single distributed transaction.**

---

# 22. Where It Helps

Saga is particularly useful when:

### 1. A business transaction spans multiple services

```text
Order
 ↓
Payment
 ↓
Inventory
 ↓
Shipping
```

### 2. Each service owns its own database

```text
DB A
DB B
DB C
DB D
```

### 3. We want services to remain independently deployable

We don't want every service participating in one giant database transaction.

### 4. Operations have meaningful compensations

For example:

```text
Reserve → Release
Charge → Refund
Create → Cancel
```

---

# 23. Where It Doesn't Help

Saga is not always appropriate.

### Simple single-database operation

If everything lives inside one database:

```text
BEGIN
...
COMMIT
```

a normal transaction is usually simpler.

Don't introduce Saga unnecessarily.

---

### Irreversible operations

If an operation cannot be meaningfully compensated:

```text
Send money to external bank
```

you need to think very carefully about whether Saga is appropriate.

---

### Extremely complex workflows

A Saga involving dozens of steps and complicated branching can itself become difficult to manage.

---

### Strong immediate consistency requirements

If the business absolutely requires:

> "All services must observe the change atomically at exactly the same moment."

Saga isn't designed for that.

---

# 24. Tradeoffs

## Advantages

### Avoids one giant distributed transaction

Each service can maintain its own local transaction.

### Works well with microservices

Services can retain ownership of their data.

### Supports failure recovery

Compensation provides a way to undo business effects.

### Can scale independently

Services aren't tied together through one database transaction.

---

## Disadvantages

### Much more complex than a local transaction

You now need:

- workflow state
- compensation logic
- retries
- idempotency
- failure handling

### Eventual consistency

The system can temporarily be in an intermediate state.

### Compensation isn't always possible

Some real-world actions can't be truly undone.

### Debugging can be difficult

Especially with choreography.

### Compensation can fail

You need recovery mechanisms for the recovery mechanism.

That last point is worth remembering.

---

# 25. Common Interview Questions

## Q1. What is a Saga?

A Saga is a sequence of local transactions across services where each successful step has a corresponding compensating action that can undo its business effect if a later step fails.

---

## Q2. Does Saga provide ACID transactions across services?

**No.**

Each service performs its own local transaction.

Saga provides a workflow for achieving the desired business outcome through forward actions and compensations.

---

## Q3. Saga vs distributed transaction?

Saga:

```text
Local transactions
      +
Compensations
```

Distributed transaction:

```text
One coordinated transaction
across multiple participants
```

Saga generally favors availability, autonomy, and eventual consistency.

Distributed transactions provide stronger atomicity but require more coordination.

We'll compare them directly in **Chapter 9**.

---

## Q4. Choreography vs orchestration?

Choreography:

```text
Services react to events.
No central coordinator.
```

Orchestration:

```text
Central orchestrator
controls the workflow.
```

Choreography can be simpler for small workflows but can become difficult to understand as dependencies grow.

Orchestration centralizes workflow logic but introduces a coordinator.

---

## Q5. Why is idempotency important in Saga?

Because messages and commands may be retried.

If:

```text
Refund
```

is executed twice, the result should not accidentally become:

```text
2 × refund
```

Therefore operations need to safely tolerate duplicate execution.

---

## Q6. What if compensation fails?

Treat compensation as a distributed operation that can itself fail.

Possible mechanisms include:

```text
Retry + Backoff
        ↓
Retry
        ↓
Persistent Saga state
        ↓
Alert / Manual intervention
```

The exact strategy depends on business requirements.

---

# 26. Before vs After Architecture

### Before: One monolithic transaction

If everything lives together:

```text
                Application
                    |
             +------+------+
             |             |
          Database      Database
```

We might simply do:

```text
BEGIN TRANSACTION

Create Order
Charge Payment
Reserve Inventory
Create Shipment

COMMIT
```

If something fails:

```text
ROLLBACK
```

---

### After: Independent services

```text
Order Service       → Order DB
     |
Payment Service     → Payment DB
     |
Inventory Service   → Inventory DB
     |
Shipping Service    → Shipping DB
```

There is no single local transaction spanning all four.

Saga introduces:

```text
             Saga
              |
      +-------+-------+
      |       |       |
     T1      T2      T3      T4
      |       |       |       |
     C1      C2      C3      C4
```

If T4 fails:

```text
T1 → T2 → T3 → T4 ✗
          ↑
          |
         C3
          ↑
         C2
          ↑
         C1
```

---

# 27. The Deeper Connection

Now look at the journey through Module 8:

```text
Timeout
    ↓
Don't wait forever

Retry + Backoff
    ↓
Recover carefully

Circuit Breaker
    ↓
Stop unhealthy dependencies

Bulkhead
    ↓
Isolate resources

Rate Limiting
    ↓
Control traffic

Distributed Locking
    ↓
Coordinate resource ownership

Leader Election
    ↓
Coordinate cluster authority

Saga
    ↓
Coordinate multi-service business operations
```

We're now moving from **infrastructure-level coordination** to **business-level coordination**.

And that leads directly to the next chapter.

---

# 28. Connections

## Next: Chapter 9 — Distributed Transactions: 2PC vs Saga

Saga solves the problem by saying:

```text
Don't make everything one transaction.

Perform local transactions
+
compensate when necessary.
```

But there's another approach:

> **What if we actually tried to make all participants commit or abort together?**

That's the idea behind **Two-Phase Commit (2PC)**.

Conceptually:

```text
             Coordinator
             /    |    \
            /     |     \
           v      v      v
         DB A   DB B   DB C

        "Can you all commit?"

              ↓

        "Now commit."
```

This gives us a fundamentally different set of tradeoffs.

We'll compare:

```text
2PC
vs
Saga
```

and understand **why modern distributed architectures often prefer Saga for long-running business workflows, while 2PC remains useful in certain tightly controlled environments.**

---

# 29. Key Takeaways

1. **Saga coordinates a business transaction across multiple independent services.**
2. It breaks the workflow into **local transactions**.
3. Each step can have a **compensating action**.
4. Compensation is **not database rollback**; it is a new business operation that counteracts the previous one.
5. **Choreography** distributes workflow logic among services.
6. **Orchestration** centralizes workflow logic in an orchestrator.
7. Saga usually implies **eventual consistency** rather than atomic consistency across services.
8. Compensation itself can fail, so Saga needs retries, persistence, and recovery mechanisms.
9. **Idempotency is critical** because commands may be retried or delivered more than once.
10. Saga is useful when services own separate data and a business operation spans multiple services.

### One sentence to remember

> **A Saga replaces one impossible-to-rollback distributed transaction with a sequence of local transactions and business-level compensations that gradually drive the system toward a consistent final outcome.**
