# Chapter 4 — Event Streaming

## Goal

Understand why some systems need more than delivering a message once, why events sometimes need to be stored as a durable sequence, and how event streaming allows multiple consumers to process, replay, and independently track a continuous flow of events.

---

# 1. The Problem

In the previous chapter, we learned about Publish / Subscribe.

A service publishes an event:

```text
OrderCreated
```

Multiple systems can react:

```text
                 ┌──► Notification
                 │
OrderCreated ────┼──► Analytics
                 │
                 ├──► Loyalty
                 │
                 └──► Fraud Detection
```

This solved an important problem.

One event can reach many independent consumers.

But now imagine a large e-commerce company.

Every second, thousands of events are happening:

```text
OrderCreated
OrderPaid
OrderCancelled
ProductViewed
ProductAddedToCart
PaymentFailed
ShipmentCreated
OrderDelivered
```

Day after day.

Month after month.

Now suppose the analytics team builds a new service.

They ask:

> Can we analyze all orders from the last six months?

A simple message-processing model might say:

```text
Event arrives
     ↓
Consumer processes it
     ↓
Event is gone
```

That does not help.

The analytics team was not even consuming those events six months ago.

Or imagine a consumer goes offline for several hours.

During that time:

```text
10:00 ─ Event 1
10:01 ─ Event 2
10:02 ─ Event 3
10:03 ─ Event 4
10:04 ─ Event 5
```

When it comes back:

> Can it continue from where it stopped?

Or perhaps a bug is discovered.

```text
Analytics Service
      │
      ▼
Processed events incorrectly
```

The engineering team fixes the bug.

Now they ask:

> Can we process the old events again using the corrected logic?

This requires a different way of thinking.

---

# 2. Why Existing Solutions Fail

Let's compare the models we have seen so far.

## Direct Communication

```text
Service A
   │
   ▼
Service B
```

The request happens now.

The response happens now.

Once complete, the interaction is over.

There is no built-in history.

---

## Message Queue

```text
Producer
   │
   ▼
Queue
   │
   ▼
Consumer
```

The focus is:

> Deliver this unit of work to a worker.

Conceptually:

```text
Task
  ↓
Process
  ↓
Done
```

Once successfully handled, the task may no longer be needed.

---

## Publish / Subscribe

```text
Publisher
   │
   ▼
Topic
   │
   ├── Subscriber A
   ├── Subscriber B
   └── Subscriber C
```

The focus is:

> This happened. Interested systems can react.

But now we need something more.

We need:

```text
Event history
```

We need consumers to move independently.

We need the possibility of:

```text
Read now
Read later
Pause
Resume
Replay
Reprocess
```

That leads us to event streaming.

---

# 3. The Big Idea

Instead of treating an event as something that disappears after being consumed, we treat events as an **ordered, durable sequence of records**.

Conceptually:

```text
Event 1
   ↓
Event 2
   ↓
Event 3
   ↓
Event 4
   ↓
Event 5
   ↓
Event 6
   ↓
...
```

This sequence is a stream.

Consumers can move through that stream at their own pace.

```text
Event Stream

[1] [2] [3] [4] [5] [6] [7] [8] [9]
                    ▲
                    │
              Consumer A

          ▲
          │
    Consumer B

                              ▲
                              │
                        Consumer C
```

The fundamental idea is:

> **Events are not just delivered. They become a sequence that consumers can read.**

---

# 4. From Messages to a Log

A useful mental shift is this.

A traditional queue often feels like:

```text
Inbox

┌──────────────┐
│ Task 1       │
│ Task 2       │
│ Task 3       │
└──────────────┘

Take one
```

An event stream is more like an append-only log.

```text
Beginning
   │
   ▼

[ Event 1 ]
      ↓
[ Event 2 ]
      ↓
[ Event 3 ]
      ↓
[ Event 4 ]
      ↓
[ Event 5 ]
      ↓

      ...
```

New events are continuously added to the end.

```text
Existing Stream

[1] [2] [3] [4] [5]

New event arrives

[1] [2] [3] [4] [5] [6]
```

We do not necessarily remove Event 1 just because someone read it.

That is the key difference.

---

# 5. Producers Append Events

Imagine an e-commerce system.

The Order Service creates an order.

