# Data Consistency in Distributed Systems

## Introduction

**Consistency** is one of the most critical and complex aspects of distributed systems design. It defines what guarantees a system makes about the state of data when multiple clients read and write concurrently, especially when failures occur.

There are two main consistency models you'll encounter:
1. **ACID Consistency** (single-database focus)
2. **CAP Consistency** (distributed systems focus)

Understanding the difference and tradeoffs between them is essential for system design interviews.

---

## Part 1: Consistency Models - CAP vs ACID

### ACID Consistency (Database-Level)

**ACID** is an acronym for four properties of database transactions:

#### A - Atomicity
**Definition**: A transaction is "all-or-nothing" - it either completely succeeds or completely fails.

```
Example: Bank transfer from Account A to B

Transaction: 
  BEGIN
  Debit Account A: balance -= $100
  Credit Account B: balance += $100
  END

Atomicity Guarantee:
  ✓ Both operations succeed → Transfer complete
  ✓ Both operations fail → No change (money safe)
  ✗ NEVER: Debit succeeds but credit fails
  ✗ NEVER: Credit succeeds but debit fails

If system crashes mid-transaction:
  → Either roll back to previous state (both operations undone)
  → Never in inconsistent state (100$ deducted but not added)
```

#### C - Consistency
**Definition**: Database moves from one valid state to another valid state. All business rules/constraints are maintained.

```
Database Rules:
  - Account balance >= 0
  - Total money in system = conserved
  - Foreign key references must exist

State Before Valid Transaction:
  Account A: $500
  Account B: $300
  Total: $800

Transaction: Transfer $100 from A to B
  A: $500 - $100 = $400 ✓
  B: $300 + $100 = $400 ✓

State After: Valid
  Account A: $400
  Account B: $400
  Total: $800 ✓ (conserved)

All constraints satisfied!
```

#### I - Isolation
**Definition**: Concurrent transactions don't interfere with each other. Each transaction executes as if it's the only one running.

```
Scenario: Two concurrent transactions (without isolation)

Transaction 1: Transfer $100 A→B
Transaction 2: Transfer $50 C→B

Without Isolation (Problem):
  T1: Read A = $500
  T2: Read C = $200
  T1: Debit A → A = $400
  T2: Debit C → C = $150
      [Both read B = $300 STALE!]
  T1: Credit B → B = $400 (only adds $100)
  T2: Credit B → B = $350 (overwrites! $50 not added)

Result: Lost $50! B should be $450, not $350

With Isolation (Locking):
  T1: Lock A, B
  T2: Waits for T1...
  T1: Read A=$500, Debit A=$400, Read B=$300, Credit B=$400, Commit
  T1: Release locks
  T2: Lock A, B
  T2: Read C=$200, Debit C=$150, Read B=$400, Credit B=$450, Commit

Result: Correct! B = $450
```

**Isolation Levels** (increasing strictness):
- **Read Uncommitted**: Dirty reads allowed (sees uncommitted changes)
- **Read Committed**: Only sees committed data
- **Repeatable Read**: Same query returns same results within transaction
- **Serializable**: Strictest - equivalent to transactions running one-by-one

#### D - Durability
**Definition**: Once a transaction commits, it stays committed even if system fails.

```
Transaction: UPDATE balance = $500

Status: Committed
System crashes 1 second later
Power failure
Disk failure
...

Durability Guarantee:
  ✓ No data loss
  ✓ Restart system → Data still shows balance = $500
  
Use: Write-Ahead Logging (WAL)
  1. Write transaction to persistent log FIRST
  2. THEN apply to in-memory state
  3. On restart: Replay log, recover state
```

### CAP Theorem (Consistency, Availability, Partition Tolerance)

**CAP Theorem** states that in distributed systems, you can guarantee at most **2 out of 3** properties:

```
       ┌─────────────────┐
       │    CONSISTENCY  │ (all nodes see same data)
       └────────┬────────┘
                │
       ┌────────┼────────┐
       │        │        │
       ▼        ▼        ▼
      C-A      C-P      A-P
   (Impossible) (SQL)  (NoSQL)

You MUST have Partition Tolerance!
Network partitions are inevitable in real systems.

Therefore, real choice is:
  → Choose CP (Consistency + Partition Tolerance): Sacrifice Availability
  → Choose AP (Availability + Partition Tolerance): Sacrifice Strong Consistency
```

