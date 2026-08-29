# Module 8 — Chapter 7: Leader Election

## 1. Goal

**Choose one node in a distributed system to act as the leader, while ensuring that the system can recover and choose a new leader if the current one fails.**

---

# 2. The Problem

Imagine we have a system with several application servers:

```text
       +---------+
       | Server A|
       +---------+
       +---------+
       | Server B|
       +---------+
       +---------+
       | Server C|
       +---------+
       +---------+
       | Server D|
       +---------+
```

They're all capable of performing the same work.

But there is one particular responsibility that we **don't** want all of them performing simultaneously.

For example:

> "Only one server should periodically clean up expired data."

If every server does it:

```text
A → cleanup
B → cleanup
C → cleanup
D → cleanup
```

We may:

- duplicate expensive work
- create conflicts
- waste resources
- potentially corrupt state

We therefore want:

```text
             Leader
               |
               v
        Performs coordination
               |
       +-------+-------+
       |       |       |
      S2      S3      S4
```

Only one node has the special responsibility.

But now comes the difficult part.

What if the leader crashes?

```text
Leader A
   |
   X
 crash
```

We need another node to take over:

```text
Before:

A = Leader
B = Follower
C = Follower


After A crashes:

B = Leader
C = Follower
```

That's **leader election**.

---

# 3. Why Existing Solutions Fail

We just learned distributed locking.

Couldn't we simply say:

> "The server holding a distributed lock is the leader."

Sometimes, yes.

In fact, leadership is often implemented using some form of lease or lock.

But leader election has a broader responsibility.

A lock answers:

> **"Who currently owns this particular resource?"**

Leader election answers:

> **"Which node should be the authoritative coordinator for this group?"**

The distinction becomes important because leadership often lasts for a period of time and involves:

- detecting failure
- selecting a replacement
- maintaining membership
- preventing multiple leaders
- coordinating followers

So leader election is fundamentally about **cluster-wide coordination**.

---

# 4. The Big Idea

> **Leader election allows a group of distributed nodes to agree on one node that temporarily has special authority.**

Conceptually:

```text
        Cluster
     /     |     \
    A      B      C
     \     |     /
       Election
          |
          v
      Leader = B
```

If B fails:

```text
      B crashes
          |
          v
     New election
          |
          v
      Leader = C
```

The important word is **agree**.

It's not enough for Server B to say:

> "I'm the leader."

The rest of the cluster needs to agree that B is the leader.

---

# 5. Why Do We Need a Leader?

At first glance, leadership seems contradictory to distributed systems.

We created multiple servers to avoid having a single point of failure.

So why deliberately designate one as special?

Because sometimes **coordination is easier if one node has authority**.

Consider:

```text
100 servers
```

All 100 independently deciding:

```text
Should I perform this action?
```

can create conflicts.

Instead:

```text
               Leader
             /   |   \
            /    |    \
          S2     S3    S4
```

The leader can coordinate:

```text
Leader
  |
  +-- assign work to S2
  +-- assign work to S3
  +-- assign work to S4
```

We trade some decentralization for simpler coordination.

---

# 6. Example: Scheduled Work

Suppose we have:

```text
10 application instances
```

and every five minutes we need to:

```text
delete expired sessions
```

Without leader election:

```text
A → cleanup
B → cleanup
C → cleanup
D → cleanup
...
```

With a leader:

```text
             Leader A
                |
                v
          Cleanup job
```

The other nodes remain available to take over if A fails.

---

# 7. Example: Cluster Coordinator

Suppose we have a distributed worker system:

```text
              Leader
                |
        +-------+-------+
        |       |       |
       W1      W2      W3
```

The leader might decide:

```text
Job 1 → W1
Job 2 → W2
Job 3 → W3
```

Now imagine the leader dies.

Without a replacement:

```text
Workers
   |
   X
No coordinator
```

With leader election:

```text
Leader A
   |
  crash
   |
   v
Election
   |
   v
Leader B
```

The cluster can continue.

---

# 8. What Does Leader Election Actually Require?

There are several pieces.

## 1. Membership

Nodes need some understanding of:

```text
Who are the participants?
```

For example:

```text
A
B
C
D
```

---

## 2. Failure Detection

Nodes need some way to determine:

> "Is the current leader still alive?"

This is harder than it sounds.

Suppose B doesn't respond.

Does that mean:

```text
B crashed?
```

Maybe.

But perhaps:

```text
network problem
```

Or:

```text
B is overloaded
```

Or:

