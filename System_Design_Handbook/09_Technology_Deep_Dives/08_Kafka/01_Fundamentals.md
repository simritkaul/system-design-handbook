# Module 9 — Chapter 8: Kafka

RabbitMQ gave us one mental model for asynchronous communication:

```text
Producer
   ↓
Broker
   ↓
Queue
   ↓
Worker
```

That model is excellent when the question is:

> **"How do I reliably get this piece of work to a consumer?"**

Kafka starts from a slightly different question:

> **"How do I build a durable, distributed stream of events that many consumers can independently consume, at their own pace, and potentially replay later?"**

That difference is the heart of Kafka.

---

# 1. The Problem

Imagine a large e-commerce system.

When an order is created:

```text
OrderCreated
```

many different systems care about it:

```text
OrderCreated
   │
   ├── Inventory
   ├── Payment
   ├── Notifications
   ├── Fraud Detection
   ├── Analytics
   ├── Recommendation System
   └── Data Warehouse
```

Now imagine that tomorrow you build a new service:

```text
Fraud Detection v2
```

It would be useful if it could process **old order events**, not just future ones.

Or perhaps Analytics goes down for two hours.

When it comes back, you don't want:

> "Sorry, those events happened while you were down. They're gone."

You want:

```text
Analytics
    ↓
Resume from where I stopped
```

This introduces a fundamentally different requirement:

> **Messages need to be durable history, not merely temporary work waiting in a queue.**

---

# 2. Why RabbitMQ's Mental Model Isn't Enough

With a traditional work queue:

```text
Producer
   ↓
Queue
   ↓
Consumer
   ↓
Message processed
   ↓
Message gone
```

That's perfect for:

> "Someone needs to do this job."

But consider:

```text
OrderCreated
```

You might want:

```text
Inventory Service → process it
Analytics Service → process it
Fraud Service → process it
Recommendation Service → process it
```

And tomorrow:

```text
NewAnalyticsService → process ALL historical events
```

Now the message isn't merely:

> **work to be completed**

It is:

> **an event that happened and may be useful to many consumers.**

That's the problem Kafka is designed to address.

---

# 3. The Big Idea

> **Kafka is a distributed event-streaming platform that durably stores ordered sequences of events so multiple consumers can independently read, track their progress, and replay those events.**

The phrase to remember is:

> **Durable event log.**

That's the fundamental Kafka mental model.

---

# 4. The Mental Model: A Log

Imagine a giant append-only log:

```text
Offset
  ↓

0   OrderCreated
1   PaymentCompleted
2   OrderCreated
3   ItemShipped
4   OrderCancelled
5   OrderCreated
6   PaymentCompleted
```

New events are appended:

```text
7   ...
8   ...
9   ...
```

The important thing is:

> **Kafka doesn't primarily think "remove the message when somebody consumes it."**

Instead:

> **The event remains in the log according to the configured retention policy.**

Consumers remember where they are.

---

# 5. Consumer Position

Suppose the log is:

```text
0  A
1  B
2  C
3  D
4  E
5  F
```

Consumer A has processed:

```text
0 → 3
```

So it is currently around:

```text
offset = 4
```

Consumer B might only have processed:

```text
0 → 1
```

So it is around:

```text
offset = 2
```

Kafka doesn't need to delete:

```text
A
B
C
```

just because Consumer A processed them.

Both consumers can have independent positions.

---

# 6. This Is the Fundamental Difference

RabbitMQ's common mental model:

```text
Message
   ↓
Queue
   ↓
Consumer
   ↓
Acknowledged
   ↓
Done
```

Kafka's mental model:

```text
Event
   ↓
Durable log
   ↓
Consumer reads event
   ↓
Consumer advances position
```

The event can still exist.

That's what enables:

- replay
- multiple independent consumers
- recovering from failures
- rebuilding derived systems

This is probably the single most important conceptual distinction between Kafka and RabbitMQ.

---

# 7. Topics

Kafka organizes events into **topics**.

For example:

```text
orders
payments
shipments
user-events
```

You can think of a topic as:

> **A named stream of related events.**

For example:

```text
orders
───────────────────────────────
0  OrderCreated
1  OrderCreated
2  OrderCancelled
3  OrderCreated
4  OrderShipped
5  OrderCreated
...
```

Producers publish events to topics.

Consumers subscribe to topics.

---

# 8. Partitions

Now suppose:

```text
orders
```

contains billions of events.

One machine isn't enough.