#### C - Consistency
**CAP Definition**: When a write completes, ALL subsequent reads return that written value.

```
Write propagates to all nodes BEFORE returning to client:

Client writes X=100
     ↓
   Master updates X=100
     ↓
Master waits for all Replicas to update
     ↓
Replica 1: X=100 ✓
Replica 2: X=100 ✓
Replica 3: X=100 ✓
     ↓
Master returns "Write successful" to Client

Now ANY read from ANY node:
  → Reads X=100 (strongly consistent)

Guarantee: No stale reads
Cost: Network latency, reduced throughput
```

#### A - Availability
**CAP Definition**: Every request receives a response (success or failure), even with failures.

```
System with 3 nodes:
  Master
  Replica 1
  Replica 2

Write arrives while Master is down
  └─ System must still accept and process write

Availability Guarantee:
  → Can write to Replica 1 (accepts write despite Master failure)
  → Can read from Replica 2 (gets response)

Challenge: Replicas may have different values!
  → Replica 1 (latest): X=100
  → Master (down): X=90
  → Replica 2 (stale): X=80

Result: Stale/inconsistent reads, but system stays online
```

#### P - Partition Tolerance
**CAP Definition**: System continues operating despite network partitions.

```
Network partition creates two zones:

Zone A:                     Zone B:
Master                      Replica 1
└─ Can serve requests       └─ Can serve requests
└─ Makes decisions          └─ Makes decisions independently

Network link down between zones:
Master can't reach Replica 1
Replica can't reach Master

Partition Tolerance:
  ✓ Zone A: Master continues accepting writes
  ✓ Zone B: Replica continues accepting reads/writes
  ✓ System operates in both zones

Alternative (no partition tolerance):
  × Zone A: Master sees partition, shuts down writes
  × Zone B: Replica sees partition, shuts down writes
  × Entire system becomes unavailable
  (This violates A - Availability)
```

### Comparing ACID vs CAP

| Aspect | ACID | CAP |
|--------|------|-----|
| **Scope** | Single database | Distributed systems |
| **Consistency** | Business rules maintained | All nodes see same data |
| **Focus** | Transaction properties | System properties during failures |
| **Partition** | Assumed not to fail | Assumes partitions occur |
| **Example Systems** | PostgreSQL, MySQL | Cassandra, DynamoDB |

**Key Relationship**:
- ACID is about **what** consistency means for transactions
- CAP is about **when** consistency can be guaranteed in distributed systems
- A system can aim for ACID properties within each node, but CAP limits what entire system can achieve

---

## Part 2: How Databases Resolve Consistency

### Consistency Resolution Strategies

#### Strategy 1: Immediate Propagation (Strong Consistency)

```
Goal: All nodes always have same data

Write Path:
Client: "Set User.age = 30"
     ↓
Primary Node: Apply write locally
     ↓
Primary Node: Send to all replicas
     ↓
Primary Node: WAIT for acknowledgment from every replica
     ↓
All replicas: Acknowledge "Write applied"
     ↓
Primary: Return success to client
     ↓
Read: Can read from ANY node → always get latest value


Guarantees:
  ✓ Strong consistency (no stale reads)
  ✓ ACID properties maintained
  ✗ High latency (wait for slowest replica)
  ✗ Reduced availability (one slow replica blocks all)

Used by: PostgreSQL, MySQL (with synchronous replication), VoltDB
```

#### Strategy 2: Asynchronous Propagation (Eventual Consistency)

```
Goal: Replicate eventually, but don't wait

Write Path:
Client: "Set User.age = 30"
     ↓
Primary Node: Apply write locally
     ↓
Primary Node: Immediately return success to client
     ↓
Primary Node: (Background) Send to replicas
     ↓
Replicas: Apply when they get the change (may be delayed)


Guarantees:
  ✓ Low latency (return immediately)
  ✓ High availability (don't wait for replicas)
  ✗ Eventual consistency (temporary stale reads)
  ✗ Data loss risk (primary fail before replication)

Used by: Cassandra, DynamoDB, MongoDB (default replication)
```

