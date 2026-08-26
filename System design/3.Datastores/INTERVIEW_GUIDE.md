# System Design Datastores - Interview Guide

---

## 1. Bloom Filters

### What is a Bloom Filter?
A **Bloom filter** is a probabilistic data structure that efficiently tests whether an element belongs to a set. It uses minimal memory but allows false positives (never false negatives).

### Key Characteristics:
- **Time Complexity**: O(k) where k = number of hash functions
- **Space Complexity**: O(m) bits regardless of element count
- **False Positive Rate**: Tunable based on size and hash functions
- **False Negatives**: 0% (definite answer if element is NOT in set)

### How It Works:
```
Initial State: [0][0][0][0][0][0][0][0]

Add "apple":
- Hash1("apple") = 1 → set bit[1]
- Hash2("apple") = 3 → set bit[3]
- Hash3("apple") = 6 → set bit[6]
Result: [0][1][0][1][0][0][1][0]

Add "banana":
- Hash1("banana") = 2 → set bit[2]
- Hash2("banana") = 4 → set bit[4]
- Hash3("banana") = 7 → set bit[7]
Result: [0][1][1][1][1][0][1][1]

Query "apple":
- Check bit[1], bit[3], bit[6] → All SET → "Probably in set" ✓

Query "orange":
- Check bits required → Some NOT set → "Definitely not in set" ✓
```

### Multi-Layer Bloom Filters:
Stack multiple bloom filters to manage tradeoffs:
- **Layer 1 (Small)**: Quick false-positive check
- **Layer 2 (Medium)**: Secondary filtering
- **Layer 3 (Large)**: Final verification
This reduces queries to expensive resources.

### Real-Time Use Cases:
1. **Cache Checking**: Check if URL exists before cache lookup
2. **DB Queries**: Avoid expensive disk reads for non-existent records
3. **Spellchecking**: Quick rejection of misspelled words
4. **CDN Networks**: Identify which edge servers have content

### Interview Answer Template:
"Bloom filters are probabilistic data structures that sacrifice accuracy for speed and space. They guarantee that if an element is NOT in the set, it's definitely not there. But if it appears to be in the set, we need verification. This makes them perfect for caching layers where false positives are acceptable but false negatives are not."

---

## 2. Data Replication

### Master-Slave Architecture

```
Master (Write Authority)
    ↓
    ├─→ Replication Log
    │
    ├─→ Slave 1 (Read Replica)
    ├─→ Slave 2 (Read Replica)
    └─→ Slave N (Read Replica)
```

### Synchronous Replication

**How It Works:**
```
Client Request → Master → Write to Disk → Replicate to Slaves
                                ↓
                        Wait for ACK from ALL slaves
                                ↓
                        Response to Client
```

**Pros:**
- Strong consistency (all replicas always in sync)
- No data loss
- ACID guarantees maintained

**Cons:**
- High latency (slowest replica determines speed)
- Reduced availability (one slow replica affects all)
- Network partitions cause blocking

### Asynchronous Replication

**How It Works:**
```
Client Request → Master → Write to Disk → Immediate ACK to Client
                                ↓
                    Background: Replicate to Slaves (eventual consistency)
```