```text
Order Service
      │
      ▼
OrderCreated
```

Instead of thinking:

```text
Send this event to someone.
```

we think:

```text
Add this event to the order event stream.
```

```text
Order Event Stream

[OrderCreated #101]
[OrderPaid #101]
[OrderCreated #102]
[OrderCreated #103]
[OrderCancelled #102]
...
```

New events keep arriving.

```text
                        New event
                            │
                            ▼
[1] [2] [3] [4] [5] [6] [7] [8]

```

The stream becomes a continuously growing history.

---

# 6. Consumers Track Their Own Position

Suppose Consumer A has processed the first five events.

```text
Stream

[1] [2] [3] [4] [5] [6] [7] [8]
                    ▲
                    │
              Consumer A
```

Consumer B may be slower.

```text
Stream

[1] [2] [3] [4] [5] [6] [7] [8]
          ▲
          │
    Consumer B
```

Consumer C might have just started.

```text
Stream

[1] [2] [3] [4] [5] [6] [7] [8]
 ▲
 │
Consumer C
```

Each consumer has its own position.

This position is often conceptually represented as an **offset** or checkpoint.

For example:

```text
Consumer A → Last processed: 5

Consumer B → Last processed: 2

Consumer C → Last processed: 0
```

The important insight is:

> One consumer moving forward does not necessarily affect another consumer.

Consumer A processing Event 5 does not mean Consumer B loses access to Event 5.

---

# 7. Replay

Now we get one of the most powerful properties of event streaming.

Suppose the Analytics Service had a bug.

It processed:

```text
Event 1 ✓
Event 2 ✓
Event 3 ✕ Incorrectly
Event 4 ✕ Incorrectly
Event 5 ✕ Incorrectly
```

The bug is fixed.

With a traditional consume-and-discard model:

```text
Too late.
```

The events are gone.

With an event stream:

```text
Reset position
      │
      ▼
Replay Event 3
      │
      ▼
Replay Event 4
      │
      ▼
Replay Event 5
```

Conceptually:

```text
Before

[1] [2] [3] [4] [5] [6]
                    ▲
                    │
                 Consumer

Reset

[1] [2] [3] [4] [5] [6]
          ▲
          │
       Consumer

Read again ───────────►
```

This is called **replaying events**.

---

# 8. Why Replay Is Powerful

Replay allows systems to do things that are difficult with one-time message delivery.

## Fix Bugs

```text
Old processing logic
       ↓
Incorrect results
       ↓
Fix logic
       ↓
Replay history
```

---

## Build New Consumers

Imagine a company creates a new service.

```text
Fraud Detection Service
```

It did not exist last year.

But the company has retained order events.

```text
Order Stream

Jan ─────────────────────────── Aug
[1][2][3][4][5] ... [Millions of events]
```

The new service can start from historical data.

```text
Start here
    │
    ▼
[Old events] ───────────► [Current events]
```

---

## Rebuild Derived Data

Suppose a service maintains:

```text
Daily Sales Dashboard
```

The database becomes corrupted.

If the original events still exist:

```text
Order events
      │
      ▼
Rebuild dashboard
```

The derived state can potentially be recreated.

This is an important idea.

Sometimes:

```text
Events = Source history

Database view = Current interpretation
```

We will revisit this later when we study:

- CQRS
- Event Sourcing

For now, do not confuse event streaming with those patterns.

Event streaming gives us a durable flow of events.

CQRS and Event Sourcing are architectural patterns that may use event streams, but they solve different problems.

---

# 9. Consumers Can Move at Different Speeds

Suppose an order stream receives:

```text
100,000 events/sec
```

Different consumers may have different needs.

```text
Fraud Detection
    → Needs low latency

Analytics
    → Can process slightly later

Data Warehouse
    → May process in batches

Machine Learning
    → May replay historical data
```

Their positions may look like:

```text
Stream

[1][2][3][4][5][6][7][8][9][10]

Fraud Detection                    ▲

Analytics                     ▲

Data Warehouse          ▲

ML Training       ▲
```

The stream allows each consumer to progress independently.

This is very different from:

```text
Message removed
      ↓
Nobody else can read it
```

---

# 10. Retention

Of course, we cannot store everything forever without cost.

Suppose we receive:

```text
1 million events per second
```

If every event is stored forever:

```text
Storage
   │
   ▼
████████████████████████████
```

Eventually, the cost becomes enormous.

So event streams usually need a retention policy.

Conceptually:

```text
Keep events for:

7 days
30 days
90 days
1 year
Forever
```

For example:

```text
Today

◄──────── 7 days ────────►

[Old events] [Recent events] [New events]
     ✕             ✓             ✓
```

Once an event falls outside the retention window, it may no longer be available.

Therefore:

> Replay is only possible while the required history still exists.

Retention is an engineering decision.

---

# 11. The Difference Between Delivery and Retention

This distinction is extremely important.

Imagine:

```text
Producer
   │
   ▼
Event Stream
   │
   ├── Consumer A
   └── Consumer B
```

There are two separate questions.

## Question 1: Was the event delivered?

```text
Did Consumer A receive Event 10?
```

## Question 2: Is the event still stored?

```text
Can Consumer A read Event 10 again tomorrow?
```

These are different concerns.

A system may successfully deliver an event but still retain it.

```text
Delivered ✓
Still stored ✓
```

This allows replay.

---

# 12. Event Streaming vs Message Queues

Let's make the distinction clearer.

## Message Queue

Think:

```text
Task
   │
   ▼
Someone should do this.
```

Example:

```text
Generate invoice for Order #123.
```

Architecture:

```text
Producer
   │
   ▼
Queue
   │
   ▼
Worker
```

The primary concern is:

> Who will process this work?

---

## Event Streaming

Think:

```text
Fact
   │
   ▼
This happened.
```

Example:

```text
Order #123 was created.
```

Architecture:

```text
Producer
   │
   ▼
Event Stream
   │
   ├── Consumer A
   ├── Consumer B
   └── Consumer C
```

The primary concern is:

> What systems want to process this sequence of events?

And importantly:

```text
Consumer A can replay history.
Consumer B can process slowly.
Consumer C can start later.
```

---

# 13. Event Streaming vs Pub/Sub

The difference here is more subtle.

Both can have:

```text
One producer
      │
      ▼
Many consumers
```

The important distinction is often how events are treated.

A simple Pub/Sub model may conceptually be:

```text
Event happens
      │
      ▼
Deliver to subscribers
      │
      ▼
Move on
```

An event-streaming model is:

```text
Event happens
      │
      ▼
Append to durable sequence
      │
      ▼
Consumers read independently
      │
      ▼
History remains available
```

A useful mental model is:

```text
Pub/Sub
───────
"Something happened. Who should hear about it?"

Event Streaming
───────────────
"Here is the history of what happened. Read it at your own pace."
```

In real systems, the boundary can blur because technologies may support both patterns.

But conceptually, this distinction is useful.

---

# 14. Ordering

A stream naturally suggests order.

```text
[1] → [2] → [3] → [4]
```

But we need to be careful.

In a large distributed system, events may come from:

```text
Server A
Server B
Server C
Server D
```

Achieving one perfect global order across everything can be expensive and unnecessary.

Consider:

```text
User A liked a post.
```

and:

```text
User B placed an order.
```

Does it matter which happened first?

Probably not.

But for a single order:

```text
OrderCreated
      ↓
OrderPaid
      ↓
OrderShipped
```

Order may matter.

This leads to an important principle:

> **Do not pay for stronger ordering than the business problem requires.**

Later, when we discuss message ordering, we will explore this deeply.

For now:

```text
Global ordering
        ≠
Ordering where it actually matters
```

---

# 15. Scaling an Event Stream

Eventually, one stream processor may not be enough.

Suppose:

```text
1 million events/sec
```

arrive.

One machine cannot necessarily handle everything.

So conceptually, we divide the stream.

```text
Single Stream

[1][2][3][4][5][6][7][8]
```

becomes:

```text
Stream A

[1][4][7][10]

Stream B

[2][5][8][11]

Stream C

[3][6][9][12]
```

Now processing can happen in parallel.

```text
                    ┌──► Consumer Group A
Stream Partition A ─┤
                    └──► Workers

Stream Partition B ─┤
                    └──► Workers

Stream Partition C ─┤
                    └──► Workers
```

This introduces tradeoffs.

We gain:

```text
More throughput
```

but perfect global ordering becomes harder.

Again, this will connect directly to our later chapter on ordering.

---

# 16. Real-World Usage

Event streaming is useful whenever systems generate a continuous flow of meaningful events.

## E-Commerce

```text
OrderCreated
OrderPaid
ProductViewed
CartUpdated
ShipmentCreated
```

Consumers might include:

- Analytics
- Recommendations
- Fraud detection
- Inventory
- Data warehouses

---

## Ride Sharing

```text
RideRequested
DriverAssigned
DriverArrived
RideStarted
RideCompleted
```

Different systems may use the stream for:

```text
Billing
Analytics
Driver incentives
Fraud detection
Machine learning
```

---

## Video Platforms

```text
VideoUploaded
VideoProcessed
VideoViewed
VideoLiked
VideoShared
```

These can feed:

```text
Analytics
Recommendations
Creator dashboards
Moderation
Data pipelines
```

---

## Financial Systems

```text
PaymentInitiated
PaymentAuthorized
PaymentCompleted
PaymentFailed
```

Historical processing can be extremely valuable for:

- Reconciliation
- Auditing
- Fraud analysis
- Reporting

Although the exact reliability and retention requirements become much stricter in such systems.

---

# 17. Where Event Streaming Helps

## Continuous Data Flows

When events are continuously produced:

```text
Events → Events → Events → Events → ...
```

---

## Multiple Independent Consumers

Different consumers can process the same history independently.

---

## Replay

Old events can be processed again while they are retained.

---

## Late-Joining Consumers

A new consumer can start from historical events.

---

## Data Pipelines

Operational events can flow into:

```text
Applications
Analytics
Data Warehouses
Machine Learning
Monitoring
```

---

## Rebuilding State

If derived data is lost or corrupted, retained events may help rebuild it.

---

# 18. Where Event Streaming Doesn't Help

## Simple One-Time Tasks

If you simply need:

```text
Send this email.
```

an event stream may be unnecessary.

A work queue is often simpler.

---

## Immediate Request-Response

```text
User asks
    ↓
System decides
    ↓
User needs answer now
```

A stream does not replace synchronous communication.

---

## Small Systems With No Replay Requirement

If you have:

```text
One producer
One consumer
Simple workload
```

introducing durable event streams may add unnecessary complexity.

---

## When Event History Has No Value

If events do not need:

```text
Replay
Historical processing
Multiple consumers
Independent progress
```

then retaining them may not justify the operational and storage cost.

---

# 19. Mental Model

## A Queue Is a Restaurant Ticket System

```text
New order
    │
    ▼
Kitchen ticket
    │
    ▼
Chef processes it
    │
    ▼
Done
```

The primary question is:

> Who will do this work?

---

## An Event Stream Is a Book

Imagine a company writes every important event into a giant book.

```text
Page 1 → Order Created
Page 2 → Payment Completed
Page 3 → Order Shipped
Page 4 → User Registered
Page 5 → Order Created
...
```

Different readers can read the book independently.

```text
Reader A → Page 1000

Reader B → Page 750

Reader C → Starts from Page 1
```

Reader A finishing Page 1000 does not remove the previous pages.

A reader can even go back.

```text
Go back to Page 500.
```

That is the core mental model:

> **A queue is a waiting line for work. An event stream is a durable record of history.**

---

# 20. Tradeoffs

## Advantages

### Replayability

Consumers can reprocess historical events.

### Independent Consumers

Each consumer can move at its own pace.

### Late Joining

New consumers can process historical data.

### Decoupling

Producers do not need to know who consumes events.

### Useful for Data Pipelines

The same events can power many different systems.

### State Reconstruction

Derived state can potentially be rebuilt from retained events.

---

## Disadvantages

### More Storage

Retaining history costs money.

### More Complexity

Consumers must track progress.

### Event Schema Evolution

Events may need to remain understandable even as systems evolve.

### Ordering Complexity

Parallel processing and distributed producers complicate ordering.

### Duplicate Processing

Consumers may need to handle events more than once.

### Operational Complexity

Retention, scaling, recovery, and lag all become important concerns.

---

# 21. Common Interview Questions

## What is the difference between a message queue and event streaming?

A message queue primarily focuses on distributing work.