#### Strategy 3: Quorum-Based Resolution

```
Goal: Balance consistency and availability

Config: 5 node cluster
  Write Quorum: 3 nodes
  Read Quorum: 3 nodes

Write Process:
  Replicate to 3 nodes (W=3)
  └─ Only 2 replies? FAIL the write
  └─ 3+ replies? Succeed and return

Read Process:
  Read from 3 nodes (R=3)
  └─ Compare versions: pick latest by timestamp
  └─ Return latest value

Consistency with Quorum:
  If W + R > N (total nodes):
    ✓ Read will overlap with last write
    ✓ Guaranteed to read latest value
  
  Example: W=3, R=3, N=5
  3 + 3 = 6 > 5 ✓ → Consistency!
  
  Example: W=2, R=2, N=5
  2 + 2 = 4 < 5 ✗ → Potential stale reads

Trade-off:
  W=1, R=5: Fast writes, slow reads
  W=5, R=1: Slow writes, fast reads
  W=3, R=3: Balanced

Used by: DynamoDB, Cassandra (tunable), Dynamo (Amazon internal system)
```

### Conflict Resolution Techniques

When replicas have diverged (due to partition or async replication), databases use conflict resolution:

#### Last-Write-Wins (LWW)

```
Node A: user_profile = {name: "Alice", updated_at: 1000}
Node B: user_profile = {name: "Bob", updated_at: 1005}

Conflict detected during sync

LWW Strategy:
  Compare timestamps: 1005 > 1000
  Winner: Node B's version
  Result: name = "Bob"

Pros: Simple, automatic resolution
Cons: Data loss (Node A's write discarded)
Used by: Cassandra (default), DynamoDB
```

#### Vector Clocks

```
Vector Clock: [Node A: 5, Node B: 3, Node C: 7]
  Encodes which node wrote data and in what order

Scenario: Concurrent writes to same key

Node A writes: vc = [1, 0]
             key = "value_A"

Node B writes: vc = [0, 1]
             key = "value_B"

Network partition: Nodes diverge
Both writes succeed in separate zones

On merge:
  Compare vectors: [1, 0] vs [0, 1]
  Neither dominates (neither is larger in all positions)
  → Conflict! Both writes happened concurrently

Resolution:
  Store both versions (siblings)
  Let application choose or merge
  Example: Riak, Voldemort

Advantage: Distinguishes concurrent vs sequential writes
```

#### Application-Level Resolution

```
Scenario: Shopping cart merge

User has two shopping carts (due to replication):
  Cart A (updated on Node A): [iPhone, Airpods]
  Cart B (updated on Node B): [MacBook]

Naive last-write-wins:
  → One cart discarded (data loss)

Smart conflict resolution:
  Merge operation: Combine items
  → Final cart: [iPhone, Airpods, MacBook]
  
  Application chooses merge strategy based on business logic
  
Used by: CouchDB, Riak, custom application code
```

---

## Part 3: CAP Theorem - Consistency vs Availability Tradeoff

### The Fundamental Impossibility

**CAP Theorem Statement**: In presence of a network partition, you must choose between **Consistency** and **Availability**.

### Why You Can't Have All Three