So Kafka divides a topic into **partitions**.

```text
orders topic

Partition 0:
0  A
1  B
2  C

Partition 1:
0  D
1  E
2  F

Partition 2:
0  G
1  H
2  I
```

Now the topic can be distributed across machines.

This is where your Module 2 knowledge comes back.

```text
Partitioning
     ↓
Horizontal scale
```

---

# 9. Ordering

Here's an extremely important detail.

Kafka guarantees ordering **within a partition**.

For example:

```text
Partition 0

0 OrderCreated
1 PaymentCompleted
2 OrderShipped
```

The order is preserved.

But across partitions:

```text
Partition 0:
Order A
Order B

Partition 1:
Order C
Order D
```

there isn't one global ordering across the entire topic.

This is a critical interview point.

---

# 10. Why Partitioning Is Necessary

Suppose Kafka had:

```text
1 topic
1 partition
1 consumer
```

Eventually:

```text
traffic ↑
```

and that one partition becomes a bottleneck.

Instead:

```text
             orders
                │
      ┌─────────┼─────────┐
      ▼         ▼         ▼
 Partition 0 Partition 1 Partition 2
      │         │         │
      ▼         ▼         ▼
    Node A    Node B    Node C
```

Now writes and reads can be distributed.

This gives Kafka horizontal scalability.

---

# 11. The Key Determines the Partition

Suppose events contain:

```text
userId
```

Kafka can use a key to determine the partition.

Conceptually:

```text
userId
   ↓
partitioning function
   ↓
partition
```

For example:

```text
user 42 → Partition 1
user 73 → Partition 2
user 91 → Partition 0
```

The important reason for using a key isn't merely distribution.

It's often:

> **Preserving ordering for related events.**

---

# 12. Why the Key Matters

Suppose we have:

```text
OrderCreated
PaymentCompleted
OrderShipped
```

for:

```text
orderId = 123
```

We probably want:

```text
OrderCreated
      ↓
PaymentCompleted
      ↓
OrderShipped
```

to be processed in that order.

If all events for order 123 are assigned to the same partition:

```text
orderId = 123
       ↓
Partition 2
```

then Kafka preserves their ordering within that partition.

So:

> **Partitioning is not merely about scaling; it can also determine your ordering guarantees.**

Very important HLD insight.

---

# 13. Consumers

Now suppose:

```text
orders
 ├── Partition 0
 ├── Partition 1
 └── Partition 2
```

and we have three consumers.

They can process partitions concurrently:

```text
Consumer A → Partition 0
Consumer B → Partition 1
Consumer C → Partition 2
```

This gives us parallel processing.

But Kafka introduces another extremely important concept:

# Consumer Groups

---

# 14. Consumer Groups

Suppose:

```text
Order Processing Service
```

has three instances:

```text
Consumer A
Consumer B
Consumer C
```

They belong to the same **consumer group**.

Kafka distributes partitions across them.

Conceptually:

```text
             orders
          ┌────┼────┐
          ▼    ▼    ▼
         P0   P1   P2
          │    │    │
          ▼    ▼    ▼
         C1   C2   C3
```

The group collectively processes the topic.

Each partition is assigned to one consumer within that group at a time.

---

# 15. Why Consumer Groups Are Powerful

Now imagine we have two completely different applications:

```text
Inventory Service
Analytics Service
```

They both want the same events.

We create:

```text
Consumer Group A
   ↓
Inventory

Consumer Group B
   ↓
Analytics
```

Both can independently consume the same topic.

Conceptually:

```text
                    orders
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
      Inventory Group      Analytics Group
             │                   │
        consumers             consumers
```

This is one of Kafka's superpowers.

> **One event stream can feed many independent applications without those applications competing for the same messages.**

---

# 16. RabbitMQ Comparison

This is where the distinction becomes clearer.

Imagine:

```text
OrderCreated
```

With a queue-based work model:

```text
Queue
 ├── Consumer A
 ├── Consumer B
 └── Consumer C
```

the consumers are typically competing to process the work.

A message goes to one consumer.

With Kafka:

```text
Topic
   │
   ├── Inventory Group
   ├── Analytics Group
   ├── Fraud Group
   └── Recommendation Group
```

each consumer group gets its own logical view of the stream.

So:

```text
RabbitMQ
→ distribute work

Kafka
→ distribute events / maintain event history
```

This isn't an absolute rule, but it's an excellent starting mental model.

---

# 17. Offsets

We said consumers track their position.