```text
Task
  ↓
Worker processes it
```

Event streaming focuses on storing and distributing a sequence of events.

```text
Event 1
  ↓
Event 2
  ↓
Event 3
  ↓
...
```

Consumers can independently track their position and replay history.

---

## What is replay?

Replay means processing events again from an earlier position.

For example:

```text
[1][2][3][4][5][6]
              ▲
              Consumer

Reset to 3

[1][2][3][4][5][6]
        ▲
        Consumer
```

The consumer processes events 3 onward again.

---

## Why do consumers need offsets or checkpoints?

The consumer needs to remember:

> How far have I processed?

For example:

```text
Consumer A → Event 1000
```

If it crashes, it can resume around that position rather than starting from the beginning.

---

## Can multiple consumers process the same event?

Yes.

That is one of the major benefits.

```text
Event
  │
  ├── Analytics
  ├── Fraud
  ├── Recommendations
  └── Data Warehouse
```

Each can process the event independently.

---

## Is event streaming always better than a message queue?

No.

Event streaming provides powerful capabilities, but with additional complexity.

Choose it when:

- History matters.
- Replay matters.
- Many consumers need independent progress.
- Events form an important continuous data flow.

Otherwise, a simpler queue may be better.

---

# 22. Before vs After Architecture

## Before: Event Delivered Once

```text
Order Service
      │
      ▼
┌──────────────┐
│   Messaging  │
└──────────────┘
      │
      ▼
Consumer
      │
      ▼
Process
      │
      ▼
Event gone
```

---

## After: Durable Event Stream

```text
Order Service
      │
      ▼
┌──────────────────────────────┐
│        EVENT STREAM          │
│                              │
│ [1][2][3][4][5][6][7][8]...  │
└──────────────────────────────┘
       │              │
       │              │
       ▼              ▼
 Consumer A       Consumer B

Position: 8      Position: 5
```

The events remain available according to the retention policy.

Consumers move independently.

---

# 23. Connections

So far, our story has evolved like this:

## First: Direct Communication

```text
Service A
   │
   ▼
Service B
```

Simple, but tightly coupled.

---

## Then: Asynchronous Communication

```text
Producer
   │
   ▼
Message
   │
   ▼
Consumer
```

Work no longer needs to happen immediately.

---

## Then: Message Queues

```text
Producer
   │
   ▼
Queue
   │
   ▼
One worker processes work
```

We gained buffering and independent scaling.

---

## Then: Publish / Subscribe

```text
Publisher
   │
   ▼
Topic
   │
   ├── Consumer A
   ├── Consumer B
   └── Consumer C
```

We gained fan-out.

---

## Now: Event Streaming

```text
Producer
   │
   ▼
[1] → [2] → [3] → [4] → [5] → ...
       │
       ├── Consumer A
       ├── Consumer B
       └── Consumer C
```

We gained:

- Durable history.
- Independent progress.
- Replay.
- Late consumers.

But now we have a new problem.

Suppose a producer sends a message.

```text
Producer
   │
   ▼
Message
```

What if the producer crashes before knowing whether the message was safely stored?

Or:

```text
Message delivered
      │
      ▼
Consumer processes it
      │
      ▼
Consumer crashes
```

Should the message be delivered again?

Could it be processed twice?

Could it be lost?

This brings us to one of the most important questions in distributed messaging:

> **What guarantee do we actually have that a message will be delivered and processed?**

That leads naturally to:

# Chapter 5 — Delivery Guarantees

---

# 24. Key Takeaways

- Event streaming treats events as a durable, ordered sequence rather than something that simply disappears after delivery.
- Producers append new events to the stream.
- Consumers track their own position in the stream.
- Different consumers can process the same stream independently and at different speeds.
- Retained events can be replayed.
- Replay helps with bug fixes, rebuilding state, and new consumers.
- Retention determines how long historical events remain available.
- Event streaming is especially useful for continuous data flows, analytics, data pipelines, and systems where history matters.
- Event streaming introduces additional complexity around storage, offsets, ordering, duplicates, and schema evolution.
- The next fundamental question is no longer how events flow.

It is:

> **What happens when something fails while they are flowing?**

That takes us to **Chapter 5 — Delivery Guarantees**.