```
Three properties in CAP:

C (Consistency): All nodes same data
A (Availability): System keeps running
P (Partition Tolerance): System works despite network splits

Scenario: Network Partition Occurs!

Before Partition:
┌─────────────────────────────────────┐
│ Master  ←→  Replica  ←→  Replica   │
│ All in sync, all available          │
└─────────────────────────────────────┘

After Partition:
┌──────────────────────────────┬──────────────────────────────┐
│        Zone A                │         Zone B               │
│  Master (isolated)           │   Replicas (isolated)        │
│  Can it serve writes?        │   Can they serve reads/writes│
└──────────────────────────────┴──────────────────────────────┘
              ×××××× Network Down ××××××

Now the system must choose:

OPTION 1: Choose Consistency (CP)
  Zone A (Master): "I'm isolated, might conflict with Zone B"
    → Reject all writes (maintain consistency)
    → Zone A becomes unavailable for writes
  
  Zone B (Replicas): "Master unreachable, don't accept writes"
    → Can only serve reads of old data
    → Zone B becomes unavailable
  
  Result: System unavailable (no A)
  
  ✓ C - All nodes have consistent data (old but same)
  ✗ A - Cannot write/serve requests
  ✓ P - System prepared for partition

OPTION 2: Choose Availability (AP)
  Zone A (Master): "Accept writes, serve clients"
    → Updates: {balance: 100}
    → Clients: Returns success
  
  Zone B (Replicas): "Accept writes independently"
    → Updates: {balance: 50}
    → Clients: Returns success
  
  Result: System available in both zones
  ✗ C - Nodes have different data (conflict!)
  ✓ A - Can write and read in both zones
  ✓ P - System prepared for partition
  
  When partition heals:
    Reconcile conflicting data (with data loss or merge)

OPTION 3: Choose Ignore Partition (Impossible)
  × Partitions happen in real systems (network failures)
  × You can't prevent them
  × Must handle them
  
  Therefore: Partition Tolerance is mandatory
  Real choice: CP vs AP
```

### Visual: CAP Triangle

```
        ┌─────────────────────┐
        │   CONSISTENCY       │─── All replicas
        │  (C)                │    same value
        └──────────────────────┘
              │        │
              │        │
        CP    │        │    CA
      (SQL DB)│        │  (RDBMS
             │        │    no partition
             │        │     tolerance)
        ┌─────┴────────┴──────┐
        │                     │
        │  PARTITION          │
        │  TOLERANCE          │
        │  (P)                │
        │  Network fails      │
        │  System continues   │
        │                     │
        └─────┬────────┬──────┘
              │        │
         AP   │        │   CP
      (NoSQL) │        │ (Paxos,
              │        │  Raft)
        ┌─────┴────────┴──────┐
        │   AVAILABILITY      │─── Serve on
        │  (A)                │    any failure
        └─────────────────────┘


Regions:
CP: Consistency + Partition (sacrifices Availability)
    Example: PostgreSQL with synchronous replication
    
AP: Availability + Partition (sacrifices Consistency)
    Example: MongoDB with async replication
    
CA: Consistency + Availability (no Partition tolerance)
    Example: Single monolithic database (no replication)
```

---

## Part 4: Leader-Follower Architecture and Consistency/Availability

### Leader-Follower Model

**Leader-Follower** (also called Master-Slave) is the most common replication pattern in distributed databases.

```
Architecture:

              LEADER (Master)
              ├─ Accepts ALL writes
              ├─ Has authoritative data
              └─ Replicates to followers
                  ↓
        ┌─────────┼─────────┐
        ↓         ↓         ↓
    Follower   Follower   Follower
    ├─ Read    ├─ Read    ├─ Read
    └─ Replica └─ Replica └─ Replica
```

### How Leader-Follower Affects Consistency

#### Strong Consistency with Leader-Follower (Synchronous Replication)

```
Configuration:
  Replication mode: SYNCHRONOUS
  Write-concern ACK: ALL replicas must acknowledge

Write Path:
Client: "Write user.age = 30"
     ↓
   Leader: Apply write to own storage
     ↓
   Leader: Send replication log to Follower 1, 2, 3
     ↓
   Leader: BLOCK - wait for acknowledgments
     ↓
   Followers: Apply write
     ↓
   Followers: Send ACK back
     ↓
   Leader: All ACKs received? YES
     ↓
   Leader: Return success to client
     ↓
   Read: Any node can serve (all have same data)

Guarantees:
  ✓ Strong Consistency - all nodes identical before responding
  ✓ No data loss
  ✓ CAP: Provides Consistency + Partition Tolerance (CP)

Tradeoff - Availability Problem:
  If one Follower slow/down:
    Leader waits → Client waits → Latency increases
    
  If network partition:
    Leader can't reach followers
    → Cannot proceed with writes
    → Write unavailable (sacrifices A)
```

#### Weak Consistency with Leader-Follower (Asynchronous Replication)

