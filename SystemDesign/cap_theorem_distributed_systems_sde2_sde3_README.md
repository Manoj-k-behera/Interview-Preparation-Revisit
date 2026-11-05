# CAP Theorem & Distributed-System Consistency — SDE-2/SDE-3 Interview Master Notes

> **Goal:** Master CAP theorem well enough to make architecture decisions in Tier-1 SDE-2/SDE-3 interviews.
>
> The objective is **not** to memorize "CAP = choose any 2".
>
> You should be able to look at a distributed-system requirement and explain:
>
> **What happens during a network partition? What consistency guarantee does the business need? What availability can we sacrifice? Which database/architecture fits? What are the trade-offs and failure modes?**

---

# 1. The Core Mental Model

CAP is about a **distributed system** where multiple nodes communicate over a network.

Imagine:

```text
                    Network
              ─────────────────
                 /          \
                /            \
               ▼              ▼
          ┌─────────┐    ┌─────────┐
          │ Node A  │    │ Node B  │
          │ DB      │    │ DB      │
          └─────────┘    └─────────┘
```

Both nodes may contain copies of the same logical data.

Now imagine the network breaks:

```text
          ┌─────────┐       X       ┌─────────┐
          │ Node A  │   PARTITION    │ Node B  │
          │ DB      │                │ DB      │
          └─────────┘                └─────────┘
```

Node A cannot communicate with Node B.

At this point, the system faces a fundamental choice:

```text
              NETWORK PARTITION
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
   Continue serving       Stop/reject some
   requests               requests
          │                     │
          ▼                     ▼
     Availability            Consistency
```

That is the heart of CAP.

---

# 2. What Does CAP Mean?

## C — Consistency

In CAP, **Consistency specifically means that every successful read sees the latest successful write according to a single, linearizable ordering**.

Simplified mental model:

```text
Write:
x = 10
   │
   ▼
Read from ANY node
   │
   ▼
x = 10
```

If a write has successfully completed, a subsequent read should not return the older value.

### Important interview clarification

CAP's "C" is **not simply database consistency in the broad sense**.

It is closer to:

> **Linearizability / single-copy semantics**

Do not confuse CAP consistency with:

- ACID consistency
- Eventual consistency
- Application-level validation rules
- Referential integrity

---

# 3. A — Availability

Availability means:

> Every request to a non-failing node receives a non-error response.

Simplified:

```text
Client
   │
   ▼
Node A
   │
   X  Node B unreachable
   │
   ▼
Still respond
```

An AP-oriented system may continue accepting requests even when some replicas cannot communicate.

But this may mean that different replicas temporarily disagree.

---

# 4. P — Partition Tolerance

A partition occurs when communication between nodes is disrupted.

```text
Node A  ───────── X ───────── Node B
                 Network
                partition
```

Partition tolerance means:

> The system continues operating despite arbitrary network communication failures between nodes.

### Why P is generally unavoidable

In a distributed system, network failures can happen.

Therefore, the practical CAP question is usually:

> **When a partition happens, do we prefer consistency or availability?**

Not:

> "Should we choose P?"

---

# 5. CAP Visualized

```text
                 CAP
                  ▲
                 / \
                /   \
               /     \
      Consistency     Availability
               \     /
                \   /
                 \ /
          Partition Tolerance
```

But avoid saying:

> "You can always choose any two."

A better statement is:

> **When a network partition occurs, a distributed system cannot simultaneously guarantee both strong consistency and availability.**

---

# 6. The Most Important CAP Diagram

Suppose:

```text
              ┌──────────────┐
              │   Client     │
              └──────┬───────┘
                     │
                ┌────┴────┐
                ▼         ▼
           ┌────────┐ ┌────────┐
           │ Node A │ │ Node B │
           │ x = 10 │ │ x = 10 │
           └────────┘ └────────┘
```

Client writes:

```text
x = 20
```

But the network partition occurs:

```text
           ┌────────┐      X      ┌────────┐
           │ Node A │             │ Node B │
           │ x = 20 │             │ x = 10 │
           └────────┘             └────────┘
```

Now a client reads from Node B.

It could receive:

```text
x = 10
```

If Node B continues responding, the system preserves **availability** but violates strong consistency.

Alternatively:

```text
Node B
  │
  └── "I cannot guarantee the latest value"
             │
             ▼
          Reject
```

Now the system sacrifices availability to preserve consistency.

---

# 7. CP vs AP

## CP — Consistency + Partition Tolerance

During a partition:

```text
Partition
    │
    ▼
Cannot safely coordinate
    │
    ▼
Reject / block some requests
    │
    ▼
Preserve strong consistency
```

Characteristics:

- Stronger consistency
- Reduced availability during partitions
- Usually quorum/leader coordination
- Good for correctness-critical data

Examples of workloads:

- Bank balance
- Inventory reservation
- Distributed locks
- Metadata
- Leader election
- Unique resource allocation

---

## AP — Availability + Partition Tolerance

During a partition:

```text
Partition
    │
    ▼
Continue serving requests
    │
    ▼
Replicas may temporarily diverge
    │
    ▼
Reconcile later
```

Characteristics:

- High availability
- Eventual convergence
- Conflict resolution may be required
- Good for workloads where temporary inconsistency is acceptable

