# Module 9 — Chapter 9: Amazon SQS

We've now seen two messaging models:

```text
RabbitMQ
    ↓
Message broker + queues + routing

Kafka
    ↓
Distributed event log + partitions + replay
```

SQS introduces a different architectural tradeoff:

> **What if I want reliable asynchronous messaging, but I don't want to build or operate the messaging infrastructure myself?**

That's the problem SQS solves.

---

# 1. Why Does SQS Exist?

Imagine you're building an application on AWS.

You have:

```text
API Service
    ↓
Background Job
    ↓
Worker
```

You don't want the API service to wait for the worker.

So you need:

```text
API
 ↓
Queue
 ↓
Worker
```

You could deploy and operate a messaging system yourself.

But now you have to worry about:

- brokers
- replication
- failover
- capacity
- scaling
- upgrades
- operational monitoring

For a simple work-queue problem, that can be unnecessary infrastructure.

SQS asks:

> **Why should your application team operate a message broker when all you really need is a reliable managed queue?**

---

# 2. The Big Idea

> **Amazon SQS is a fully managed message queue that lets producers and consumers communicate asynchronously without requiring you to operate the underlying messaging infrastructure.**

The important word here is:

**managed**.

Your architecture becomes:

```text
Producer
    │
    ▼
  SQS
    │
    ▼
Consumer
```

Instead of:

```text
Producer
    │
    ▼
RabbitMQ cluster
    │
    ▼
Consumer
```

where your organization is responsible for the broker infrastructure.

---

# 3. The Fundamental Model

SQS is fundamentally a **queue**.

A producer sends:

```text
Message A
Message B
Message C
```

into a queue:

```text
SQS Queue

A
B
C
```

Consumers retrieve messages:

```text
Queue
  │
  ▼
Consumer
```

After successfully processing a message, the consumer deletes it.

So the basic lifecycle is:

```text
Send
  ↓
Queue
  ↓
Receive
  ↓
Process
  ↓
Delete
```

That last step is important.

SQS doesn't assume:

> "Receiving means processing succeeded."

The consumer explicitly confirms successful processing by deleting the message.

---

# 4. Why Not Delete Immediately?

Suppose:

```text
Queue
  ↓
Consumer
  ↓
Receive message
  ↓
Processing...
  X
 crash
```

If SQS deleted the message as soon as it was received, the message would be lost.

Instead, SQS temporarily hides the message from other consumers.

This is called the:

> **visibility timeout**

Conceptually:

```text
Message
   ↓
Consumer receives it
   ↓
Message becomes invisible
   ↓
Consumer processes it
   ↓
Success → Delete
```

If the consumer crashes:

```text
Message
   ↓
Consumer
   X
 crash
   ↓
Visibility timeout expires
   ↓
Message becomes visible again
```

Another consumer can then process it.

---

# 5. Visibility Timeout

This is one of the most important SQS concepts.

Imagine:

```text
Visibility timeout = 60 seconds
```

Consumer receives:

```text
"GenerateInvoice"
```

For the next 60 seconds, that message is hidden from other consumers.

If processing succeeds:

```text
GenerateInvoice
      ↓
Success
      ↓
Delete
```

Done.

But if processing takes longer than expected:

```text
GenerateInvoice
      ↓
processing...
      ↓
60 seconds
      ↓
message visible again
```

Another consumer could potentially receive it.

Therefore:

> **Visibility timeout must be chosen with the expected processing time and failure behavior in mind.**

---

# 6. This Leads to the Same Lesson Again

Suppose:

```text
Consumer A
   ↓
processes payment
   ↓
success
   ↓
crashes before delete
```

SQS eventually makes the message visible again.

Then:

```text
Consumer B
   ↓
processes payment again
```

So once again:

```text
Message delivery
      ≠
Business operation exactly once
```

Your consumer should often be **idempotent**.

For example:

```text
paymentId = 12345
```

Before charging:

```text
Has payment 12345 already been processed?
```

If yes:

```text
Don't charge again.
```

This is the same distributed-systems lesson you saw with RabbitMQ and Kafka.

---

# 7. Standard Queues

SQS has a standard queue model optimized for:

- high throughput
- scalability
- availability

But there is an important tradeoff:

> **Messages may occasionally be delivered more than once, and strict ordering is not guaranteed.**

So the application should generally tolerate:

```text
M1
M2
M1 again
M3
```

This is why idempotency is so important.

The mental model is:

```text
Standard SQS
      ↓
Very scalable work queue
      ↓
At-least-once-oriented processing
      ↓
Design consumers to tolerate duplicates
```

---

# 8. FIFO Queues

SQS also provides FIFO queues.

FIFO stands for:

> **First In, First Out**

These queues are designed when ordering is important.

For example:

```text
OrderCreated
PaymentCompleted
OrderShipped
```

You may require:

```text
OrderCreated
      ↓
PaymentCompleted
      ↓
OrderShipped
```

rather than allowing them to be processed arbitrarily.

FIFO queues provide stronger ordering semantics and deduplication capabilities, but generally involve different throughput/scaling characteristics than standard queues.

So the choice is essentially:

```text
Standard
    ↓
Maximum scalability / throughput
    +
duplicate tolerance

FIFO
    ↓
Ordering / deduplication requirements
    +
different throughput constraints
```

---

# 9. SQS Doesn't Work Like Kafka

This is important.

With Kafka:

```text
Topic
  ↓
Events retained
  ↓
Consumers track offsets
```

With SQS:

```text
Queue
  ↓
Consumer receives message
  ↓
Consumer processes
  ↓
Consumer deletes message
```

Kafka asks:

> **"Where are you in the event stream?"**

SQS asks:

> **"Have you successfully processed this piece of work?"**

That's a useful distinction.

---

# 10. SQS Doesn't Work Like RabbitMQ Either

RabbitMQ:

```text
Producer
   ↓
Exchange
   ↓
Queue
   ↓
Consumer
```

The broker provides sophisticated routing capabilities.

SQS is much simpler:

```text
Producer
   ↓
Queue
   ↓
Consumer
```

You don't get RabbitMQ's exchange/routing model as the central abstraction.

That's intentional.

SQS is saying:

> **"You need a durable queue. We'll operate it for you."**

---

# 11. The Main Architectural Advantage

The biggest advantage isn't some magical messaging feature.

It's:

> **You don't operate the messaging infrastructure.**

Think about the difference.

### Self-managed broker

```text
Your team
   ↓
Deploy broker
   ↓
Configure cluster
   ↓
Handle failures
   ↓
Scale it
   ↓
Upgrade it
   ↓
Monitor it
```

### SQS

```text
Your team
   ↓
Create queue
   ↓
Send / receive messages
```

The infrastructure responsibility is pushed to AWS.

That's an architectural tradeoff, not merely a convenience.

---

# 12. Why This Matters in HLD

Suppose you're designing an AWS-native application.

Requirement:

> "Process image thumbnails asynchronously."

You need:

```text
Upload API
    ↓
Queue
    ↓
Thumbnail Workers
```

You could choose:

```text
RabbitMQ
```

or:

```text
Kafka
```

or:

```text
SQS
```

The question isn't:

> "Which technology is best?"

The question is:

> **"What properties does this workload actually require?"**

If it's simply:

```text
Submit work
   ↓
Process later
```

SQS can be an excellent fit.

---

# 13. SQS + Worker Scaling

Suppose:

```text
Incoming:
5,000 jobs/sec

Workers:
1,000 jobs/sec
```

The queue accumulates:

```text
SQS
 ├── Job
 ├── Job
 ├── Job
 ├── Job
 └── ...
```

You can increase worker capacity:

```text
             SQS
              │
      ┌───────┼───────┐
      ▼       ▼       ▼
   Worker   Worker   Worker
      │       │       │
      └───────┼───────┘
              ▼
          Processing
```

This is the same **competing consumers** pattern you learned earlier.

---

# 14. Queue Depth Becomes a Signal

Suppose your queue depth is:

```text
100
200
500
2,000
10,000
```

The backlog is growing.

That tells you:

> **Consumers aren't keeping up with producers.**

You can use queue metrics to trigger scaling.

Conceptually:

```text
Queue depth ↑
      ↓
More workers
      ↓
Processing capacity ↑
      ↓
Queue depth ↓
```

This is a very common architecture pattern.

---

# 15. Dead-Letter Queues

SQS also supports dead-letter queues.

Suppose:

```text
Message
   ↓
Consumer
   ↓
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

Eventually we don't want:

```text
retry forever
```

So after some configured number of failed receives:

```text
Main Queue
    ↓
max retries exceeded
    ↓
Dead-Letter Queue
```

Then engineers can inspect or separately process the failed message.

This is the same concept you learned with RabbitMQ.

---

# 16. Poison Messages

Consider:

```text
Message:
"Process customer = NULL"
```

Every worker receives it:

```text
Worker A → failure
Worker B → failure
Worker C → failure
```

If we retry indefinitely:

```text
Queue
  ↓
failure
  ↓
retry
  ↓
failure
  ↓
retry
  ↓
...
```

we waste resources.

A DLQ isolates the problematic message.

This is why DLQs are a reliability mechanism, not merely a debugging feature.

---

# 17. Long Polling

There's another practical SQS concept worth understanding.

Imagine a consumer repeatedly asking:

```text
"Any message?"
```

and the queue is empty.

If it does this continuously:

```text
poll
poll
poll
poll
poll
```

we waste requests.

SQS supports **long polling**, where the consumer can wait for messages to arrive rather than immediately returning an empty response.

Conceptually:

```text
Consumer
   │
   │ "Give me a message"
   ▼
