# Chapter 3 — Publish / Subscribe

## Goal

Understand why a message queue is not enough when **multiple independent systems need to react to the same event**, and how the Publish / Subscribe model allows producers to broadcast information without knowing who consumes it.

---

# 1. The Problem

In the previous chapter, we learned about message queues.

The basic model was:

```text
Producer
   │
   ▼
┌─────────────┐
│    Queue    │
└─────────────┘
   │
   ▼
Consumer
```

This works extremely well when the producer creates a unit of work that should be handled by **one worker**.

For example:

```text
Send this email.
```

```text
Resize this image.
```

```text
Generate this report.
```

```text
Process this uploaded file.
```

If we have multiple workers:

```text
                ┌──► Worker 1
                │
Queue ──────────┼──► Worker 2
                │
                └──► Worker 3
```

they compete for work.

One message might go to Worker 1.

Another to Worker 2.

Another to Worker 3.

The basic idea is:

> **One message should be processed by one member of the worker group.**

But now imagine a different situation.

A customer places an order.

```text
Order Service
      │
      ▼
OrderCreated
```

Now several completely independent systems care about this.

```text
OrderCreated
      │
      ├── Notification Service
      │
      ├── Analytics Service
      │
      ├── Loyalty Service
      │
      ├── Fraud Detection
      │
      └── Recommendation System
```

Every one of these systems may need to know that the order was created.

This is no longer:

> "Here is a task. Someone should do it."

Instead, it is:

> "This event happened. Anyone interested may react to it."

That is a fundamentally different communication model.

---

# 2. Why Existing Solutions Fail

Let's first try the obvious approach.

The Order Service directly calls everyone.

```text
                    ┌──► Notification
                    │
                    ├──► Analytics
                    │
Order Service ──────┼──► Loyalty
                    │
                    ├──► Fraud Detection
                    │
                    └──► Recommendations
```

This creates several problems.

The Order Service now knows:

- Every service that cares about an order.
- How to communicate with each one.
- When to call them.
- What happens when they fail.

Now imagine a new team builds:

```text
Customer Insights Service
```

It also wants to react to every new order.

What needs to happen?

The Order Service must be modified.

```text
Before

Order Service
   │
   ├── Notification
   ├── Analytics
   └── Loyalty
```

After:

```text
Order Service
   │
   ├── Notification
   ├── Analytics
   ├── Loyalty
   └── Customer Insights
```

The producer keeps growing more complicated every time a new consumer appears.

This is the opposite of decoupling.

---

# 3. What About Using One Message Queue?

Let's try this:

```text
Order Service
      │
      ▼
┌──────────────┐
│ Order Queue  │
└──────────────┘
      │
      ├── Notification
      ├── Analytics
      └── Loyalty
```

But remember what a normal competing-consumer model means.

Suppose:

```text
OrderCreated #123
```

enters the queue.

If consumers are competing for messages:

```text
OrderCreated #123
       │
       ▼
   Queue chooses
       │
       ▼
Analytics Service
```

Now Analytics processes it.

But what about Notification?

What about Loyalty?

They never received that particular message.

A work queue generally answers:

> **Who should perform this task?**

But our new problem is:

> **Who wants to know that this event happened?**

We need a different model.

---

# 4. The Big Idea

Instead of the producer sending a message to a specific consumer, it **publishes information to a shared channel**.

Consumers express interest in that information.

```text
                ┌──► Subscriber A
                │
                ├──► Subscriber B
Publisher ──► Topic
                │
                ├──► Subscriber C
                │
                └──► Subscriber D
```

The publisher does not need to know who receives it.

Its only responsibility is:

> **Publish the event.**

Consumers independently say:

> **I am interested in this type of event.**

This is the **Publish / Subscribe**, or **Pub/Sub**, model.

---

# 5. Publisher, Topic and Subscriber

There are three basic concepts.

## Publisher

The publisher produces information.

```text
Order Service
      │
      ▼
Publishes:

OrderCreated
```

The publisher does not need to know:

```text
Who consumes it?
How many consumers exist?
What they do with it?
```