Examples:

- Social feeds
- Likes/views counters
- Product recommendations
- Some user preferences
- Presence/status systems
- Shopping cart designs where availability is prioritized

---

# 8. CP vs AP Example — Inventory

Suppose:

```text
Product: iPhone
Inventory = 1
```

Two users attempt to buy it simultaneously.

```text
User A ──► Node A
User B ──► Node B
```

If Node A and Node B cannot communicate because of a partition:

### AP approach

Both may see:

```text
Inventory = 1
```

Both accept the purchase.

Result:

```text
Inventory = -1
```

or two successful orders for one item.

That may be unacceptable.

### CP approach

One side may reject/block the transaction:

```text
User A → SUCCESS
User B → RETRY / FAILURE
```

This sacrifices availability but protects correctness.

---

# 9. CP vs AP Example — Social Likes

Suppose a post has:

```text
Likes = 10,000
```

During a network partition:

```text
Node A → 10,001
Node B → 10,003
```

Temporary divergence may be acceptable.

Later:

```text
10,001
10,003
   │
   ▼
Reconciliation
   │
   ▼
10,004
```

For this type of workload, availability may matter more than immediate global consistency.

---

# 10. CAP Is About Partitions

This is one of the most important interview points.

CAP does **not** say:

> "Your system is always either consistent or available."

The trade-off becomes fundamental **when a partition prevents nodes from communicating**.

Without a partition:

```text
Node A ◄────────────► Node B
```

The system can coordinate.

During a partition:

```text
Node A ◄────── X ──────► Node B
```

The system cannot guarantee both:

```text
Strong consistency
        +
Availability
```

at the same time.

---

# 11. The Classic CAP Timeline

```text
Time ──────────────────────────────────────►

Node A:  x=10 ── write x=20 ── x=20 ─────────
                         │
                         │
Network: ────────────────X────────────────────
                         │
Node B:  x=10 ─────────────── read ── x=10 ───
```

If Node B returns `x=10`:

```text
Availability ✓
Strong consistency ✗
```

If Node B refuses the read:

```text
Strong consistency ✓
Availability ✗
```

---

# 12. Why "CA" Is Usually Misleading

You may see the CAP triangle labeled:

```text
      C
     / \
    /   \
   A --- P
```

and hear:

> "A database can be CA."

This needs qualification.

A **single-node database** can provide consistency and availability because there is no distributed network partition between replicas.

But once you have a genuinely distributed replicated system, network partitions are possible.

Therefore:

> **A distributed system cannot guarantee C + A in the presence of P.**

So in practical distributed-system design, we usually reason about:

```text
CP vs AP
```

---

# 13. CAP Consistency vs ACID Consistency

These are different concepts.

## CAP Consistency

Concerned with:

```text
Distributed replicas
        │
        ▼
What does a read observe?
```

Think:

> "Do all clients observe one consistent, linearizable view?"

---

## ACID Consistency

Concerned with:

```text
Database transaction
        │
        ▼
Valid state → Valid state
```

For example:

```text
Account A: -₹100
Account B: +₹100
```

A transaction must preserve application/database invariants.

### Interview answer

> "CAP consistency and ACID consistency are different concepts. CAP consistency is primarily about the visibility/order of operations across distributed replicas, while ACID consistency is about preserving database/application invariants across transactions."

---

# 14. Strong Consistency

A strong consistency model gives clients a view that behaves close to a single authoritative copy.

Example:

```text
Write x = 100
       │
       ▼
Read x
       │
       ▼
100
```

Common mechanisms:

- Leader-based replication
- Quorum protocols
- Consensus
- Synchronous replication
- Linearizable reads/writes

### Advantages

- Easier application reasoning
- Fewer stale reads
- Easier correctness guarantees

### Disadvantages

- Higher latency
- Lower availability during partitions
- Cross-region coordination can be expensive

---

# 15. Eventual Consistency

Eventual consistency allows replicas to temporarily disagree.

But if updates stop and communication resumes:

```text
Replica A ── x=20
Replica B ── x=10

        │
        ▼
Replication / reconciliation

        │
        ▼

Replica A ── x=20
Replica B ── x=20
```

The key property:

> **Eventually, replicas converge to the same state, assuming updates stop and the system can communicate.**

---

# 16. Strong vs Eventual Consistency

| Feature | Strong | Eventual |
|---|---|---|
| Stale reads | Usually no | Possible |
| Latency | Higher | Often lower |
| Availability | Lower during partition | Higher |
| Cross-region | Expensive | Easier |
| Application complexity | Lower | Higher |
| Conflict handling | Usually simpler | May be required |
| Good for | Money/inventory | Feeds/counters |

---

# 17. Quorum

A common way to reason about replicated systems is quorum.

Suppose:

```text
N = 3 replicas
```

For example:

```text
        ┌─────────┐
        │ Replica │
        │    A    │
        └─────────┘
             │
        ┌────┴────┐
        ▼         ▼
   Replica B   Replica C
```

A quorum could require:

```text
2 out of 3
```

For writes:

```text
W = 2
```

For reads:

```text
R = 2
```

If:

```text
R + W > N
```

then read and write quorums overlap.

For:

```text
N = 3
R = 2
W = 2
```

we get:

```text
2 + 2 > 3
```

Therefore, at least one replica participates in both operations.

### Important

Quorum overlap can help provide strong guarantees, but **quorum configuration alone does not automatically mean the whole system is linearizable**.

The protocol and read/write semantics matter.

---

# 18. Leader-Based Replication

A common CP-oriented architecture:

```text
                 ┌────────────┐
                 │   Leader   │
                 └─────┬──────┘
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
        ┌─────────┐         ┌─────────┐
        │Follower │         │Follower │
        └─────────┘         └─────────┘
```

Writes go through the leader:

```text
Client
   │
   ▼
Leader
   │
   ├────► Follower
   │
   └────► Follower
```

If the leader cannot replicate enough state:

```text
Write
  │
  ▼
Cannot safely commit
  │
  ▼
Retry / reject
```

This can sacrifice availability to preserve consistency.

---

# 19. Consensus and CAP

Protocols such as:

- Raft
- Paxos
- Multi-Paxos

help distributed nodes agree on an ordered sequence of operations.

Simplified Raft-like model:

```text
             Leader
            /      \
           ▼        ▼
       Follower  Follower
```

If the leader loses contact with the majority:

```text
Leader
  │
  X
  │
Majority unavailable
  │
  ▼
Cannot safely commit new writes
```

The system sacrifices availability rather than allowing conflicting committed histories.

---

# 20. Network Partition vs Server Failure

These are not identical.

## Server failure

```text
Client
  │
  ▼
Node A  ✗
  │
  ▼
Node B
```

The network may still work.

A healthy replica can potentially take over.

---

## Network partition

```text
        Network
          X
         / \
        /   \
   Node A   Node B
```

Both nodes may be alive.

The difficult problem is:

> **Which side is allowed to make progress?**

This is where CAP becomes important.

---

# 21. Split Brain

A dangerous partition scenario:

```text
        X NETWORK X

       Node A       Node B
       Leader       Leader
```

Both sides believe they are leaders.

Possible consequences:

- Conflicting writes
- Duplicate work
- Data corruption
- Divergent state

Distributed consensus systems use mechanisms such as:

- Majority/quorum
- Terms/epochs
- Leader election
- Fencing

to prevent unsafe split-brain behavior.

---

# 22. Majority Quorum

Suppose:

```text
5 nodes
```

Majority:

```text
3 nodes
```

Partition:

```text
          NETWORK PARTITION

        ┌─────────┐     X     ┌──────────────┐
        │ Node 1  │           │ Node 3       │
        │ Node 2  │           │ Node 4       │
        │ Node 5  │           │              │
        └─────────┘           └──────────────┘

           3 nodes                 2 nodes
           MAJORITY                MINORITY
```

The majority side can continue leader-based operations.

The minority side should generally stop accepting writes that require consensus.

Why?

Because otherwise both sides could independently commit conflicting operations.

---

# 23. CAP and Multi-Region Systems

CAP becomes especially interesting across regions.

Example:

```text
India Region              US Region

┌──────────────┐          ┌──────────────┐
│ DB replicas  │◄────────►│ DB replicas  │
└──────────────┘   WAN    └──────────────┘
```

The WAN introduces:

- Higher latency
- Packet loss
- Temporary disconnections
- Region failures

If you require synchronous cross-region replication:

```text
Write
  │
  ▼
India
  │
  │ WAN round trip
  ▼
US
  │
  ▼
Commit
```

Consistency improves, but write latency increases.

If you allow asynchronous replication:

```text
Write
  │
  ▼
India → ACK
  │
  └──── async ───► US
```

Latency improves, but US may temporarily have stale data.

---

# 24. CAP Trade-off in Multi-Region Design

### Option A — Strong cross-region consistency

```text
Client
  │
  ▼
Region A
  │
  │ synchronous
  ▼
Region B
  │
  ▼
Commit
```

Pros:

- Stronger consistency
- Easier global correctness

Cons:

- Higher latency
- Region-to-region dependency
- Reduced availability during WAN partitions

---

### Option B — Local writes + async replication

```text
Client
  │
  ▼
Region A
  │
  ├── ACK immediately
  │
  └──── async ───► Region B
```

Pros:

- Low latency
- High availability
- Region isolation

Cons:

- Stale reads
- Conflict resolution
- More application complexity

---

# 25. CAP Does Not Mean "Choose a Database"

A common interview mistake is:

> "MongoDB is AP, PostgreSQL is CP."

This is too simplistic.

Modern databases often provide **multiple consistency/replication modes**.

The correct question is:

> **What consistency and availability guarantees are configured for this particular operation/deployment?**

You need to consider:

- Replication model
- Read concern
- Write concern
- Quorum
- Leader/follower behavior
- Transaction semantics
- Region topology
- Failure mode

---

# 26. Database Choice Should Follow Requirements

Instead of:

```text
MongoDB → AP
PostgreSQL → CP
```

Think:

```text
Business requirement
        │
        ▼
Consistency requirement
        │
        ▼
Availability requirement
        │
        ▼
Latency requirement
        │
        ▼
Replication model
        │
        ▼
Database technology
```

---

# 27. Practical Examples

## Banking / Money Transfer

Requirement:

> Never lose or duplicate money.

Typical preference:

```text
Strong consistency
       +
Transactional guarantees
       +
Durability
       +
Correct concurrency control
```

A CP-style design is usually preferred for critical coordination.

---

## Social Media Feed

Requirement:

> Users should see posts quickly.

A temporary stale view may be acceptable.

Potential approach:

```text
AP-oriented replication
       +
Caching
       +
Async propagation
```

---

## Inventory

Requirement:

> Never sell the last item twice.

Potential approach:

```text
Strong consistency
       +
Atomic update
       +
Concurrency control
```

But the exact architecture depends on business tolerance for overselling.

---

## Analytics Dashboard

Requirement:

> Data can be delayed by a few seconds/minutes.

Potential approach:

```text
Eventual consistency
       +
Async pipeline
       +
High throughput
```

---

## User Profile

Requirement:

> User updates profile from any region and it eventually appears everywhere.

Potential approach:

```text
Eventual consistency
```

if temporary staleness is acceptable.

---

# 28. Consistency Is Not Binary

One of the biggest upgrades from junior to senior distributed-systems thinking is realizing:

```text
Consistency
    │
    ├── Linearizable
    ├── Sequential
    ├── Causal
    ├── Session
    ├── Read-your-writes
    ├── Monotonic reads
    └── Eventual
```

Different applications can choose different guarantees.

---

# 29. Linearizability

Linearizability means operations appear to happen atomically at some point between invocation and response, respecting real-time ordering.

Simplified:

```text
Write x=20
    │
    └──── completes

Later:

Read x
    │
    ▼
20
```

A completed write cannot be followed by a read that observes an older value.

---

# 30. Sequential Consistency

Sequential consistency guarantees that operations appear in some single sequential order that respects the program order of each individual client.

It does **not necessarily preserve real-time ordering across different clients** in the same way linearizability does.

This distinction matters in advanced interviews.

---

# 31. Causal Consistency

If operation B depends on operation A:

```text
A → B
```

everyone should observe:

```text
A before B
```

Example:

```text
User creates post
        │
        ▼
User comments on post
```

A replica should not show:

```text
Comment
  ↓
Post
```

without the post.

---

# 32. Session Guarantees

For user-facing systems, weaker consistency models can still provide a good experience.

Important guarantees include:

### Read Your Writes

If I update my profile:

```text
WRITE name = "John"
```

my next read should return:

```text
John
```

not:

```text
OldName
```

### Monotonic Reads

If I have seen version 10, I should not later see version 8.

```text
Read v10
   │
   ▼
Read v8  ✗
```

These guarantees are often more useful than demanding global linearizability.

---

# 33. PACELC — The Next Level After CAP

CAP only talks about the system **during a partition**.

PACELC extends the reasoning:

> **If there is a Partition, choose between Availability and Consistency; Else, choose between Latency and Consistency.**

```text
                 System
                   │
             Partition?
              /       \
            YES        NO
             │          │
             ▼          ▼
          A vs C      L vs C
```

This is extremely useful in system design.

---

# 34. PACELC Example

Suppose a multi-region database can choose:

### Strong synchronous replication

```text
Consistency ↑
Latency     ↑
```

### Async replication

```text
Latency     ↓
Consistency ↓
```

Even when there is **no partition**, you still face:

```text
Latency vs Consistency
```

That is the "ELC" part of PACELC.

---

# 35. CAP vs PACELC

| Model | Main question |
|---|---|
| CAP | What happens during a partition? |
| PACELC | During partition: A vs C; otherwise: L vs C |

### Interview-level answer

> "CAP helps reason about the unavoidable consistency/availability trade-off during network partitions. PACELC extends the model by highlighting that even when the network is healthy, stronger consistency can require coordination and therefore higher latency."

---

# 36. CAP and Caching

Caching can introduce another consistency problem.

```text
Database
   │
   ▼
Cache
   │
   ▼
Client
```

Suppose:

```text
DB = 100
Cache = 100
```

Update:

```text
DB = 200
```

But cache still has:

```text
Cache = 100
```

A read returns stale data.

Therefore, in system design, always ask:

> **What is the consistency requirement between the cache and source of truth?**

---

# 37. Cache Consistency Strategies

## Cache-aside

```text
Read
 │
 ▼
Cache ── hit ──► return
 │
 miss
 ▼
DB
 │
 ▼
Cache
```

Simple but stale data is possible.

---

## Write-through

```text
Write
 │
 ▼
Cache
 │
 ▼
DB
```

Cache and database are updated together from the application's perspective.

---

## Write-behind

```text
Write
 │
 ▼
Cache
 │
 └──── async ───► DB
```

Lower write latency but greater durability/consistency complexity.

---

# 38. CAP and Kafka

Kafka is often casually called "AP" or "CP".

That is not a sufficiently precise statement.

Kafka provides configurable durability, replication and acknowledgment semantics.

For example:

```text
acks=0
acks=1
acks=all
```

and replication settings influence durability and availability behavior.

The correct interview approach:

> Analyze the Kafka cluster configuration, replication factor, ISR behavior, producer acknowledgments, consumer semantics, and failure scenario rather than assigning one CAP label to "Kafka."

---

# 39. CAP and Message Delivery

Suppose:

```text
Producer
   │
   ▼
Broker
   │
   ▼
Consumer
```

Ask:

- Can messages be lost?
- Can they be duplicated?
- Is ordering required?
- How long can delivery be delayed?
- What happens when the broker is unavailable?
- Can the producer continue?
- Is acknowledgement synchronous?
- What happens during partition?

This is much more valuable than simply saying:

> "Use Kafka because it is scalable."

---

# 40. Availability Has Multiple Meanings

"Highly available" is not a complete requirement.

Ask:

### Available for reads?

```text
Can users still read?
```

### Available for writes?

```text
Can users still write?
```

### Available globally?

```text
Can every region continue?
```

### Available locally?

```text
Can unaffected regions continue?
```

### Available with stale data?

```text
Can the system respond with older data?
```

A system might provide:

```text
Read availability = 99.99%
Write availability = lower
```

and still be a good design.

---

# 41. Partial Availability

A sophisticated distributed system does not necessarily choose:

```text
Everything works
       OR
Everything fails
```

It can degrade selectively.

Example:

```text
Payment        → unavailable
Product browse → available
Search         → available
Recommendations→ stale but available
Order history  → read-only
```

This is often better than sacrificing availability of the entire application.

---

# 42. Graceful Degradation

During a partition:

```text
Normal system
     │
     ▼
Network failure
     │
     ▼
Degraded mode
```

Examples:

- Serve cached product data
- Disable non-critical writes
- Show stale recommendations
- Queue non-critical operations
- Make critical operations read-only
- Reject only correctness-sensitive transactions

This allows business requirements to determine where CAP trade-offs are made.

---

# 43. CAP Is a Per-Operation / Per-Subsystem Design Decision

A large system does not have to be globally "CP" or "AP."

Example:

```text
E-commerce Platform

Payments       → strong consistency
Inventory      → strong consistency
Product catalog→ eventual consistency
Search index   → eventual consistency
Recommendations→ eventual consistency
Analytics      → eventual consistency
Session data   → relaxed consistency
```

This is a very important architecture insight.

---

# 44. One System Can Contain Both CP and AP Components

```text
                    E-Commerce

       ┌─────────────┴─────────────┐
       ▼                           ▼
   Checkout                    Product Feed
       │                           │
      CP                          AP
       │                           │
       ▼                           ▼
 Strong DB                    Replicated Cache
       │                           │
       └──────────┬────────────────┘
                  ▼
               Kafka
                  │
                  ▼
              Analytics
                 AP
```

Do not force the entire architecture into one CAP category.

---

# 45. Conflict Resolution

AP systems may accept concurrent updates.

Example:

```text
Replica A:
quantity = 10

Replica B:
quantity = 10

Partition

A → quantity = 8
B → quantity = 7
```

Now:

```text
How do we merge?
```

Possible strategies:

- Last-write-wins
- Version vectors
- Vector clocks
- CRDTs
- Application-specific merge
- Conflict rejection
- Manual reconciliation

---

# 46. Last-Write-Wins

A simple conflict strategy:

```text
Update A:
x = 20 @ 10:00:01

Update B:
x = 30 @ 10:00:02

Winner:
x = 30
```

### Pros

- Simple
- Easy to implement

### Cons

- Can lose valid updates
- Clock assumptions can matter
- Not suitable for every domain

For financial data, blindly using LWW can be dangerous.

---

# 47. CRDTs

CRDT = Conflict-Free Replicated Data Type.

The idea:

> Design data structures whose concurrent updates can be merged deterministically without central coordination.

Useful for some:

- Collaborative editing
- Counters
- Sets
- Distributed state

Example conceptual counter:

```text
Replica A → +3
Replica B → +5

Merge → +8
```

CRDTs are powerful but not a universal solution.

They require the data operation to have appropriate merge semantics.

---

# 48. CAP + Idempotency

Distributed systems often retry operations.

```text
Client
  │
  │ POST
  ▼
Server
  │
  │ operation succeeds
  │
  X response lost
  │
Client retries
  │
  ▼
Server
```

Without idempotency:

```text
Charge ₹100
Charge ₹100
```

The user may be charged twice.

Therefore, when designing distributed systems, consider:

- Idempotency keys
- Deduplication
- Request IDs
- Exactly-once business semantics

---

# 49. CAP + Retries

Retries can make an outage worse.

Imagine:

```text
Service A
   │
   │ 1000 requests
   ▼
Service B
   │
   X overloaded
```

A retries:

```text
1000
 ↓
2000
 ↓
4000
 ↓
8000
```

This can create a retry storm.

Use:

- Exponential backoff
- Jitter
- Retry budgets
- Timeouts
- Circuit breakers
- Rate limits

---

# 50. CAP + Timeouts

Never design distributed systems assuming:

> "The network will eventually respond."

A network call can:

- succeed
- fail immediately
- hang
- succeed but response is lost
- partially complete

Therefore:

```text
Every distributed call
        │
        ├── Timeout
        ├── Retry policy
        ├── Idempotency
        └── Failure handling
```

---

# 51. CAP + "Exactly Once"

Be careful with statements like:

> "We guarantee exactly-once processing."

In distributed systems, achieving true end-to-end exactly-once behavior is difficult.

Usually systems combine:

```text
At-least-once delivery
       +
Idempotent processing
       +
Deduplication
       +
Transactional boundaries
```

to achieve:

> **Exactly-once business effect**

This distinction is excellent interview material.

---

# 52. Real-World System Design: URL Shortener

Suppose:

```text
POST /shorten
```

creates:

```text
abc123 → https://example.com/...
```

### Short-code creation

Correctness requirement:

> Two URLs must not receive the same code.

Potentially CP-oriented operation:

```text
Generate ID
    │
    ▼
Uniqueness check / atomic write
    │
    ▼
Commit
```

### Redirect

For:

```text
GET /abc123
```

temporary stale data may be acceptable depending on requirements.

Therefore:

```text
Creation → stronger consistency
Redirect → high availability + caching
Analytics → eventual consistency
```

This is a much more realistic use of CAP than saying:

> "URL shortener is AP."

---

# 53. Real-World System Design: Ride Sharing

Different components have different requirements.

```text
Driver location
    → high availability
    → low latency
    → temporary stale data acceptable

Trip assignment
    → stronger consistency
    → avoid assigning one driver to two trips

Payment
    → strong consistency
    → transactional correctness

Analytics
    → eventual consistency
```

Therefore:

```text
Ride-sharing system
        │
        ├── Location → AP-ish
        ├── Assignment → CP-ish
        ├── Payment → Strong consistency
        └── Analytics → Eventual
```

---

# 54. Real-World System Design: Banking

Consider a transfer:

```text
A: ₹10,000
B: ₹5,000

Transfer ₹2,000

A: ₹8,000
B: ₹7,000
```

You need to preserve invariants.

You generally cannot tolerate:

```text
A: ₹8,000
B: ₹5,000
```

because money disappeared.

The design therefore prioritizes:

- Strong correctness
- Transactional integrity
- Durable commits
- Concurrency control
- Appropriate isolation
- Recovery

Availability can be sacrificed for critical operations if required.

---

# 55. Real-World System Design: Social Media

For a feed:

```text
User opens feed
      │
      ▼
Read replica / cache
      │
      ▼
Possibly slightly stale feed
```

Users usually tolerate:

> "I see a post a few seconds late."

They generally do not tolerate:

> "The application is unavailable every time replicas are temporarily disconnected."

So the system can favor:

```text
Availability
+
Eventual consistency
```

---

# 56. Real-World System Design: Stock Trading

This is more nuanced than simply saying "CP."

Different operations have different requirements:

```text
Market data display
    → very low latency
    → stale by milliseconds may or may not be acceptable

Order placement
    → correctness critical

Order matching
    → strict ordering and deterministic processing

Account balance
    → strong consistency

Historical analytics
    → eventual consistency acceptable
```

Again:

> **CAP decisions belong to specific data flows and operations, not just the product name.**

---

# 57. CAP Decision Framework

Use this in every system-design interview:

```text
                    START
                      │
                      ▼
            Is the system distributed?
                      │
                      ▼
             Can network partition?
                      │
                      ▼
        What must happen during partition?
               /                 \
              /                   \
             ▼                     ▼
     Must remain correct       Must remain
     / strongly consistent?   available?
             │                     │
             ▼                     ▼
            CP                    AP
             │                     │
             ▼                     ▼
    Reject/block some        Accept/serve
       operations            potentially stale
             │                     │
             └──────────┬──────────┘
                        ▼
             Define business invariants
                        │
                        ▼
              Define consistency level
                        │
                        ▼
               Define failure behavior
```

---

# 58. The Better Question: "What Is the Cost of Inconsistency?"

Instead of asking:

> "Should I choose CP or AP?"

Ask:

> **"What happens if two users temporarily see different values?"**

### If answer is:

> "Money can be lost."

Prefer stronger consistency.

### If answer is:

> "A user may see a feed update a few seconds late."

Eventual consistency may be fine.

### If answer is:

> "We can reconcile later."

AP becomes attractive.

### If answer is:

> "We cannot automatically reconcile."

Prefer stronger coordination.

---

# 59. Business Invariants

The strongest system-design answers start from invariants.

Examples:

### Inventory

```text
inventory >= 0
```

### Unique username

```text
username must be unique
```

### Money

```text
total money must be conserved
```

### Seat booking

```text
one seat → one confirmed booking
```

### Social likes

```text
like count should eventually converge
```

The invariant tells you how much consistency you need.

---

# 60. CAP vs Latency

Strong consistency often requires coordination.

Suppose:

```text
Region A
   │
   │ 150 ms
   ▼
Region B
```

A globally synchronous write may cost approximately:

```text
local processing
+
network round trip
+
replication
```

This affects:

- p50 latency
- p95 latency
- p99 latency
- throughput

Therefore, in global systems:

> **Consistency is often a latency decision too.**

This is why PACELC is valuable.

---

# 61. CAP and Read/Write Paths

A system may make different decisions for reads and writes.

Example:

```text
WRITE
  │
  ▼
Leader / quorum
  │
  ▼
Strongly committed
```

while:

```text
READ
  │
  ▼
Nearest replica
  │
  ▼
Possibly stale
```

This creates a hybrid consistency model.

The design may be perfectly valid if the business requirement allows it.

---

# 62. Read-Your-Writes Architecture