That position is represented using an **offset**.

Suppose:

```text
Partition 0

0 A
1 B
2 C
3 D
4 E
```

A consumer processes:

```text
A
B
C
```

Its progress is around:

```text
offset 3
```

If it crashes:

```text
Consumer ✗
```

a new consumer can resume from the stored position.

```text
0 A
1 B
2 C
3 D ← resume
4 E
```

This is why Kafka can support replay and recovery.

---

# 18. Replay

Now imagine Analytics has a bug.

It incorrectly processed events for the last six hours.

Because the events remain in Kafka:

```text
Historical events
       ↓
reset/reposition consumer
       ↓
process again
```

You can replay the events.

That's a completely different capability from:

```text
message consumed
→ message deleted
```

This is why Kafka is so powerful for event-driven architectures.

---

# 19. Retention

But Kafka doesn't store events forever by default.

Events are retained according to configured policies.

Conceptually:

```text
Today
 │
 ▼
Events
 │
 │ retained
 ▼
Retention boundary
 │
 ▼
old events removed
```

Retention could be based on:

```text
Time
or
Storage constraints
```

The key point:

> **Kafka is a durable log, but it is not automatically an infinite historical database.**

---

# 20. Replication

Now let's connect Kafka to distributed systems.

Suppose:

```text
Partition 0
```

is stored on one broker.

If that broker dies:

```text
Broker ✗
```

we don't want to lose the partition.

So Kafka replicates partitions.

Conceptually:

```text
Partition 0
    │
    ├── Replica A
    ├── Replica B
    └── Replica C
```

One replica acts as the leader for normal operations, while replicas provide redundancy.

So again:

```text
Partitioning
+
Replication
=
Scalable + fault-tolerant distributed log
```

---

# 21. Kafka Is Not "Just a Queue"

This is probably the biggest misconception to avoid.

Kafka has queues-like behavior:

```text
Producer
   ↓
Consumers
```

But its fundamental abstraction is:

```text
Distributed append-only log
```

That gives it capabilities such as:

```text
Durable retention
Independent consumer groups
Replay
Partition-level ordering
High-throughput streaming
```

Thinking:

> "Kafka = really big RabbitMQ"

will cause problems later.

---

# 22. Throughput

Kafka is designed particularly well for high-throughput workloads.

Instead of treating every event as an independent request with lots of overhead, Kafka works heavily around:

```text
Append
Batch
Sequential access
Partition
```

Conceptually:

```text
Producer
   ↓
batch of events
   ↓
partition
   ↓
sequential append
```

This architecture is very efficient for large streams of events.

The important HLD conclusion:

> **Kafka is particularly attractive when the system needs to move huge volumes of events continuously.**

---

# 23. Delivery Semantics

You learned this concept in Module 7.

Kafka can be configured around different processing guarantees.

At a high level:

```text
At-most-once
At-least-once
Exactly-once semantics
```

But remember the subtle distinction:

> **Messaging semantics do not automatically make arbitrary business operations exactly once.**

Suppose:

```text
Kafka event
    ↓
Consumer
    ↓
Update external database
    ↓
Consumer crashes
```

Kafka may retry the event.

Your external operation could happen twice unless the overall workflow is designed carefully.

Again:

```text
Delivery guarantee
       ≠
Business-operation guarantee
```

This should now feel familiar from RabbitMQ.

---

# 24. What Happens When a Consumer Is Slow?

Suppose:

```text
Producer
→ 100,000 events/sec

Consumer
→ 20,000 events/sec
```

Kafka doesn't necessarily need to block the producer immediately.

Instead:

```text
Partition
   ↓
events accumulate
   ↓
consumer catches up
```

The consumer develops **lag**.

Conceptually:

```text
Latest event:     1,000,000
Consumer position:  900,000

Lag = 100,000
```

Consumer lag is one of the most important operational metrics in Kafka.

---

# 25. Scaling Consumers

Suppose we have:

```text
6 partitions
```

and:

```text
3 consumers
```

Kafka can distribute:

```text
C1 → P0 P1
C2 → P2 P3
C3 → P4 P5
```

Now suppose we add:

```text
C4
C5
C6
```

we can process partitions more concurrently.

But here's an important limitation:

> **You generally cannot get useful parallelism beyond the number of partitions for a consumer group.**

If:

```text
3 partitions
10 consumers
```

some consumers will have nothing to process.

Therefore:

```text
Partitions
   ↓
maximum useful parallelism
```