```
Configuration:
  Replication mode: ASYNCHRONOUS
  Write-concern ACK: Immediate (don't wait for followers)

Write Path:
Client: "Write user.age = 30"
     ↓
   Leader: Apply write to own storage
     ↓
   Leader: Immediately return success to client
     ↓
  (Background) Leader: Send to followers
     ↓
  Followers: Apply when received (may be seconds later)

Guarantees:
  ✓ High Availability - returns immediately
  ✗ Eventual Consistency - followers lag behind leader
  ✗ Data loss risk - leader fails before replication
  ✓ CAP: Provides Availability + Partition Tolerance (AP)

Availability Benefits:
  If follower slow:
    Client doesn't wait → Fast response
    
  If network partition:
    Leader continues accepting writes
    Followers continue serving reads (stale data)
    System remains available in both zones

Visibility of Inconsistency:
```

### Impact on Availability

```
Scenario: 3-node cluster (Leader + 2 Followers)

SYNCHRONOUS REPLICATION (Strong Consistency):
────────────────────────────────────────

Failure 1: One Follower Down
  Leader: Needs ACK from F2 at minimum
  Result: Writes still proceed
  Availability: 100%

Failure 2: Both Followers Down
  Leader: Needs ACK from followers, but none available
  Result: Writes BLOCKED
  Availability: 0% for writes (reads OK on leader)

Failure 3: Network Partition
  Leader in Zone A, Followers in Zone B
  Leader: Cannot reach followers
  Result: Writes REJECTED (can't guarantee consistency)
  Availability: 0% (must choose C over A)

Total downtime: High (any follower failure impacts availability)


ASYNCHRONOUS REPLICATION (Eventual Consistency):
───────────────────────────────────────────────

Failure 1: One Follower Down
  Leader: Doesn't wait, sends to other follower
  Result: Writes proceed
  Availability: 100%

Failure 2: Both Followers Down
  Leader: Doesn't wait for followers
  Result: Writes proceed to leader (followers catch up later)
  Availability: 100% (but data loss if leader crashes immediately)

Failure 3: Network Partition
  Leader in Zone A: Continues accepting writes
  Followers in Zone B: Serve stale reads, lag behind
  Result: Both zones available
  Availability: 100% (sacrifices consistency)

Total downtime: Low (master failure only impacts availability)
```

### Leader Election and Availability

```
Scenario: Leader Fails

Synchronous Replication:
  Old Leader dies
  System must elect new leader from followers
  During election: NO writes possible (0% availability)
  After election: New leader elected, writes resume
  
  Election time: 10-30 seconds typical
  Business Impact: Downtime for all writes

Asynchronous Replication:
  Old Leader dies
  Followers detect via heartbeat
  New leader elected quickly
  During election: Followers continue serving reads (with stale data)
  After election: New leader available, writes resume
  
  Election time: 1-5 seconds typical
  Business Impact: Minimal, clients may see stale data briefly
  
  Trade-off: One write may be lost (that didn't replicate before crash)
```

---

## Part 5: Eventual Consistency

### What is Eventual Consistency?

**Eventual Consistency** is a weak consistency model that guarantees:
- If no new writes occur, all replicas will converge to the same value
- Temporary inconsistency allowed during propagation
- All writes eventually replicated to all nodes

```
Timeline:

Time 0: Write arrives
  Master: value = 100
  Replica 1: value = old_value
  Replica 2: value = old_value

Time 1-100ms: Replication in progress
  Master: value = 100
  Replica 1: value = 100 (updated)
  Replica 2: value = old_value (still waiting)
  
Result: INCONSISTENT (Master and R1 different from R2)

Time 101-200ms: More replication
  Master: value = 100
  Replica 1: value = 100
  Replica 2: value = 100 (finally updated)
  
Result: CONSISTENT (All have same value)

Guarantee: Eventually all replicas converge
           (if no new writes occur)
```

### Eventual Consistency in Practice