Suppose:

```text
User updates profile
       │
       ▼
Primary
       │
       └──── async ───► Replica
```

Immediately reading from the replica can return stale data.

A common solution:

```text
After write:
route user's reads to primary
       │
       ▼
until replica catches up
```

or use:

```text
session/version token
```

to ensure reads meet the user's required consistency.

---

# 63. CAP + Load Balancers

Load balancing can expose consistency problems.

```text
             Load Balancer
              /         \
             ▼           ▼
         Server A     Server B
            │             │
          Cache A       Cache B
```

If caches are independent:

```text
User 1 → A → value 10
User 1 → B → value 8
```

The load balancer is not the root problem.

The issue is:

> **Distributed state consistency.**

Solutions include:

- Shared cache
- Sticky sessions
- Cache invalidation
- Versioning
- Central source of truth

---

# 64. CAP + CDN

CDNs intentionally cache data close to users.

```text
Origin
  │
  ├────► CDN Edge A
  │
  ├────► CDN Edge B
  │
  └────► CDN Edge C
```

After an update:

```text
Origin = v2

Edge A = v2
Edge B = v1
Edge C = v1
```

This is a deliberate consistency trade-off.

The system accepts:

```text
Temporary staleness
```

for:

```text
Lower latency
+
Lower origin load
+
Higher availability
```

---

# 65. CAP + DNS

DNS itself demonstrates distributed consistency trade-offs.

DNS uses:

- Caching
- TTL
- Replication
- Distributed authoritative servers

After an update:

```text
Authoritative DNS = new IP

Some clients/cache:
old IP
```

For a period, different clients may observe different values.

This is acceptable because DNS prioritizes scalability and availability while using TTLs to control staleness.

---

# 66. CAP + Search Index

Suppose:

```text
Primary DB
     │
     │ async
     ▼
Search Index
```

User creates:

```text
Product X
```

Database immediately has it.

Search index may receive it later.

```text
DB       → Product X ✓
Search   → Product X ✗
```

This is eventual consistency.

Usually acceptable because:

> A product appearing in search a few seconds later is better than making the entire write path dependent on synchronous search indexing.

---

# 67. CAP + Event-Driven Architecture

Event-driven systems naturally introduce asynchronous boundaries.

```text
Service A
   │
   │ event
   ▼
Kafka
   │
   ├────► Service B
   ├────► Service C
   └────► Service D
```

Now:

```text
A state changed
      │
      ▼
B may update later
C may update later
D may update later
```

This is often eventual consistency.

The benefit:

- Loose coupling
- Independent scaling
- Fault isolation
- High throughput

The cost:

- Delayed propagation
- More complex debugging
- Duplicate events
- Ordering issues
- Reconciliation

---

# 68. CAP + Saga

In distributed transactions, you often cannot use one ACID transaction across all services.

Instead:

```text
Order
  │
  ▼
Payment
  │
  ▼
Inventory
  │
  ▼
Shipping
```

A failure may require compensation:

```text
Payment succeeded
Inventory failed
      │
      ▼
Refund payment
```

This is a **Saga** pattern.

The system chooses distributed coordination/compensation instead of a single global transaction.

---

# 69. CAP + Distributed Transactions

Two-phase commit (2PC) can provide stronger coordination:

```text
Coordinator
   │
   ├────► Service A
   │
   └────► Service B

Prepare
   │
   ▼
Commit
```

But it can have:

- Higher latency
- Blocking behavior
- Coordinator dependency
- Reduced availability

For many microservice architectures:

```text
Saga + eventual consistency
```

is preferred over global 2PC.

But the business invariant determines the choice.

---

# 70. Common Interview Mistakes

## Mistake 1

> "CAP means choose two of three."

### Better

> "During a network partition, a distributed system cannot simultaneously guarantee both strong consistency and availability."

---

## Mistake 2

> "P is optional."

### Better

> "For a distributed system, partitions are unavoidable failure modes, so practical designs assume partition tolerance and reason about C vs A during the partition."

---

## Mistake 3

> "MongoDB is AP."

### Better

> "We need to inspect the deployment and configured read/write consistency and replication semantics."

---

## Mistake 4

> "PostgreSQL is CP."

### Better

> "PostgreSQL provides strong transactional semantics, but CAP classification depends on the distributed architecture and replication/failure configuration."

---

## Mistake 5

> "Eventual consistency means bad consistency."

Wrong.

It means:

> Replicas can temporarily disagree but converge when the system stabilizes, assuming the relevant conditions hold.

---

## Mistake 6

> "Strong consistency means no failures."

Wrong.

Strong consistency is a **correctness guarantee**, not a promise that the system never fails.

---

# 71. Senior Interview Answer Template

When asked:

> "Would you choose CP or AP?"

Answer:

```text
1. Identify the business invariant.
2. Determine whether stale reads are acceptable.
3. Determine whether writes can continue during partition.
4. Define what happens when replicas cannot communicate.
5. Choose consistency semantics.
6. Explain the availability trade-off.
7. Discuss latency impact.
8. Explain recovery/reconciliation.
```

Example:

> "For inventory reservation, I would favor a CP-style design because the key invariant is that we cannot allocate the same scarce item beyond its available quantity. During a partition, I'd rather reject or delay some writes than accept potentially conflicting reservations. For product browsing, I'd choose a more available and eventually consistent design because stale product information for a short period is acceptable."