This is a critical Kafka scaling concept.

---

# 26. Too Many Partitions Isn't Free Either

You might think:

> "Then let's create one million partitions."

Not so fast.

More partitions mean more:

- metadata
- coordination
- resource consumption
- operational complexity

So partition count is an architectural decision.

You want enough partitions for:

```text
Expected throughput
+
parallelism
+
future growth
```

without creating unnecessary overhead.

---

# 27. Ordering vs Parallelism

This creates a beautiful distributed-systems tradeoff.

Suppose we want strict global ordering:

```text
A
B
C
D
E
```

One partition makes this straightforward.

But:

```text
1 partition
→ limited parallelism
```

If we create many partitions:

```text
P0
P1
P2
P3
```

we gain throughput.

But now there is no single global order across all partitions.

So:

```text
More partitions
     ↓
More parallelism
     ↓
Less global ordering
```

This is exactly the kind of tradeoff you should be able to explain in an HLD interview.

---

# 28. Kafka as an Event Backbone

Now imagine a large system:

```text
                       Kafka
                         │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
   Inventory          Analytics          Fraud
       │                 │                 │
       ▼                 ▼                 ▼
   Database          Data Lake         ML System
```

Kafka becomes the central event backbone.

Services don't necessarily need direct connections to each other.

Instead:

```text
Service A
   ↓
Kafka
   ↓
Service B
```

and:

```text
Service C
   ↑
Kafka
```

This can dramatically reduce point-to-point coupling.

---

# 29. But Kafka Can Become a Coupling Point

This is an important counterpoint.

If everything depends on Kafka:

```text
Kafka
 ├── Service A
 ├── Service B
 ├── Service C
 ├── Service D
 └── Service E
```

then Kafka becomes extremely important infrastructure.

If it has problems:

```text
Kafka ✗
```

many asynchronous workflows can be affected.

So introducing Kafka isn't:

> "Let's decouple everything."

It is more accurately:

> **"Let's move coupling from direct service-to-service communication toward an event contract and a shared streaming infrastructure."**

That's better, but it's not zero coupling.

---

# 30. Event Schema Matters

Suppose we publish:

```text
OrderCreated
```

and ten services consume it.

Then changing the event from:

```text
{
    orderId,
    userId
}
```

to:

```text
{
    id,
    customer,
    metadata
}
```

can break consumers.

So Kafka architectures introduce another important concept:

> **Event contracts and schema evolution.**

You don't need to become an expert in schema-registry tooling yet.

Understand the architectural problem:

```text
One producer
+
Many consumers
=
Changing event structure safely becomes important
```

---

# 31. When Would I Choose Kafka?

Strong signals include:

```text
Huge event volume
+
Multiple independent consumers
+
Durable event history
+
Replay
+
Streaming pipelines
+
Consumer independence
+
High throughput
```

Examples:

### Activity/event tracking

```text
UserClicked
UserViewedProduct
UserAddedToCart
```

### Financial events

```text
PaymentCreated
PaymentCompleted
PaymentFailed
```

### Data pipelines

```text
Application events
     ↓
Kafka
     ↓
Analytics / Warehouse / ML
```

### Microservice event backbone

```text
Service
   ↓
Event
   ↓
Kafka
   ↓
many consumers
```

---

# 32. When Would I Not Choose Kafka?

Suppose you simply need:

```text
Upload image
   ↓
Process image asynchronously
```

with:

```text
Producer
   ↓
Queue
   ↓
Worker
```

and you don't need:

- replay
- many independent consumers
- long-lived event history
- massive event streaming

Kafka may be unnecessary complexity.

A traditional message queue may be a better fit.

This is an important principle:

> **Don't choose Kafka because your architecture diagram looks more impressive.**

Choose it because your workload needs the properties Kafka provides.

---

# 33. Kafka vs RabbitMQ

Let's make the comparison explicit.

|                                | RabbitMQ                         | Kafka                         |
| ------------------------------ | -------------------------------- | ----------------------------- |
| Primary mental model           | Message broker / queue           | Distributed event log         |
| Messages                       | Typically work to process        | Durable events in a stream    |
| Consumption                    | Message delivery/acknowledgement | Consumer tracks offset        |
| Replay                         | Not the central model            | Core capability               |
| Multiple independent consumers | Possible                         | Core strength                 |
| Routing                        | Very flexible                    | Topic/partition model         |
| Ordering                       | Queue-dependent                  | Strong within partition       |
| Huge event throughput          | Good                             | Core strength                 |
| Work queues                    | Excellent                        | Possible, but different model |
| Event streaming                | Possible                         | Excellent                     |
| Retention                      | Queue/message lifecycle          | Explicit event retention      |