SQS
   │
   │ waits briefly
   │
   ▼
Message arrives
   │
   ▼
Consumer
```

This reduces unnecessary empty polling and can improve efficiency.

You don't need to memorize configuration values for HLD.

Just understand:

> **Long polling reduces waste when queues are intermittently empty.**

---

# 18. SQS and Backpressure

Remember Module 8's rate limiting/backpressure concepts.

Suppose downstream processing can only handle:

```text
1,000 jobs/sec
```

but incoming traffic is:

```text
10,000 jobs/sec
```

Instead of allowing all 10,000 requests to immediately hit the workers:

```text
Producer
   ↓
SQS
   ↓
Workers
```

the queue absorbs the temporary burst.

This gives the system a form of **load leveling**.

Instead of:

```text
Traffic spike
    ↓
Workers overloaded
    ↓
Failure
```

we get:

```text
Traffic spike
    ↓
Queue grows
    ↓
Workers process steadily
```

Again:

> A queue buys you time. It doesn't eliminate capacity requirements.

---

# 19. Queue Backlog Is Not Automatically Healthy

Suppose the queue has:

```text
1 million messages
```

That doesn't necessarily mean the system is broken.

Maybe there's a temporary burst and workers will catch up.

But if:

```text
Queue depth
    ↑
    ↑
    ↑
continuously
```

then the system's processing capacity is insufficient.

So you should think about:

```text
Queue depth
+
oldest message age
+
consumer processing rate
```

rather than queue depth alone.

The **age of the oldest unprocessed message** can be particularly useful for understanding whether users are actually experiencing increasing delay.

---

# 20. A Complete Example

Let's design a report-generation system.

User requests:

```text
Generate my annual report
```

Generating the report takes:

```text
30 seconds
```

We don't want:

```text
HTTP Request
    ↓
30 seconds
    ↓
Response
```

Instead:

```text
Client
  ↓
API
  ↓
SQS
  ↓
Worker
  ↓
Generate Report
  ↓
Object Storage
```

The API can immediately respond:

```text
202 Accepted
jobId = 123
```

The worker processes asynchronously.

Later:

```text
GET /reports/123
```

can return:

```text
status = COMPLETED
url = ...
```

This is an extremely common use case for queues.

---

# 21. SQS + S3

Notice something interesting.

We've already learned:

```text
S3
```

is object storage.

So we can combine:

```text
Request
   ↓
SQS
   ↓
Worker
   ↓
S3
```

For example:

```text
User uploads video
      ↓
S3
      ↓
SQS message
      ↓
Video processing worker
      ↓
Processed video → S3
```

This is a classic cloud architecture pattern.

The queue carries **work metadata**.

S3 carries the **large object**.

You generally don't want to put a massive video itself inside a queue.

---

# 22. SQS Is Not an Event Store

This distinction is worth emphasizing.

If you need:

```text
Event history
Replay
Multiple independent consumers
Long-lived streams
High-throughput event processing
```

you should start thinking:

```text
Kafka
```

If you need:

```text
Work queue
Background processing
Retry
Decoupling
Managed infrastructure
```

you should consider:

```text
SQS
```

A useful first-pass decision tree:

```text
Do I need durable event history/replay?
          │
       Yes│
          ▼
       Kafka

          No
          │
          ▼
   Is this asynchronous work?
          │
         Yes
          │
          ▼
    Need managed queue?
          │
         Yes
          │
          ▼
         SQS
```

Of course, real architectures have more nuance.

---

# 23. SQS vs RabbitMQ

This is probably the most natural comparison.

|                             | SQS           | RabbitMQ                                           |
| --------------------------- | ------------- | -------------------------------------------------- |
| Fundamental model           | Managed queue | Message broker                                     |
| Infrastructure              | Fully managed | Typically you operate/manage broker infrastructure |
| Routing                     | Simpler       | Rich exchange/routing model                        |
| Work queues                 | Excellent     | Excellent                                          |
| Async processing            | Excellent     | Excellent                                          |
| Operational control         | Lower         | Higher                                             |
| AWS integration             | Excellent     | Possible, but less native                          |
| Infrastructure burden       | Low           | Higher                                             |
| Flexible messaging patterns | More limited  | Strong                                             |

The key tradeoff is:

```text
SQS
↓
Less control
+
Less operational burden