```text
our own network connection is broken
```

Distributed systems cannot directly observe another machine's internal state.

They infer failure from communication.

---

## 3. Election

If the leader appears unavailable:

```text
Leader B
    |
    X
```

the remaining nodes need to select a replacement.

---

## 4. Agreement

The nodes need to converge on:

```text
Leader = C
```

rather than:

```text
A thinks B is leader
C thinks D is leader
```

---

# 9. The Split-Brain Problem

This is one of the most important concepts.

Imagine:

```text
        A
       / \
      B   C
```

Suppose A is leader.

Then a network problem occurs.

```text
        A

       X X X

      B   C
```

A cannot communicate with B and C.

From A's perspective:

```text
"B and C aren't responding."
```

From B/C's perspective:

```text
"A isn't responding."
```

Now B and C might elect B as the new leader.

But A doesn't know this.

A might continue operating as leader.

We now have:

```text
Leader A       Leader B
   |               |
   v               v
Group 1          Group 2
```

That's **split brain**.

Two nodes believe they have authority simultaneously.

This can be disastrous if both perform conflicting operations.

---

# 10. Why Failure Detection Is Not Perfect

This is a foundational distributed-systems idea:

> **You generally cannot distinguish "the other server crashed" from "I cannot currently communicate with the other server."**

Consider:

```text
A → B
```

A sends a message.

No response.

Possible explanations:

```text
1. B crashed
2. Network packet was lost
3. Network is partitioned
4. B is extremely slow
5. B is overloaded
6. Response was delayed
```

A doesn't know which one happened.

Therefore leader election cannot simply mean:

```text
"No response → definitely dead."
```

It requires carefully designed coordination rules.

---

# 11. Terms You Should Know

### Leader

The node currently holding leadership.

### Follower

A node that accepts the leader's authority and is ready to take over if necessary.

### Election

The process of choosing a new leader.

### Term / Epoch

A monotonically increasing identifier representing a leadership generation.

Conceptually:

```text
Term 1 → Leader A
Term 2 → Leader B
Term 3 → Leader C
```

Why is this useful?

Because it allows the system to distinguish **old leadership** from **current leadership**.

---

# 12. Why Terms Matter

Imagine:

```text
Term 5
Leader A
```

A network problem occurs.

A new election happens:

```text
Term 6
Leader B
```

But A is still alive and doesn't know about the election.

A sends:

```text
"I am leader."
```

with:

```text
term = 5
```

B's system knows:

```text
current term = 6
```

Therefore:

```text
5 < 6
```

A's leadership is stale.

The cluster can reject A's action.

This is conceptually similar to the fencing tokens we discussed in distributed locking.

---

# 13. Leadership Is Temporary

A leader generally shouldn't be thought of as:

> "Server B is permanently the leader."

Instead:

```text
Leader B
   |
   | leadership valid
   |
   v
period of time
```

Leadership can expire or be revoked.

If B fails:

```text
B
|
X
|
v
new election
```

This is why leader election is closely related to **leases, heartbeats, and failure detection**.

---

# 14. Heartbeats

A common conceptual mechanism is a heartbeat.

The leader periodically sends:

```text
"I'm alive."
```

For example:

```text
Leader
  |
  | heartbeat
  v
Followers

Leader
  |
  | heartbeat
  v
Followers
```

If heartbeats stop:

```text
heartbeat
heartbeat
heartbeat
...
(no heartbeat)
```

followers may begin an election.

But remember:

> Missing heartbeats don't prove the leader crashed.

They only indicate that communication has failed long enough to trigger the system's failure-detection policy.

---

# 15. Election Example

Let's simplify the process.

We have:

```text
A
B
C
```

Initially:

```text
Leader = A
Term = 1
```

A fails.

```text
A → X
```

B and C detect that the leader is unavailable.

They start an election.

Suppose B wins.

Now:

```text
Leader = B
Term = 2
```

C accepts B's leadership.

```text
A → old leader
B → current leader
C → follower
```

The cluster continues.

---

# 16. How Does a Node "Win"?

There are multiple algorithms for leader election.

At the conceptual level, the system needs some rule such as:

- majority vote
- priority
- deterministic ordering
- consensus protocol

For example:

```text
A
B
C
```

could vote.

```text
B → vote
C → vote
```

B gets:

```text
2 / 3
```

and becomes leader.

The critical idea isn't the specific algorithm yet.

It's:

> **Leadership should be established through a protocol that gives the cluster a consistent view of who has authority.**

---

# 17. Why Majority Matters

