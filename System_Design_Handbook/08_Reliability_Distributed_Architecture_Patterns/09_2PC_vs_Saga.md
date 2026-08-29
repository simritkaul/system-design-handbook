# Module 8 — Chapter 9: Distributed Transactions — 2PC vs Saga

## 1. Goal

**Understand how to maintain correctness when one logical operation spans multiple independent systems, and why we might choose a distributed transaction such as 2PC or a Saga.**

---

# 2. The Problem

Let's return to our e-commerce example.

A customer places an order:

```text
              Place Order
                   |
       +-----------+-----------+
       |           |           |
       v           v           v
    Order       Payment     Inventory
    Service     Service     Service
```

Each service owns its own data:

```text
Order DB
Payment DB
Inventory DB
```

Now imagine:

```text
Order       ✓
Payment     ✓
Inventory   ✗
```

The business operation is now incomplete.

The question is:

> **How do we make several independent systems behave like one logical transaction?**

There are two major approaches we'll compare:

```text
Distributed Transaction
        |
        +── 2PC
        |
        +── Saga
```

They solve the same broad problem, but using very different philosophies.

---

# 3. Why Existing Solutions Fail

A normal database transaction gives us:

```text
BEGIN

Operation A
Operation B
Operation C

COMMIT
```

Either everything commits:

```text
A ✓
B ✓
C ✓
```

or everything rolls back:

```text
A ✗
B ✗
C ✗
```

This works because one database controls the transaction.

But with multiple services:

```text
Service A → DB A

Service B → DB B

Service C → DB C
```

there isn't naturally one transaction manager controlling all three.

We need some way to coordinate them.

---

# 4. The Big Idea

There are two fundamentally different strategies.

### 2PC

> **Ask all participants whether they can commit, then tell them all to commit only if everyone agrees.**

### Saga

> **Let each service commit locally, and if something later fails, execute compensating actions for the work that already happened.**

So:

```text
2PC:
Prepare → Prepare → Prepare
             ↓
          Commit
```

versus:

```text
Saga:
T1 → T2 → T3 → failure
               ↓
             C2 → C1
```

The distinction is fundamental.

---

# 5. Two-Phase Commit (2PC)

Let's start with 2PC.

2PC introduces a **coordinator**.

```text
                  Coordinator
                 /     |     \
                /      |      \
               v       v       v
             DB A    DB B    DB C
```

The coordinator manages the distributed transaction.

There are two phases:

```text
Phase 1 → Prepare
Phase 2 → Commit
```

Hence:

**Two-Phase Commit.**

---

# 6. Phase 1 — Prepare

Suppose we want to perform:

```text
Order creation
Payment
Inventory reservation
```

The coordinator asks each participant:

> "Can you commit this transaction?"

```text
             Coordinator
             /    |    \
            ↓     ↓     ↓
          DB A   DB B   DB C

          "Prepare?"
```

Each participant performs the necessary local work and reaches a state where it is ready to commit.

For example:

```text
DB A → YES
DB B → YES
DB C → YES
```

The coordinator now knows:

```text
Everyone is prepared.
```

But they haven't necessarily committed yet.

---

# 7. Phase 2 — Commit

Now the coordinator sends:

```text
COMMIT
```

to everyone.

```text
             Coordinator
             /    |    \
            ↓     ↓     ↓
         COMMIT COMMIT COMMIT
```

Participants commit:

```text
DB A → COMMITTED
DB B → COMMITTED
DB C → COMMITTED
```

The distributed transaction succeeds.

---

# 8. What If One Participant Says No?

Suppose:

```text
DB A → YES
DB B → YES
DB C → NO
```

The coordinator cannot safely commit the transaction.

So it tells everyone:

```text
ABORT
```

```text
             Coordinator
             /    |    \
            ↓     ↓     ↓
          ABORT ABORT ABORT
```

The transaction is rolled back.

Conceptually:

```text
Prepare
  ↓
Everyone says YES?
  |
  +── YES → COMMIT
  |
  +── NO  → ABORT
```

This gives us a strong all-or-nothing model.

---

# 9. Why Is 2PC Attractive?

Because it looks very much like the transaction model we're already familiar with.

We want:

```text
A ✓
B ✓
C ✓
```

or:

```text
A ✗
B ✗
C ✗
```