Don't interpret this as:

```text
RabbitMQ = old/bad
Kafka = new/good
```

They solve overlapping but different problems.

---

# 34. A Real HLD Scenario

Let's design a ride-sharing platform.

Events:

```text
RideRequested
DriverAssigned
DriverArrived
RideStarted
RideCompleted
PaymentCompleted
```

We want:

```text
Trip Service
Analytics
Fraud
Notifications
Pricing
ML
```

to consume these events.

A Kafka architecture could look like:

```text
                    Services
                       │
                       ▼
                     Kafka
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     Analytics       Fraud          Pricing
     Group           Group          Group
        │              │              │
        ▼              ▼              ▼
    Warehouse       Fraud DB       Pricing DB
```

If Analytics crashes for two hours:

```text
Kafka
  │
  └── events continue being retained

Analytics returns
  ↓
resume from previous offset
```

That's exactly where Kafka shines.

---

# 35. A More Interesting Scenario: Rebuilding a Service

Suppose:

```text
Recommendation Service
```

has a bug.

We delete its derived database.

Can we rebuild it?

If relevant historical events remain in Kafka:

```text
Kafka
  ↓
Replay events
  ↓
Rebuild Recommendation DB
```

This is a very powerful architecture.

Kafka effectively becomes:

> **a durable history of what happened.**

Not necessarily the source of truth for every business entity, but a durable event stream from which downstream systems can derive state.

---

# 36. Kafka and Event Sourcing

This connects directly to Module 8.

You learned:

> **Event Sourcing stores state changes as events rather than only storing the latest state.**

Kafka can be used as infrastructure in event-driven/event-sourced architectures.

But don't conclude:

```text
Kafka = Event Sourcing
```

They're not the same thing.

Kafka is a streaming platform.

Event sourcing is an architectural pattern.

Kafka can support it, but doesn't automatically make your system event-sourced.

Again:

```text
Concept
   ↓
Pattern
   ↓
Technology
```

Module 9 is teaching you how these layers relate.

---

# 37. The Biggest Kafka Tradeoffs

### Advantages

- Extremely high throughput
- Horizontal scalability
- Durable event retention
- Replay
- Multiple independent consumers
- Strong ordering within partitions
- Excellent event-streaming backbone
- Consumers can process at different rates

### Disadvantages

- Operational complexity
- Partition planning matters
- Ordering across partitions is difficult
- Consumer lag must be managed
- Event schemas become contracts
- Duplicate processing still needs consideration
- Can be excessive for simple background jobs

---

# 38. The Most Important Kafka Tradeoff

If the interviewer asks:

> **"What's the biggest thing you need to think about when designing Kafka?"**

One excellent answer is:

> **"Partitioning, because it simultaneously determines scalability, parallelism, and often the ordering boundary."**

That's a very strong answer.

Because:

```text
Partition count
      │
      ├── throughput
      ├── consumer parallelism
      ├── ordering scope
      └── data distribution
```

One design decision affects multiple properties.

---

# 39. Kafka Mental Model

Keep this diagram:

```text
                    Kafka Topic
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
           Partition  Partition  Partition
              │          │          │
              ▼          ▼          ▼
          Event log   Event log   Event log
              │          │          │
              └──────────┼──────────┘
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
       Consumer Group A      Consumer Group B
          Inventory             Analytics
```

And the one-sentence mental model:

> **Kafka is a distributed, durable event log where producers append events and independent consumer groups track their own position and process those events at their own pace.**

---

# 40. RabbitMQ → Kafka: The Evolution

This is the important learning journey:

```text
RabbitMQ

"How do I reliably hand work
from producers to workers?"

            ↓

Kafka

"How do I maintain a durable,
scalable stream of events that
many independent consumers can
process and replay?"
```

Neither supersedes the other.

The requirement determines the choice.

---

## Next: Amazon SQS

We've now covered two very different messaging philosophies:

```text
RabbitMQ
   ↓
Broker + queues + routing + workers

Kafka
   ↓
Distributed event log + partitions
+ consumer groups + replay
```

Now we'll introduce a third dimension:

> **What if I want a queue, but I don't want to operate a messaging broker at all?**

That's where **Amazon SQS** comes in.