Suppose we have:

```text
5 nodes
```

A majority is:

```text
3
```

Why?

Because two different groups cannot both contain three nodes without overlapping.

For example:

```text
Group 1 = A B C
Group 2 = C D E
```

They overlap at C.

This property helps prevent two independent majorities from electing conflicting leaders simultaneously.

Compare that with:

```text
Group 1 = A B
Group 2 = C D
```

Neither has a majority.

Therefore:

> **When the cluster cannot establish a majority, it should generally stop making authoritative decisions rather than risk having two independent leaders.**

This is a crucial distributed-systems tradeoff.

---

# 18. Network Partition

Let's make that concrete.

Five nodes:

```text
A B C D E
```

Leader = A.

Network partition:

```text
A B C     |     D E
```

The left side has:

```text
3 nodes
```

The right side has:

```text
2 nodes
```

Only the majority side can safely establish new leadership.

```text
A B C
```

can continue coordinating.

```text
D E
```

cannot safely declare a new leader because they don't have a majority.

This sacrifices availability on the minority side to preserve consistency.

And this connects directly back to **CAP theorem**.

---

# 19. Leader Election and CAP

Suppose we have:

```text
5-node cluster
```

and the network partitions.

If both sides continue making authoritative decisions:

```text
A B C → leader A

D E → leader D
```

we risk conflicting state.

Instead, we can require:

```text
majority → allowed to elect
minority → cannot elect
```

This means the minority partition may become unavailable for certain operations.

We're choosing:

> **Consistency over availability during the partition.**

This is one reason many coordination systems rely on quorum/majority-based approaches.

---

# 20. Leader Election vs Distributed Locking

Let's make the distinction very clear.

### Distributed lock

```text
Resource X
    |
    v
Who owns X right now?
```

Example:

```text
Inventory:123
```

Server A owns it.

---

### Leader election

```text
Cluster
   |
   v
Who is the coordinator right now?
```

Example:

```text
Cluster of workers
       |
       v
Leader = Server A
```

A leader may then coordinate **many resources or operations**.

---

# 21. Leader Election vs Load Balancing

Another distinction.

A load balancer says:

```text
Request
   |
   +----→ A
   +----→ B
   +----→ C
```

It distributes work.

Leader election intentionally says:

```text
Special responsibility
          |
          v
       Leader A
```

Only one node has that responsibility.

So:

> **Load balancing spreads work; leader election concentrates authority.**

---

# 22. Where It Helps

Leader election is useful when:

### 1. Exactly one coordinator is desirable

```text
Cluster
   ↓
one coordinator
```

### 2. Scheduled jobs should have a single executor

```text
10 servers
   ↓
one executes
```

### 3. Work needs centralized assignment

```text
Leader
  |
  +→ Worker A
  +→ Worker B
  +→ Worker C
```

### 4. Cluster state needs an authority

Certain distributed systems use a leader to coordinate updates or manage metadata.

### 5. Failover is required

```text
Leader A
   ↓
failure
   ↓
Leader B
```

---

# 23. Where It Doesn't Help

Leader election isn't automatically beneficial.

### It can create a bottleneck

If every decision must pass through:

```text
Leader
```

the leader may become overloaded.

### Leader failure causes disruption

Even if only briefly:

```text
Leader failure
      ↓
Election
      ↓
New leader
```

There can be a period where the cluster can't safely perform leader-dependent operations.

### It doesn't remove distributed failures

Network partitions still exist.

Messages can still be delayed or lost.

Nodes can still disagree temporarily.

### It may be unnecessary

If work can be processed independently:

```text
A → task 1
B → task 2
C → task 3
```

without coordination, don't introduce a leader simply because you can.

---

# 24. Tradeoffs

## Advantages

### Simpler coordination

One node has clear authority.

### Efficient decision-making

Instead of all nodes coordinating every decision:

```text
Leader → makes decision
```

### Automatic failover

A new leader can take over.

### Useful for centralized scheduling

Only one node performs singleton tasks.

---

## Disadvantages

### Election complexity

Choosing a leader safely in a distributed system is difficult.

### Temporary unavailability

During an election:

```text
No confirmed leader
```

so some operations may need to pause.

### Potential bottleneck

The leader may receive too much work.

### Split-brain risk

Incorrect failure detection can result in multiple nodes believing they're leaders.

### Dependency on coordination

If the coordination mechanism fails, leadership may be unavailable.

---

# 25. Common Interview Questions

## Q1. Why do we need leader election?