No business-level compensation is required.

The coordinator ensures everyone agrees before committing.

This provides stronger atomicity than a Saga.

But there is a major price.

---

# 10. The Blocking Problem

Let's say everyone has prepared:

```text
DB A → PREPARED
DB B → PREPARED
DB C → PREPARED
```

Then:

```text
Coordinator
     |
     X
   CRASH
```

The participants don't know whether the coordinator intended:

```text
COMMIT
```

or:

```text
ABORT
```

So they may have to wait.

```text
DB A
  |
  | "What should I do?"
  |
  v
waiting...
```

This is one of the fundamental weaknesses of classic 2PC.

Participants can be left holding prepared state while waiting for the coordinator's decision.

---

# 11. Why Can't a Participant Just Commit?

Suppose DB A says:

> "I'll just commit."

But perhaps DB B has already discovered a problem.

If A commits while B aborts:

```text
A → COMMIT
B → ABORT
```

we've broken atomicity.

Therefore A cannot safely decide independently.

---

# 12. Why Can't It Just Abort?

Same problem.

Suppose the coordinator had actually decided:

```text
COMMIT
```

and some participants already committed.

If A independently aborts:

```text
A → ABORT
B → COMMIT
C → COMMIT
```

again, we've broken atomicity.

So participants need the coordinator's decision.

That's the fundamental coordination cost of 2PC.

---

# 13. Performance Cost

2PC requires multiple rounds of communication.

Conceptually:

```text
Coordinator
   |
   | PREPARE
   v
Participants
   |
   | YES
   v
Coordinator
   |
   | COMMIT
   v
Participants
```

Every distributed transaction therefore incurs:

- network round trips
- coordination overhead
- persistent transaction state
- locks or reserved resources
- dependency on coordinator availability

And participants may need to hold resources while waiting.

This becomes particularly painful for long-running business operations.

---

# 14. Why Long Transactions Are Dangerous

Imagine:

```text
Customer places order
```

and the transaction involves:

```text
Payment provider
Inventory
Shipping
Fraud detection
External partner
```

If we're using a distributed transaction, participants may need to remain prepared while the entire operation is coordinated.

Now imagine one external dependency takes:

```text
30 seconds
```

or:

```text
2 minutes
```

Holding transactional resources for that long is expensive.

Distributed transactions work best when participants are tightly controlled and operations are relatively short.

---

# 15. Saga Takes a Different Approach

Saga says:

> "Why are we trying to keep everyone inside one transaction?"

Instead:

```text
Order → commit
Payment → commit
Inventory → commit
Shipping → commit
```

Each service owns its local transaction.

If something fails:

```text
Shipping ✗
```

we compensate:

```text
Inventory → release
Payment → refund
Order → cancel
```

So:

```text
2PC:
        Coordinate BEFORE committing

Saga:
        Commit locally
        compensate AFTER failure
```

---

# 16. Direct Comparison

Here's the core difference.

|                        | 2PC                            | Saga                        |
| ---------------------- | ------------------------------ | --------------------------- |
| Coordination           | Central coordinator            | Orchestrator or events      |
| Transactions           | One distributed transaction    | Multiple local transactions |
| Failure handling       | Abort/rollback                 | Compensation                |
| Atomicity              | Stronger                       | Not globally atomic         |
| Consistency            | Stronger                       | Usually eventual            |
| Long-running workflows | Poor fit                       | Good fit                    |
| Availability           | Can suffer during coordination | Generally better            |
| Complexity             | Infrastructure-heavy           | Business-logic-heavy        |

The tradeoff is essentially:

```text
2PC
↓
stronger atomicity
but
more coordination + blocking
```

versus:

```text
Saga
↓
more availability + autonomy
but
eventual consistency + compensation complexity
```

---

# 17. Saga Doesn't "Rollback"

This deserves emphasis.

Suppose:

```text
Payment → ₹80,000 charged
```

Saga compensation:

```text
Refund → ₹80,000
```

The payment transaction itself wasn't rolled back.

We performed another business operation.

Therefore:

```text
2PC:

Charge
  ↓
ROLLBACK
```

versus:

```text
Saga:

Charge
  ↓
Refund
```

This difference has major implications.

---

# 18. What If Compensation Isn't Possible?

Suppose:

```text
Send physical package
```

and then:

```text
Shipping fails
```