---

# 72. CAP One-Minute Explanation

> **CAP says that in a distributed system, when a network partition occurs, you cannot simultaneously guarantee strong consistency and availability. Partition tolerance means the system continues to deal with communication failures. A CP system sacrifices some availability during the partition to preserve consistency, while an AP system continues serving requests and may temporarily return divergent or stale data. The correct choice depends on the business cost of inconsistency.**

---

# 73. CAP in 30 Seconds

```text
Network partition happens
        │
        ▼
Nodes cannot coordinate
        │
        ├── Continue serving?
        │       │
        │       ▼
        │      AP
        │
        └── Reject/block unsafe operations?
                │
                ▼
               CP
```

---

# 74. One-Page Revision

```text
CAP
│
├── C = Consistency
│      └── In CAP: linearizable/single-copy style visibility
│
├── A = Availability
│      └── Requests receive non-error responses
│
└── P = Partition Tolerance
       └── Continue despite network communication failure


During partition:

CP
│
├── Preserve strong consistency
├── Reject/block some requests
├── Good for correctness-critical data
└── Example: inventory / coordination / critical metadata


AP
│
├── Continue serving
├── Temporary divergence/stale reads possible
├── Reconcile later
└── Example: feeds / recommendations / analytics


IMPORTANT:
CAP is not simply "choose 2".
The fundamental trade-off is C vs A when P occurs.
```

---

# 75. CAP Decision Matrix

| Requirement | Prefer | Why |
|---|---|---|
| Bank balance | CP / strong consistency | Incorrect value is unacceptable |
| Inventory reservation | CP | Avoid overselling |
| Seat booking | CP | One seat cannot have two confirmed owners |
| Distributed lock | CP | Split-brain is dangerous |
| Leader election | CP | Need one authoritative leader |
| Product catalog | Often relaxed/eventual | Short stale window acceptable |
| Social feed | AP-oriented | Availability and latency matter |
| Likes/views | AP-oriented | Exact immediate count often unnecessary |
| Recommendations | AP | Stale data is acceptable |
| Analytics | AP/eventual | Delayed data acceptable |
| Search index | Eventual | Async indexing is acceptable |
| CDN cache | Eventual | Staleness traded for latency |
| Critical metadata | Strong/CP-oriented | Incorrect metadata can break system |

---

# 76. The Most Important Design Insight

Do not ask:

> **"Is my system CP or AP?"**

Ask:

> **"Which operations require strong consistency, and which operations can tolerate temporary inconsistency?"**

Then design each subsystem accordingly.

For example:

```text
                   E-Commerce

             ┌──────────────────┐
             │      System      │
             └────────┬─────────┘
                      │
       ┌──────────────┼───────────────┐
       ▼              ▼               ▼
   Payments       Inventory       Product Feed
      CP              CP                AP
       │              │                 │
       ▼              ▼                 ▼
 Strong DB        Strong DB         Eventual DB
       │              │                 │
       └──────────────┼─────────────────┘
                      ▼
                   Analytics
                       AP
```

This is the level of thinking expected in strong system-design interviews.

---

# 77. Final CAP Interview Checklist

Before finalizing a distributed-system design, ask:

- [ ] What are my business invariants?
- [ ] Is the system actually distributed?
- [ ] What happens if nodes cannot communicate?
- [ ] What exactly does "consistent" mean for this operation?
- [ ] Are stale reads acceptable?
- [ ] Can writes continue during a partition?
- [ ] What is the cost of rejecting a request?
- [ ] What is the cost of accepting inconsistent data?
- [ ] Do I need linearizability?
- [ ] Would session consistency be enough?
- [ ] Can eventual consistency work?
- [ ] Do I need quorum?
- [ ] Do I need a leader?
- [ ] What happens to the minority partition?
- [ ] How do I prevent split brain?
- [ ] How are conflicts resolved?
- [ ] What is the replication model?
- [ ] Is replication synchronous or asynchronous?
- [ ] What is the cross-region latency?
- [ ] What happens during region failure?
- [ ] What happens after recovery?
- [ ] Is reconciliation required?
- [ ] Are retries safe?
- [ ] Are operations idempotent?
- [ ] What are the timeout and retry policies?
- [ ] What happens to caches?
- [ ] What happens to search indexes?
- [ ] What happens to asynchronous consumers?
- [ ] What is the user-visible degraded mode?
- [ ] Does PACELC change the decision because of latency?

---

# 78. Final Mental Model

Keep this in your head:

```text
                 DISTRIBUTED SYSTEM
                         │
                         ▼
                 Network partition?
                         │
                  ┌──────┴──────┐
                  │             │
                 YES            NO
                  │             │
                  ▼             ▼
             CAP decision     PACELC
                  │             │
             C vs A            L vs C
             /     \           /   \
            /       \         /     \
          CP         AP      Low    Strong
          │           │      latency consistency
          │           │
          ▼           ▼
       Correct      Available
       but may      but may
       reject       be stale
```

The core principle:

> **Distributed-system design is about deciding which failures and inconsistencies your business can tolerate.**

Technology selection comes **after** that decision.