```
Real System with Continuous Writes:

Master:
  T0: Write v1=100 → propagate
  T10: Write v2=200 → propagate
  T20: Write v3=300 → propagate

Replica 1 (Network: 50ms latency):
  T0: v1=100 (received at T50)
  T10: v2=200 (received at T60)
  T20: v3=300 (received at T70)

Replica 2 (Network: 100ms latency):
  T0: v1=100 (received at T100)
  T10: v2=200 (received at T110)
  T20: v3=300 (received at T120)

After T120, all replicas consistent
But writes are sliding window - new writes keep arriving
Replicas perpetually "eventually" consistent (lag = network latency)
```

### Consistency Models Within Eventual Consistency

Even with eventual consistency, there are degrees of guarantees:

#### Read-Your-Own-Writes (RYOW)

```
Scenario: User updates profile picture

User: "Update profile photo = picture.jpg"
  └─ Write goes to Master
  
Master: Accepts write to local storage
  └─ Returns "Success" to user
  
User refreshes page (read):
  └─ Read from Replica 1 (which hasn't received update yet)
  └─ Gets old picture! ✗

RYOW Guarantee:
  User's own writes always visible to user
  
Implementation:
  After write to Master: Remember write
  User's next read: Read from Master (not follower)
  └─ Sees updated picture ✓

Trade-off: Some reads go to master (less load balancing)
          But: User always sees own changes
```

#### Causal Consistency

```
Scenario: Comments on a post

User A: Posts article → Post ID = 1, version = v1
User B: Sees post (reads from Replica - stale? no matter, sees v1)
User B: Comments on post
  └─ Comment: post_id=1, post_version=v1, text="Great!"
  └─ Write goes to Master

Causal Consistency:
  Related writes in causal order
  Comment MUST be stored with reference to Post v1
  When reading, follow causal chain:
    Read Post v1
    Read Comments referencing Post v1
  Never see comment without seeing post it references

Ensures: Application logic not broken by replication lag
```

#### Session Consistency

```
Scenario: User shopping cart

Session established: session_id=ABC123
User: Select iPhone ($1000)
  Write: cart_items[ABC123] = [iPhone]
  Write to Master

Same session: View cart
  Read from Replica (may be stale)
  Replica hasn't received write yet
  Returns: empty cart ✗

Session Consistency:
  All operations in same session go to same server
  User session_id=ABC123 always reads from same Replica
    (that processes user's writes)
  Never sees own writes disappear

In practice: Sticky sessions (client→same server mapping)
```

### Eventual Consistency Problems

#### Problem 1: Stale Read Anomaly

```
Scenario: Banking system

User checks account balance
  Read from Replica: balance = $1000

Meanwhile (unbeknownst to user):
  System processed withdrawal: balance -= $500
  Write to Master (not yet replicated to this Replica)

User attempts withdrawal: $800
  Read again from same Replica: balance = $1000 (stale!)
  System checks: balance >= $800? YES
  Withdrawal succeeds

Reality: Actual balance = $500
  Overdraft by $300
  Allowed stale read to break business logic

Solution:
  Critical reads go to Master only
  Trading performance for immediate consistency
```

#### Problem 2: Conflicting Concurrent Updates

```
Scenario: Document collaborative editing

Document "budget.txt": amount = $1000

Zone A (Developer A):
  Master receives: amount = $1200
  Updates Master to 1200

Zone B (Developer B):
  Replica receives different instruction: amount = $900
  Replica accepts it (autonomous zone)

When zones merge:
  Master: amount = $1200
  Replica: amount = $900
  ✗ CONFLICT!

Resolution strategies:
  1. Last-Write-Wins: Keep 1200 (Developer B's change lost)
  2. Vector Clocks: Mark both as concurrent
  3. Merge function: amount = (1200 + 900) / 2 = 1050
     (Domain-specific resolution)
```

#### Problem 3: Read-After-Write Inconsistency

```
Scenario: User updates profile

User: "Update name to Alice"
  Write to Master
  Master: name = "Alice"
  Returns to user

User immediately: "Read my profile"
  Reads from Replica (wrong replica!)
  Replica: name = "Bob" (old, not yet replicated)
  User confused: "I just changed it!" ✗

Prevention: 
  After write: Return "timestamp" of write
  Next read: "Read after timestamp T"
  Client: "I need to read data from after timestamp T"
  System: "Route to replica that's caught up to T"
```