Can we simply undo the shipment?

Maybe not.

Or:

```text
Send email
```

Once sent:

```text
email → cannot be unsent
```

This is one reason Saga requires **business-aware compensation**.

The system designer has to understand:

> What does "undo" mean for this particular business operation?

That is not something infrastructure can automatically determine.

---

# 19. A Better Way to Think About the Choice

Don't ask:

> "Is 2PC better than Saga?"

Instead ask:

> **"What consistency guarantee does this business operation actually require?"**

For example:

### Bank-like operation

If moving money between tightly controlled accounts requires atomicity:

```text
Account A -₹100
Account B +₹100
```

we may need very strong transactional guarantees.

---

### E-commerce order

Suppose:

```text
Order
Payment
Inventory
Shipping
```

It may be acceptable for the system to temporarily show:

```text
Payment = SUCCESS
Order = PROCESSING
```

and later resolve the workflow.

Saga can be a much better fit.

---

# 20. When 2PC Makes Sense

2PC can make sense when:

### Participants are tightly controlled

For example:

```text
Service A
Service B
Service C
```

all under the same organizational and infrastructure control.

---

### Strong atomicity is essential

You genuinely need:

```text
all commit
or
all abort
```

---

### Transactions are short

The participants can prepare and commit quickly.

---

### Participants support the protocol

All participants need to understand the distributed transaction mechanism.

You can't casually coordinate arbitrary external APIs with 2PC.

---

# 21. When Saga Makes Sense

Saga is a better fit when:

### Operations are long-running

```text
Order
 ↓
Fraud check
 ↓
Payment
 ↓
Inventory
 ↓
Shipping
```

may take a significant amount of time.

---

### Services own independent databases

```text
Service A → DB A
Service B → DB B
Service C → DB C
```

---

### Eventual consistency is acceptable

The system can temporarily be:

```text
Order = PROCESSING
```

before reaching:

```text
Order = CONFIRMED
```

---

### Business operations have meaningful compensation

```text
Reserve → Release
Charge → Refund
Create → Cancel
```

---

# 22. The Availability Tradeoff

This is an important interview-level insight.

Imagine a network partition:

```text
Coordinator
    |
    X
    |
Participants
```

With 2PC, participants may be unable to determine the final transaction decision.

So they may block.

Saga doesn't require every participant to remain inside one global transaction.

Individual services can continue performing local transactions, and the workflow can recover through events, retries, and compensation.

This generally gives Saga better behavior for long-running distributed workflows.

But we pay for that with weaker immediate consistency.

---

# 23. Saga's Hidden Cost

Saga sounds easy:

```text
T1 → T2 → T3
         ↓
        fail
         ↓
       C2 → C1
```

But real systems are messy.

What if:

```text
T3 fails
C2 fails
C2 retry fails
orchestrator crashes
message is duplicated
message is delayed
service is unavailable
```

Now we need:

```text
Saga
 + Retry
 + Idempotency
 + Durable state
 + Timeouts
 + Dead-letter handling
 + Observability
```

So Saga doesn't eliminate distributed complexity.

It **moves the complexity**.

---

# 24. 2PC's Hidden Cost

2PC also has complexity:

```text
Coordinator
Participants
Prepare state
Commit state
Abort state
Recovery
Timeouts
Coordinator failure
Participant failure
Network partitions
```

So neither is "simple."

The difference is **where the complexity lives**.

### 2PC

Complexity is primarily in the **transaction infrastructure and coordination protocol**.

### Saga

Complexity is primarily in **business workflow and compensation logic**.

This is a very useful way to explain the tradeoff in an interview.

---

# 25. Common Interview Questions

## Q1. What is 2PC?

A distributed transaction protocol in which a coordinator first asks all participants to prepare, then commits the transaction only if all participants successfully prepare.

---

## Q2. What are the two phases?

### Phase 1 — Prepare

Participants perform necessary work and indicate whether they can commit.

### Phase 2 — Commit/Abort

The coordinator makes the final decision:

```text
Everyone prepared → COMMIT
Anyone failed     → ABORT
```

---

## Q3. What's the biggest weakness of 2PC?

**Blocking and coordination overhead.**

If the coordinator fails after participants prepare but before they know the final decision, participants may have to wait.

---

## Q4. Why is Saga better for microservices?