---

## Topic

A topic is a logical channel for related events.

For example:

```text
orders
```

might contain:

```text
OrderCreated
OrderCancelled
OrderShipped
OrderDelivered
```

Conceptually:

```text
Publisher
    │
    ▼
┌─────────────────┐
│   orders topic  │
│                 │
│ OrderCreated    │
│ OrderShipped    │
└─────────────────┘
```

---

## Subscriber

A subscriber expresses interest in a topic or category of messages.

```text
orders
   │
   ├── Notification Service
   │
   ├── Analytics Service
   │
   ├── Loyalty Service
   │
   └── Fraud Detection
```

Each subscriber can independently react.

---

# 6. The Fundamental Difference

Let's compare the two models.

## Message Queue

```text
Producer
   │
   ▼
Queue
   │
   ▼
One worker handles the message
```

Conceptually:

> "Someone needs to perform this task."

---

## Publish / Subscribe

```text
Publisher
   │
   ▼
Topic
   │
   ├──► Subscriber A
   ├──► Subscriber B
   └──► Subscriber C
```

Conceptually:

> "This happened. Anyone interested can react."

This distinction is one of the most important in messaging systems.

---

# 7. One Event, Many Reactions

Let's return to our order.

The Order Service publishes:

```text
OrderCreated
```

Now:

```text
                       ┌──► Notification
                       │
                       ├──► Analytics
OrderCreated ──► Topic ─┼──► Loyalty
                       │
                       ├──► Fraud Detection
                       │
                       └──► Recommendations
```

The important thing is that the Order Service does not explicitly call any of these.

Tomorrow, another team creates:

```text
Accounting Service
```

It needs order events.

The architecture becomes:

```text
                       ┌──► Notification
                       ├──► Analytics
                       ├──► Loyalty
OrderCreated ──► Topic ─┼──► Fraud Detection
                       ├──► Recommendations
                       │
                       └──► Accounting
```

The Order Service does not change.

That is structural decoupling.

---

# 8. Before vs After Architecture

## Before: Direct Calls

```text
                         ┌──► Email Service
                         │
Order Service ───────────┼──► Analytics
                         │
                         ├──► Loyalty
                         │
                         └──► Fraud Detection
```

Problems:

- Order Service knows every consumer.
- Adding a consumer requires changing the producer.
- Consumer failures can affect the producer.
- The producer becomes increasingly complex.

---

## After: Pub/Sub

```text
Order Service
      │
      │ Publish
      ▼
┌───────────────────┐
│   Order Events    │
└───────────────────┘
      │
      ├────────► Notification
      │
      ├────────► Analytics
      │
      ├────────► Loyalty
      │
      └────────► Fraud Detection
```

Now the producer only knows:

> "I publish order events here."

Consumers decide independently what to do.

---

# 9. A Subscriber Is Not Necessarily One Server

Suppose the Notification Service receives a huge number of events.

One server may not be enough.

```text
Order Events
      │
      ▼
Notification Service
      │
      ▼
One Server
```

Eventually:

```text
One Server
    ✕
Too slow
```

So we add multiple instances.

```text
Order Events
      │
      ▼
Notification Processing
      │
      ├── Worker 1
      ├── Worker 2
      └── Worker 3
```

This introduces an important distinction.

We can have:

```text
One topic
   │
   ├── Notification subscriber group
   │       ├── Worker 1
   │       ├── Worker 2
   │       └── Worker 3
   │
   ├── Analytics subscriber group
   │
   └── Loyalty subscriber group
```

The Notification group collectively receives the relevant event.

Inside that group, multiple workers can share the processing work.

Conceptually:

```text
OrderCreated
      │
      ├────────► Notification Group
      │              │
      │              ├── Worker 1
      │              ├── Worker 2
      │              └── Worker 3
      │
      ├────────► Analytics Group
      │
      └────────► Loyalty Group
```

This gives us two useful properties at the same time:

1. Multiple independent systems receive the event.
2. Each system can scale its own processing independently.

---