When a distributed cluster has a responsibility that should be performed or coordinated by exactly one node, while still allowing another node to take over after failure.

---

## Q2. What happens when the leader crashes?

Conceptually:

```text
Leader fails
    ↓
Failure detected
    ↓
Election begins
    ↓
Nodes vote/coordinate
    ↓
New leader selected
    ↓
Cluster resumes
```

The exact timing and guarantees depend on the election protocol.

---

## Q3. What is split brain?

When two or more nodes independently believe they are the leader.

```text
Leader A
    +
Leader B
```

This can cause conflicting decisions or duplicate work.

---

## Q4. How do we prevent split brain?

Common principles include:

- majority/quorum-based elections
- terms/epochs
- leases
- fencing
- rejecting stale leadership
- requiring authoritative coordination

The exact mechanism depends on the protocol.

---

## Q5. Why can't a node simply declare itself leader after a timeout?

Because:

```text
"I can't reach the leader"
```

doesn't necessarily mean:

```text
"The leader is dead."
```

It could be a network partition.

If both sides self-elect, we get split brain.

---

## Q6. Why use a majority?

A majority ensures that two independent groups cannot both simultaneously have a majority.

That gives us a stronger basis for agreement.

---

## Q7. What happens if there's no majority?

A safe design generally refuses to establish new authoritative leadership.

For example:

```text
5 nodes

Partition:

3 nodes → majority → can elect
2 nodes → minority → cannot elect
```

The minority sacrifices availability to avoid conflicting authority.

---

# 26. Before vs After Architecture

### Without leadership

```text
                 Cluster
       +----------+----------+
       |          |          |
      S1         S2         S3
       |          |          |
       +----------+----------+
              Shared Work

Everyone can make decisions.
```

This can lead to:

```text
S1 → do work
S2 → do same work
S3 → do same work
```

---

### With leader election

```text
                 Cluster
       +----------+----------+
       |          |          |
    Leader      S2          S3
       |
       +---- coordinates work
```

If leader fails:

```text
                 Cluster

                  S2
                 /  \
                /    \
              S3      S4

            Election
               ↓
           New Leader
```

The authority moves rather than disappearing permanently.

---

# 27. The Deeper Connection

Now look at the last two chapters together:

```text
Distributed Locking
        ↓
"Who owns this resource?"

Leader Election
        ↓
"Who has authority over this cluster?"
```

And look at the broader Module 8 progression:

```text
Timeout
    ↓
Control waiting

Retry + Backoff
    ↓
Recover from transient failure

Circuit Breaker
    ↓
Stop unhealthy communication

Bulkhead
    ↓
Isolate resources

Rate Limiting
    ↓
Control traffic

Distributed Locking
    ↓
Coordinate access

Leader Election
    ↓
Coordinate authority
```

There's a very deliberate progression here.

We're moving from **protecting individual requests** toward **coordinating entire distributed clusters**.

---

# 28. Connections

The next chapter is:

## Chapter 8 — Saga Pattern

And this introduces a completely different distributed problem.

Suppose one business operation touches several services:

```text
Order Service
      ↓
Payment Service
      ↓
Inventory Service
      ↓
Shipping Service
```

What happens if:

```text
Order     ✓
Payment   ✓
Inventory ✓
Shipping  ✗
```

We don't have one database transaction spanning all of these services.

So how do we undo the work that already happened?

That's the problem the **Saga Pattern** addresses.

And this will connect directly to the following chapter:

```text
Saga Pattern
     ↓
Distributed Transactions
     ↓
2PC vs Saga
```

So we're about to move from **coordination of nodes** to **coordination of multi-step business transactions**.

---

# 29. Key Takeaways

1. **Leader election chooses one node to have temporary authority in a distributed cluster.**
2. Leadership is useful when a responsibility should have exactly one coordinator.
3. The leader must be replaceable when it fails.
4. Failure detection is imperfect because communication failure doesn't necessarily mean node failure.
5. **Split brain** occurs when multiple nodes believe they are leader.
6. **Majority/quorum** mechanisms help prevent conflicting leadership.
7. **Terms/epochs** help distinguish current leadership from stale leadership.
8. Leadership is generally temporary and may be represented by a lease.
9. Leader election and distributed locking are related but solve different problems.
10. During a partition, a system may deliberately sacrifice availability in the minority partition to preserve consistency.

### One sentence to remember

> **Leader election is the process of making a distributed cluster agree on who is allowed to make certain decisions—and making that authority transferable when the current leader fails.**