Not automatically better, but often a better fit because:

- each service keeps ownership of its database
- services don't need to participate in one global transaction
- workflows can be long-running
- failures can be handled through compensation
- services can remain relatively autonomous

---

## Q5. Is Saga atomic?

**No.**

A Saga doesn't provide global ACID atomicity.

It provides a mechanism for reaching a correct business outcome through local transactions and compensating actions.

---

## Q6. Does Saga guarantee immediate consistency?

No.

Intermediate states are possible.

```text
Payment = SUCCESS
Order = PROCESSING
Inventory = RESERVED
```

Eventually the Saga reaches a final state.

---

## Q7. Why not always use Saga?

Because compensation may be impossible or extremely complicated.

If strong atomicity is essential and participants are tightly controlled, a distributed transaction may be more appropriate.

---

## Q8. Why not always use 2PC?

Because distributed coordination can:

- increase latency
- hold resources
- reduce availability during failures
- create blocking
- become problematic for long-running workflows

---

# 26. Before vs After Architecture

### Traditional local transaction

```text
                 Application
                     |
                     v
                  Database

              BEGIN
                |
             Work A
                |
             Work B
                |
             Work C
                |
              COMMIT
```

Simple.

---

### 2PC architecture

```text
                 Coordinator
                /     |     \
               /      |      \
              v       v       v
            DB A    DB B    DB C

          Phase 1: PREPARE
                ↓
          Phase 2: COMMIT
```

Strong coordination.

---

### Saga architecture

```text
              Saga
               |
      +--------+--------+
      |        |        |
      v        v        v
     T1       T2       T3
      |        |        |
     DB       DB       DB

If T3 fails:

      T1 → T2 → T3 ✗
              ↑
             C2
              ↑
             C1
```

Local transactions plus compensation.

---

# 27. The Deeper Connection

We can now see the evolution:

```text
Single Database
      ↓
ACID Transaction
      ↓
Multiple Services
      ↓
"How do we coordinate?"
      |
      +───────────────+
      |               |
     2PC             Saga
      |               |
 Strong atomicity   Eventual consistency
      |               |
 Coordination       Compensation
```

And this chapter connects almost everything we've learned so far:

```text
Timeout
   ↓
Don't wait forever

Retry
   ↓
Recover transient failures

Idempotency
   ↓
Make retries safe

Distributed Lock
   ↓
Coordinate ownership

Leader Election
   ↓
Coordinate authority

Saga
   ↓
Coordinate business workflows

2PC
   ↓
Coordinate atomic commit
```

We're essentially building the vocabulary needed to reason about **failure in distributed systems**.

---

# 28. Connections

The next chapter is:

## Chapter 10 — CQRS

So far we've been asking:

> "How do we safely change state across multiple services?"

Now we're going to look at a different problem:

> **"What if the way we write data and the way we read data have completely different requirements?"**

Imagine:

```text
Writes:
100 / second

Reads:
1,000,000 / second
```

Or:

```text
Write model:
optimized for correctness

Read model:
optimized for extremely fast queries
```

Using exactly the same model for both can become limiting.

That leads us to:

```text
          Commands
             |
             v
        Write Model
             |
             v
           Data
             |
             v
         Read Model
             |
             v
           Queries
```

This is **Command Query Responsibility Segregation — CQRS**.

---

# 29. Key Takeaways

1. **2PC and Saga solve the broad problem of coordinating multi-system operations in very different ways.**
2. **2PC uses a coordinator and two phases: Prepare → Commit/Abort.**
3. 2PC provides stronger atomicity but introduces coordination overhead and can block during failures.
4. **Saga uses local transactions plus compensating actions.**
5. Saga provides eventual consistency rather than global ACID atomicity.
6. Saga is often better suited to long-running, business-oriented workflows.
7. 2PC is more appropriate when strong atomicity is essential and participants are tightly controlled.
8. Saga's complexity lives primarily in business logic, compensation, retries, and recovery.
9. 2PC's complexity lives primarily in transaction coordination, participant state, and failure recovery.
10. **Neither is universally better—the business consistency requirement determines the appropriate approach.**

### One sentence to remember

> **2PC tries to make distributed systems behave like one atomic transaction; Saga accepts that they are separate systems and coordinates them toward a consistent business outcome through local commits and compensation.**