# 10. Fan-Out

The process of one message being distributed to multiple consumers is often called **fan-out**.

```text
                  Subscriber A
                       ▲
                       │
                       │
Publisher ──► Topic ───┼──► Subscriber B
                       │
                       │
                       ▼
                  Subscriber C
```

One event:

```text
OrderCreated
```

fans out into several independent processing paths.

```text
OrderCreated
      │
      ├── Send confirmation email
      │
      ├── Update dashboard
      │
      ├── Award loyalty points
      │
      └── Run fraud checks
```

Each consumer is responsible for its own interpretation of the event.

---

# 11. Events vs Commands Revisited

Pub/Sub is especially natural for events.

Let's compare.

## Command

```text
SendEmail
```

This implies:

> A specific action should be performed.

Someone is expected to own that responsibility.

---

## Event

```text
OrderCreated
```

This says:

> This happened.

The publisher does not decide what everyone should do.

Consumers decide.

```text
OrderCreated
      │
      ├── Notification:
      │      "I'll send an email."
      │
      ├── Analytics:
      │      "I'll record this."
      │
      └── Loyalty:
             "I'll calculate points."
```

This is why Pub/Sub fits event-driven architecture so naturally.

---

# 12. The Power of Independent Evolution

Imagine version 1 of a company.

```text
OrderCreated
      │
      ├── Notification
      └── Analytics
```

Later, the company grows.

Version 2:

```text
OrderCreated
      │
      ├── Notification
      ├── Analytics
      ├── Loyalty
      └── Fraud Detection
```

Later:

```text
OrderCreated
      │
      ├── Notification
      ├── Analytics
      ├── Loyalty
      ├── Fraud Detection
      ├── Data Warehouse
      ├── Recommendations
      └── Accounting
```

The producer can remain conceptually unchanged.

It still does one thing:

```text
Publish OrderCreated
```

This is powerful in large organizations where teams build systems independently.

A team can introduce a new consumer without necessarily requiring changes to the producer.

---

# 13. Real-World Example

Imagine a video platform.

A creator uploads a video.

```text
Upload Service
      │
      ▼
VideoUploaded
```

Now multiple things may happen.

```text
                         ┌──► Video Transcoding
                         │
                         ├──► Thumbnail Generation
VideoUploaded ──► Topic ─┼──► Content Moderation
                         │
                         ├──► Search Indexing
                         │
                         ├──► Analytics
                         │
                         └──► Notifications
```

The Upload Service does not need to orchestrate all of this.

Its primary responsibility is:

```text
Receive video
      ↓
Store video
      ↓
Publish VideoUploaded
```

Everything else can react independently.

This allows the system to grow without turning the Upload Service into a giant coordinator.

---

# 14. Subscriber Failures

Suppose Analytics goes down.

```text
VideoUploaded
      │
      ├── Transcoding ✓
      │
      ├── Moderation ✓
      │
      ├── Analytics ✕
      │
      └── Search ✓
```

Should video uploading fail?

Usually, no.

One consumer's failure should ideally affect that consumer's processing path rather than every other consumer.

This is another important benefit of decoupling.

However, this raises a question:

> What happens to the event intended for the failed subscriber?

Should it:

```text
Wait?
Retry?
Disappear?
Be stored permanently?
```

The answer depends on the messaging model and reliability requirements.

We will gradually explore these questions.

---

# 15. Push vs Pull

There are two broad conceptual ways subscribers can receive information.

## Push

The messaging system actively sends messages to subscribers.

```text
Topic
   │
   ├──► Subscriber A
   ├──► Subscriber B
   └──► Subscriber C
```

The system says:

> "Here is a new message."

This can provide low latency.

But a new problem appears.

What if:

```text
Producer = Very Fast

Subscriber = Slow
```

The subscriber may become overwhelmed.

---

## Pull

The subscriber asks for messages when it is ready.

```text
Subscriber
     │
     │ "Give me more work."
     ▼
Topic / Messaging Layer
     │
     ▼
Messages
```

This gives the consumer more control over its processing rate.

Conceptually:

```text
Push
────
System controls delivery pace.

Pull
────
Consumer controls consumption pace.
```

Both models have tradeoffs.

---

# 16. Slow Subscribers

Imagine:

```text
Publisher → 100,000 events/sec
```

One subscriber can process:

```text
1,000 events/sec
```

What happens?

```text
Events
   │
   ▼
Subscriber
   │
   ▼
BACKLOG
```

Now we need to decide what the system should do.

Possible choices include:

```text
Store events temporarily
```

```text
Add more consumer instances
```

```text
Slow producers
```

```text
Drop some events
```

Different systems make different choices depending on the importance of the data.

For example:

A monitoring dashboard may tolerate dropping some updates.

A financial transaction system probably cannot.

This introduces a deeper question:

> How long should events be retained?

That question leads toward event streaming.

---

# 17. Where Publish / Subscribe Helps

## One Event, Many Independent Consumers

```text
OrderCreated
      │
      ├── Analytics
      ├── Notifications
      └── Loyalty
```

This is the classic use case.

---

## Decoupled Systems

The publisher does not need to know every consumer.

```text
Publisher
    │
    ▼
Topic
```

New consumers can appear independently.

---

## Event-Driven Architecture

Systems can react to facts that happened.

```text
UserRegistered
```

```text
PaymentCompleted
```

```text
TripFinished
```

```text
VideoUploaded
```

Each event can trigger multiple independent workflows.

---

## Independent Scaling

Different consumers can scale differently.

```text
Order Events
      │
      ├── Notifications → 5 workers
      │
      ├── Analytics → 50 workers
      │
      └── Fraud → 10 workers
```

Each workload can grow according to its own requirements.

---

# 18. Where Publish / Subscribe Doesn't Help

## When Exactly One Worker Should Handle a Task

For example:

```text
Process this payment.
```

You generally do not want:

```text
Worker A processes payment
Worker B processes payment
Worker C processes payment
```

A work-queue model is often a more natural fit.

---

## Immediate Request-Response

If a user asks:

```text
"Is this username available?"
```

the application needs an answer immediately.

Publishing an event and waiting for someone to eventually react may unnecessarily complicate the interaction.

---

## Strong Coordination

Imagine a workflow where several steps must succeed before proceeding.

```text
Step A
   ↓
Step B
   ↓
Step C
```

Simply broadcasting events may make the workflow harder to understand and coordinate.

Sometimes direct orchestration is clearer.

---

# 19. The New Problem Pub/Sub Creates

We have now achieved something powerful.

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

But consider this.

A subscriber goes offline.

```text
Topic
   │
   └── Subscriber B ✕
```

Events continue being published.

When Subscriber B returns, what should happen?

Should it receive:

```text
Only future events?
```

Or:

```text
Everything it missed?
```

Or perhaps:

```text
All events from the last 24 hours?
```

Now imagine a new service is created today.

It wants to analyze every order from the last year.

Can it replay old events?

A simple Pub/Sub model is often focused on:

> Deliver the event to interested consumers now.

But increasingly complex systems may need:

> Keep the sequence of events so consumers can process them at their own pace—and possibly replay them later.

That is the next evolution.

---

# 20. Mental Model

## A Newspaper vs a Restaurant Order

A message queue is like a restaurant order.

```text
"Prepare Burger #123"
```

One chef should prepare it.

```text
Order
   │
   ▼
One Worker
```

Pub/Sub is more like publishing a newspaper.

```text
Publisher
    │
    ▼
Today's Newspaper
    │
    ├── Subscriber A gets a copy
    ├── Subscriber B gets a copy
    └── Subscriber C gets a copy
```

The publisher does not need to personally contact everyone.

It publishes the information.

Interested readers receive it.

That is the mental model:

> **Queues distribute work. Pub/Sub distributes information.**

This is not a perfect rule in every implementation, but it is an excellent architectural starting point.

---

# 21. Tradeoffs

## Advantages

### Strong Decoupling

Publishers do not need to know their consumers.

### Easy Fan-Out