### Monitoring Eventual Consistency

```
Metrics to track:

1. Replication Lag
   Master version: v100
   Replica version: v95
   Lag: 5 versions behind
   Time: ~50ms (depends on batch size)

2. Write Acknowledgment Latency
   Time from write to "success" response
   EC should be <10ms (return before replication)

3. Replica Sync Time
   How long until replicas consistent
   Should be predictable (network latency)

4. Conflict Resolution Rate
   How often conflicting writes occur
   Indicates partition tolerance is being tested

5. Read Staleness
   Max age of data re ad from replica
   Should be < replication lag
```

---

## Comparison: Strong vs Eventual Consistency

```
┌─────────────────────────────┬──────────────────────┬──────────────────────┐
│ Aspect                      │ Strong Consistency   │ Eventual Consistency │
├─────────────────────────────┼──────────────────────┼──────────────────────┤
│ Read Latency                │ High (wait for sync)  │ Low (return from any │
│                             │ 100ms+               │ replica) <10ms       │
├─────────────────────────────┼──────────────────────┼──────────────────────┤
│ Write Latency               │ V. High (sync all)   │ Low (return after    │
│                             │ 500ms+               │ master) <10ms        │
├─────────────────────────────┼──────────────────────┼──────────────────────┤
│ Availability on Partition   │ Degraded (blocked)   │ High (both zones up) │
├─────────────────────────────┼──────────────────────┼──────────────────────┤
│ Data Freshness              │ Always latest        │ Temporary lag        │
├─────────────────────────────┼──────────────────────┼──────────────────────┤
│ Data Loss Risk              │ None                 │ Possible if master   │
│                             │                      │ dies before replication
├─────────────────────────────┼──────────────────────┼──────────────────────┤
│ Conflict Handling           │ None (prevented)     │ LWW or app logic     │
├─────────────────────────────┼──────────────────────┼──────────────────────┤
│ Developer Complexity        │ Lower (ACID)         │ Higher (handle lag)  │
├─────────────────────────────┼──────────────────────┼──────────────────────┤
│ Use Cases                   │ Finance, Banking     │ Social, Analytics    │
│                             │ Transactions         │ E-commerce catalog   │
└─────────────────────────────┴──────────────────────┴──────────────────────┘
```

---

## Interview Tips

### Key Points to Remember

1. **CAP is about distributed systems**, ACID is about database transactions
2. **Partition Tolerance is mandatory** - real networks partition, so choose CP or AP
3. **Leader-Follower tradeoff**: Sync = consistency but less availability, Async = more availability but stale reads
4. **Eventual consistency acceptable for many systems** - knows constraints and pick right model
5. **Quorum writes/reads** - powerful technique to tune consistency/availability tradeoff
6. **Monitoring replication lag** - key metric for eventual consistency systems
7. **Idempotent operations** - critical for eventual consistency to handle retries

### Common Interview Questions

**Q: Should my system use strong or eventual consistency?**
```
Answer framework:
1. What are the business requirements?
   - Financial transaction? → Strong consistency (accuracy > speed)
   - User feed? → Eventual consistency (freshness ok, availability critical)
   
2. What's the acceptable staleness?
   - <1ms? → Strong consistency only
   - <1s? → Eventual with monitoring
   - <day? → Any approach works
   
3. What's the failure tolerance?
   - Can't lose ANY write? → Strong consistency
   - Can lose <0.1% writes? → Eventual consistency
   
4. What's the latency requirement?
   - <100ms? → Eventual consistency
   - <500ms? → Either (depends on other factors)
   - >500ms? → Strong consistency viable
```

**Q: How does leader-follower affect availability?**
```
Answer: 
With synchronous replication (strong consistency):
  - Follower failure → may block writes (lower availability)
  - Network partition → sacrifices A for C (lower availability)
  - Leader election → downtime (lower availability)
  
With asynchronous replication (eventual consistency):
  - Follower failure → no impact (high availability)
  - Network partition → both zones available (high availability)
  - Leader election → brief stale reads, no write downtime (high availability)
  
Trade-off is explicit: choose consistency or availability
```