**Pros:**
- Low latency (master doesn't wait)
- High availability
- Better throughput

**Cons:**
- Potential data loss (master fails before replication)
- Eventual consistency (temporary inconsistency)
- Slave lag (reads might be stale)

### Peer-to-Peer Data Transfer

Instead of star topology (all replicate from master):

```
Traditional (Star):
    Master
    ↙  ↓  ↘
   S1  S2  S3

P2P Topology:
    Master
    ↙  ↓  ↘
   S1 ←→ S2
    ↓  ↙  ↓
    ↓     S3
   S1 and S2 can sync with each other
```

**Benefits:**
- Reduced load on master
- Faster replication
- Better fault tolerance

### Split-Brain Problem

**Scenario:**
```
Network Partition occurs

Before:
Master (authority for writes) ← → Replica A
                              ← → Replica B

After Partition:
Master (authority)   |   Network Gap   |   Replica A (acts as master?)
      ↓              |                  |      ↓
   Writes data       |                  |   Conflicting writes?
```

**Consequences:**
- Client A writes to Master
- Client B writes to Replica A (now acting as master)
- When network heals → conflicting data versions
- Loss of single source of truth

**Solutions:**
1. **Quorum-based Consensus**: Majority must agree
2. **Heartbeat**: Master must frequently confirm authority
3. **Automatic Failover**: Only one master alive at a time
4. **Conflict Resolution**: Last-write-wins or merge logic

---

## 3. Writing in Database

### Request Condensing

**Problem:** Every write request hitting disk is slow

```
Requests (Writes)       Condensed Batch
├─ Write A              ├─→ Combined Write Operation
├─ Write B              │   [A, B, C, D]
├─ Write C      →       │
├─ Write D              └─→ Single Disk I/O
├─ Write E
└─ Write F              ├─→ Next Batch
                        │   [E, F, ...]
                        └─→ Single Disk I/O
```

**Benefits:**
- Reduces disk I/O operations
- Better throughput
- Amortized latency

### Write-Ahead Logging (WAL) with Linked List

```
Memory Buffer (LinkedList)
┌─────────────────────────────────────┐
│ Node1 → Node2 → Node3 → Node4 → null│
│(Write A)  (Write B)  (Write C)      │
└─────────────────────────────────────┘
           ↓
      Disk Log File
    (Sequential Write)
         ↓
    Persistent Storage
```

**Why Linked List?**
- O(1) append operations
- Easy to track latest state
- Simple crash recovery (replay the log)

### B+ Tree vs Linked List

| Aspect | B+ Tree | Linked List |
|--------|---------|------------|
| **Search Time** | O(log n) | O(n) |
| **Range Query** | O(log n + k) | O(n) |
| **Insert** | O(log n) | O(1) amortized |
| **Space** | Balanced | Linear |
| **Disk Efficiency** | Excellent (fewer seeks) | Poor (scattered) |

**Use Case:**
- **Linked List**: Quick sequential logging (WAL)
- **B+ Tree**: Efficient indexed searches and range queries

### Sorted String Table (SST)

```
Write Path:
1. Write to MemTable (in-memory, sorted by key)
2. When MemTable full → Flush to SST file on disk
   
SST File Structure:
┌────────────────────────────────────┐
│ [Key, Value] pairs sorted by Key   │
│                                    │
│ Key001: Value_A                    │
│ Key051: Value_B                    │
│ Key100: Value_C                    │
│ Key200: Value_D                    │
└────────────────────────────────────┘

Benefits:
- Sequential disk writes (fast)
- Sorted data (enables binary search)
- Immutable once written (consistency)
```

### Compaction & Merging Chunks of SST

**Problem:** Multiple SST files → slow reads

```
Before Compaction:
SST1: [Key1-10]
SST2: [Key5-15]
SST3: [Key10-20]
Query Key = 12 → Need to check ALL 3 files

After Compaction (Merge):
SST_merged: [Key1-20]
Query Key = 12 → Check ONE file
```

**Levels Approach (LSM Tree):**
```
Level 0:  [SST1] [SST2] [SST3]  (can overlap)
            ↓
Level 1:  [SST_L1_A] [SST_L1_B]  (no overlap)
            ↓
Level 2:  [SST_L2_A]  (larger files)
```

**Process:**
1. Compact Level 0 into Level 1 (merge overlapping files)
2. Compact Level 1 into Level 2 (when Level 1 exceeds threshold)
3. Continue as needed

**Benefits:**
- Faster reads (fewer files to check)
- Eliminates redundant updates (old values removed)
- Space optimization (duplicate keys consolidated)

### Bloom Filters in DB Reading/Writing

**Reading optimization:**
```
Query Key="X"

Step 1: Check Bloom Filter
        ↓
   "Definitely NOT in this SST?" → Skip this file
   "Might be in this SST?" → Check this file

Step 2: If passes Bloom Filter
        ↓
   Binary search in SST file
```

**Writing optimization:**
```
New data arrives → Insert into MemTable
                    ↓
                 Bloom Filter updated
                    ↓
             When flushing to SST
                    ↓
         Attach Bloom Filter to SST
```

**Impact:** Avoids unnecessary disk reads for non-existent keys

---

## 4. Location-Based Database

### Problem with 2D Representation

**Naive Approach:**
```
Simple grid division:
┌─────┬─────┬─────┐
│ A   │ B   │ C   │  Lat: 40-50N
├─────┼─────┼─────┤
│ D   │ E   │ F   │  Lat: 30-40N
├─────┼─────┼─────┤
│ G   │ H   │ I   │  Lat: 20-30N
└─────┴─────┴─────┘
Lon: 70W-80W  etc

Query: "Find restaurants near (35.5N, 75W)"
→ Need to check cells around it
→ Even checking 9 cells requires multiple lookups
```

**Drawback:** 
- Not balanced (hot zones like city centers have many points)
- Linear search required for range queries
- Poor cache locality

### Quad Trees

```
Root Node (represents entire map)
         ↓
    ┌────┼────┐
    │    │    │    (divide into 4 quadrants)
   NW   NE   SW   SE
    │    │    │    │
   (if node has >M points, subdivide further)

Visual:
        [0, 100]
          ↙↗↖↘
        ┌─┼─┐
        │ │ │  Each quadrant can be further divided
        ├─┼─┤
        │ │ │
        └─┴─┘

Query: Find points near (35, 75)
→ Navigate to relevant quadrant
→ Search only within that branch
→ Time: O(log n) instead of O(n)
```

**Advantages:**
- Balanced tree for dense regions
- Efficient range queries
- Optimal for 2D space

### Fractal Mapping (2D to 1D)

**Problem:** Quad trees still require multi-dimensional operations

**Solution:** Hilbert Curve - Map 2D coordinates to 1D while preserving locality

```
2D Space:              Hilbert Curve (1D):
(0,1)─(1,1)            0─1
  │     │               │
  └─────┘               2─3

(0,0)─(1,0)

Visual progression:
Start at (0,0) → move to (1,0) → (1,1) → (0,1)
These become: 0 → 1 → 3 → 2 on 1D line

Range Query: Find points in box [0-1, 0-1]
→ Hilbert curve maps this to segments [0-2] on 1D line
→ Can use 1D indexing (B+ Tree) on these coordinates
→ Much faster than 2D tree traversal!
```

### Range Queries

**Example:** "Find all restaurants in area [35-36N, 74-75W]"

**Approach:**
```
1. Convert 2D box to Hilbert coordinates
   ↓
2. Get min and max Hilbert values in range
   Range: [h_min, h_max]
   
3. These map to continuous segments on 1D line
   
4. Use B+ Tree to find all keys in [h_min, h_max]
   
5. Verify actual 2D coordinates (some false positives possible)
   
Result: All restaurants in the geographic box
```

**Complexity:**
- Build: O(n log n)
- Range Query: O(log n + k) where k = results returned
- Update: O(log n)

---

## Quick Interview Comparison Table

| Concept | Use Case | Time Complexity | Key Advantage |
|---------|----------|-----------------|---------------|
| **Bloom Filter** | Cache invalidation, DB lookups | O(k) | Space-efficient, no false negatives |
| **Master-Slave** | High availability, read scaling | - | Enables read replicas |
| **Sync Replication** | Financial systems | Slow | Strong consistency |
| **Async Replication** | Social media feeds | Fast | High availability |
| **SST + LSM** | Write-heavy DB (RocksDB, Cassandra) | O(log n) reads | Optimized write throughput |
| **Bloom in DB** | Secondary index optimization | O(1) check | Prevents unnecessary I/O |
| **Hilbert Curve** | Geo-location queries | O(log n) | Preserves 2D locality in 1D |
| **Quad Tree** | Spatial indexing | O(log n) | Balanced for dense regions |