One event can trigger many independent actions.

### Extensibility

New consumers can be added without modifying the publisher.

### Independent Scaling

Each consumer can scale according to its own workload.

### Better Failure Isolation

One consumer's temporary failure does not necessarily need to stop the producer or other consumers.

---

## Disadvantages

### Harder to Trace

One event may trigger many downstream actions.

```text
OrderCreated
      │
      ├── A
      ├── B
      ├── C
      └── D
```

Understanding the complete system flow becomes more difficult.

### Eventual Consistency

Different consumers process the event at different times.

For a short period:

```text
Order exists ✓
Analytics updated ✓
Email sent ✕
Loyalty updated ⏳
```

Different parts of the system may temporarily disagree.

### Hidden Dependencies

The publisher may not know who depends on an event.

Changing or removing an event can unexpectedly affect downstream systems.

### Backlogs

Slow consumers can accumulate unprocessed events.

---

# 22. Common Interview Questions

## What is the difference between a message queue and Pub/Sub?

A useful conceptual distinction is:

```text
Message Queue
─────────────
One message
     ↓
One consumer processes it
```

```text
Pub/Sub
───────
One message
     ↓
Many independent subscribers can receive it
```

Queues distribute work.

Pub/Sub distributes information.

---

## Why is Pub/Sub useful in microservices?

It reduces direct dependencies.

A service can publish an event without knowing every service that needs to react to it.

This makes the architecture easier to extend as new consumers are added.

---

## What happens when a subscriber is down?

That depends on the messaging system and its delivery guarantees.

Possible models include:

- Missing events published while offline.
- Events waiting for the subscriber.
- Events being retained for later consumption.

The retention model becomes an important architectural decision.

---

## Can multiple instances of the same subscriber exist?

Yes.

Conceptually:

```text
Topic
   │
   ▼
Notification Consumer Group
   │
   ├── Instance 1
   ├── Instance 2
   └── Instance 3
```

The group processes the events while individual instances share the workload.

---

## Does Pub/Sub eliminate all coupling?

No.

It reduces direct knowledge between producers and consumers.

But systems remain coupled through:

- Event formats.
- Event meaning.
- Delivery guarantees.
- Ordering assumptions.
- Timing expectations.

The coupling has changed; it has not disappeared.

---

# 23. Connections

We started with direct synchronous communication.

```text
Service A
   │
   ▼
Service B
```

That created temporal and structural coupling.

Then we introduced asynchronous communication.

```text
Producer
   │
   ▼
Message
   │
   ▼
Consumer
```

Then message queues gave us:

```text
One task
   │
   ▼
One worker
```

But modern distributed systems often need:

```text
One event
   │
   ▼
Many independent reactions
```

So we introduced Pub/Sub.

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

But we have now reached another limitation.

What if events need to remain available?

What if a consumer is slow?

What if a consumer joins later?

What if we need to replay history?

What if the sequence of events itself is valuable?

Instead of thinking of messages as:

```text
Receive
   ↓
Process
   ↓
Discard
```

we may need to think of them as:

```text
Event 1
   ↓
Event 2
   ↓
Event 3
   ↓
Event 4
   ↓
...
```

A continuously growing sequence of events.

That leads naturally to the next chapter.

# Chapter 4 — Event Streaming

---

# 24. Key Takeaways

- Message queues are ideal for distributing units of work among workers.
- Pub/Sub is ideal when multiple independent systems need to react to the same event.
- A **publisher** produces information.
- A **topic** acts as a logical channel for related events.
- **Subscribers** independently consume events they care about.
- Pub/Sub enables **fan-out**: one event can trigger many workflows.
- New consumers can often be added without modifying the producer.
- Each consumer can scale independently.
- Pub/Sub introduces challenges around slow consumers, event retention, debugging, and hidden dependencies.
- A simple distinction to remember is:

```text
Message Queue → Who should do this work?

Pub/Sub       → Who wants to know this happened?
```

- The next question is what happens when events themselves become durable history that consumers can read at their own pace.

That takes us to **Event Streaming**.