RabbitMQ
↓
More control
+
More operational responsibility
```

---

# 24. SQS vs Kafka

Now the three-way comparison becomes useful.

|                   | SQS                               | RabbitMQ                 | Kafka                             |
| ----------------- | --------------------------------- | ------------------------ | --------------------------------- |
| Core abstraction  | Queue                             | Broker + Queue           | Event log                         |
| Main use case     | Async work                        | Async work + routing     | Event streaming                   |
| Replay            | Not the central model             | Not the central model    | Core capability                   |
| Consumer model    | Work distribution                 | Work distribution        | Consumer groups + offsets         |
| Routing           | Simple                            | Rich                     | Topic/partition                   |
| Ordering          | Standard: limited; FIFO available | Queue-dependent          | Within partition                  |
| Managed option    | Yes                               | Requires more management | Managed options exist             |
| Best mental model | "Do this work later"              | "Route this work"        | "Record and stream what happened" |

A useful shorthand:

```text
SQS
"What work needs to be done?"

RabbitMQ
"Where should this message go?"

Kafka
"What happened, and who wants to consume the history?"
```

These aren't absolute definitions, but they're excellent interview mental models.

---

# 25. What Happens If SQS Goes Down?

Here's an important architectural consideration.

Because SQS is managed by AWS, you're not responsible for running the queue cluster yourself.

That significantly reduces operational burden.

But:

> **Managed does not mean magically immune to failure.**

Your application still needs to consider:

- retries
- timeouts
- duplicate processing
- downstream failures
- regional architecture
- queue backlog
- poison messages

The important difference is that you're not personally responsible for maintaining the underlying queue servers.

---

# 26. When Should I Choose SQS?

Strong signals:

### 1. Background jobs

```text
API → SQS → Worker
```

### 2. Traffic spikes

```text
Bursty producers
      ↓
SQS
      ↓
Steady workers
```

### 3. Decoupling services

```text
Service A
   ↓
SQS
   ↓
Service B
```

### 4. Retryable work

```text
Failure
   ↓
retry
```

### 5. AWS-native architecture

If your application is already heavily AWS-oriented and you don't need advanced broker capabilities, SQS can be an extremely natural choice.

---

# 27. When Would I Avoid SQS?

If your requirements are primarily:

```text
Long-lived event stream
+
Replay
+
Many independent consumers
+
Very high-throughput streaming
```

Kafka is likely a stronger candidate.

If you need:

```text
Complex routing
+
Exchange-based messaging
+
Fine-grained broker behavior
```

RabbitMQ may be more appropriate.

And if you don't actually need asynchronous processing:

```text
Service A
   ↓
Service B
   ↓
Response
```

then perhaps you shouldn't introduce a queue at all.

---

# 28. The Deeper Lesson

Notice what Module 9 is doing.

We're not memorizing:

> "SQS has feature X, Y, Z."

We're learning to map:

```text
Requirement
     ↓
Architectural property
     ↓
Technology
```

For example:

```text
"I need background processing"
        ↓
"Asynchronous work queue"
        ↓
SQS / RabbitMQ

"I need event replay"
        ↓
"Durable event log"
        ↓
Kafka

"I need complex routing"
        ↓
"Broker with routing abstraction"
        ↓
RabbitMQ
```

That's the skill you actually need in an SDE-II HLD interview.

---

# 29. SQS Mental Model

Keep this:

```text
                 SQS
                  │
      ┌───────────┴───────────┐
      │                       │
   Producer                Consumer
      │                       │
      ▼                       ▼
    Send                  Receive
                              │
                              ▼
                         Process
                              │
                    ┌─────────┴─────────┐
                    │                   │
                 Success             Failure
                    │                   │
                    ▼                   ▼
                  Delete          Retry / Redelivery
                                        │
                                        ▼
                                      DLQ
```

And the one sentence:

> **SQS is a managed asynchronous work queue: producers submit work, consumers process it, successful messages are deleted, and failed work can be retried or moved to a DLQ.**

---

# 30. The Messaging Trilogy

You now have three technologies that look superficially similar:

```text
                 Messaging
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
    RabbitMQ         Kafka         SQS
       │             │             │
       ▼             ▼             ▼
    Route &       Stream &       Managed
    process       replay         work queue
    work          events
```

And the architectural progression is:

```text
RabbitMQ
"How do I route and process messages?"

Kafka
"How do I retain and stream events?"

SQS
"How do I get a reliable queue without
operating the messaging infrastructure?"
```

That's the level at which I want you thinking about these technologies.

---

## Module 9 Progress

We've now covered:

```text
1. Redis
2. MySQL
3. PostgreSQL
4. MongoDB
5. Cassandra
6. DynamoDB
7. Elasticsearch
8. RabbitMQ
9. Kafka
10. SQS  ← current
```

Next in the planned sequence is:

# Amazon S3

And the question changes again:

> **We've been storing structured data and moving messages around. What if the thing we need to store is a 5 GB video, millions of images, backups, documents, or arbitrary files?**

That leads naturally to **object storage**.
