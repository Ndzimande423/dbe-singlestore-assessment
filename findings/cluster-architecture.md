# Cluster Architecture — Current vs Expanded

## Current Setup (What Is Running Now)

```
┌─────────────────────────────────────────────────────┐
│                   EC2 m6a.2xlarge                   │
│                                                     │
│   ┌─────────────────────────────────────────────┐   │
│   │     Master Aggregator (MA)  port 3306       │   │
│   │  - Receives all client queries              │   │
│   │  - Routes work to the leaf                  │   │
│   │  - Merges results back to client            │   │
│   └──────────────────┬──────────────────────────┘   │
│                      │                              │
│   ┌──────────────────▼──────────────────────────┐   │
│   │        Leaf Node 1  port 3307               │   │
│   │                                             │   │
│   │  assessment_db — 16 partitions              │   │
│   │  ┌──┬──┬──┬──┬──┬──┬──┬──┐                 │   │
│   │  │P0│P1│P2│P3│P4│P5│P6│P7│                 │   │
│   │  ├──┼──┼──┼──┼──┼──┼──┼──┤                 │   │
│   │  │P8│P9│PA│PB│PC│PD│PE│PF│                 │   │
│   │  └──┴──┴──┴──┴──┴──┴──┴──┘                 │   │
│   │  ALL 16 partitions on this one leaf         │   │
│   │                                             │   │
│   │  orders_good: 10 rows across 16 partitions  │   │
│   │  orders_bad:  10 rows across 3 partitions   │   │
│   │  products:    replicated (reference table)  │   │
│   └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

**How data is stored now:**
- All 16 partitions live on Leaf 1
- Every row in `orders_good` is hashed by `order_id` and assigned to one of the 16 partitions
- Every row in `orders_bad` is hashed by `status` — only 3 partitions ever get data
- `products` is replicated — a full copy exists on Leaf 1
- Queries run in parallel across all 16 partitions on the single leaf

---

## Expanded Setup (With Child Aggregator + Second Leaf)

This is what the cluster would look like with a proper self-managed license:

```
┌─────────────────────────────────────────────────────────────────┐
│                        EC2 m6a.2xlarge                          │
│                                                                 │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │            Master Aggregator (MA)  port 3306             │  │
│   │  - Entry point for all client connections                │  │
│   │  - Coordinates Child Aggregator                          │  │
│   │  - Knows about all leaf nodes                            │  │
│   └────────────────────┬─────────────────────────────────────┘  │
│                        │                                        │
│   ┌────────────────────▼─────────────────────────────────────┐  │
│   │           Child Aggregator (CA)  port 3308               │  │
│   │  - Handles additional client connections                  │  │
│   │  - Offloads query routing from MA                        │  │
│   │  - Distributes work to both leaf nodes                   │  │
│   └──────────────┬─────────────────────┬────────────────────┘  │
│                  │                     │                        │
│   ┌──────────────▼──────────┐  ┌───────▼─────────────────────┐  │
│   │   Leaf Node 1  port 3307│  │   Leaf Node 2  port 3309    │  │
│   │                         │  │                             │  │
│   │  Partitions: P0–P7      │  │  Partitions: P8–PF          │  │
│   │  ┌──┬──┬──┬──┬──┬──┬──┬──┐│  │  ┌──┬──┬──┬──┬──┬──┬──┬──┐│  │
│   │  │P0│P1│P2│P3│P4│P5│P6│P7││  │  │P8│P9│PA│PB│PC│PD│PE│PF││  │
│   │  └──┴──┴──┴──┴──┴──┴──┴──┘│  │  └──┴──┴──┴──┴──┴──┴──┴──┘│  │
│   │                         │  │                             │  │
│   │  orders_good: ~5 rows   │  │  orders_good: ~5 rows       │  │
│   │  orders_bad:  ~5 rows   │  │  orders_bad:  ~5 rows       │  │
│   │  products: full copy    │  │  products: full copy        │  │
│   └─────────────────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## How Data Would Be Distributed Across 2 Leaf Nodes

When a second leaf is added, SingleStore automatically rebalances — it moves partitions from Leaf 1 to Leaf 2 so each leaf owns half:

| Leaf | Partitions Owned | orders_good rows | orders_bad rows |
|---|---|---|---|
| Leaf 1 | P0, P1, P2, P3, P4, P5, P6, P7 | ~5 rows | depends on hash |
| Leaf 2 | P8, P9, PA, PB, PC, PD, PE, PF | ~5 rows | depends on hash |

**How rebalancing works:**
```sql
-- After adding Leaf 2, SingleStore automatically moves partitions
-- No manual intervention needed
-- Data moves in the background while the cluster stays online
ADD LEAF '172.17.0.4' IDENTIFIED BY 'Assessment@2025!';
-- SingleStore detects the new leaf and starts rebalancing automatically
```

---

## Current (1 Leaf) vs Expanded (2 Leaves) — Side by Side

| Aspect | Current — 1 Leaf | Expanded — 2 Leaves |
|---|---|---|
| Partitions per leaf | 16 | 8 each |
| Total partitions | 16 | 16 (same) |
| Parallel workers | 16 threads on 1 leaf | 8 threads × 2 leaves = 16 total |
| Storage capacity | Limited to 1 leaf disk | 2× storage capacity |
| Memory for data | Limited to 1 leaf RAM | 2× RAM available |
| Query throughput | Single leaf bottleneck | Both leaves work in parallel |
| Fault tolerance | None — 1 leaf fails = data lost | Leaf 1 fails = Leaf 2 still has its partitions |
| Write throughput | All writes go to 1 leaf | Writes split across 2 leaves |
| Aggregators | 1 MA | 1 MA + 1 CA — handles more client connections |

---

## What the Child Aggregator Adds

The Child Aggregator (CA) does not store data — it only routes queries. Adding one means:

```
Without CA:
All clients → MA → Leaf 1 + Leaf 2

With CA:
Some clients → MA  → Leaf 1 + Leaf 2
Other clients → CA → Leaf 1 + Leaf 2
```

- More client connections can be handled simultaneously
- MA is not a bottleneck for connection handling
- CA can be on a separate machine to distribute load
- If MA goes down, CA can still serve read queries

---

## How a Query Runs Across 2 Leaves

With the expanded cluster, the broadcast query would execute like this:

```
Client sends query to MA (or CA)
         │
         ▼
MA splits work: send to Leaf 1 AND Leaf 2 simultaneously
         │                    │
         ▼                    ▼
Leaf 1 scans P0–P7      Leaf 2 scans P8–PF
joins with local         joins with local
copy of products         copy of products
         │                    │
         └──────────┬─────────┘
                    ▼
            MA gathers results
            merges into final output
                    │
                    ▼
              Returns to client
```

**Key point:** Both leaves work at the same time. With 1 leaf the query takes X time. With 2 leaves it takes roughly X/2 time because the work is split.

---

## Commands to Add the Nodes (When License Available)

```sql
-- Step 1: Add child aggregator (must be a running SingleStore node)
ADD AGGREGATOR '172.17.0.3' IDENTIFIED BY 'Assessment@2025!';

-- Step 2: Add second leaf node
ADD LEAF '172.17.0.4' IDENTIFIED BY 'Assessment@2025!';

-- Step 3: Verify expanded cluster
SHOW AGGREGATORS;
SHOW LEAVES;

-- Step 4: Check partition rebalancing
SELECT * FROM information_schema.DISTRIBUTED_DATABASES 
WHERE DATABASE_NAME = 'assessment_db';
```

Expected output after expansion:
```
SHOW AGGREGATORS:
+-----------+------+--------+-------------------+
| Host      | Port | State  | Master_Aggregator |
+-----------+------+--------+-------------------+
| 127.0.0.1 | 3306 | online |                 1 |  ← MA
| 172.17.0.3| 3308 | online |                 0 |  ← CA
+-----------+------+--------+-------------------+

SHOW LEAVES:
+-----------+------+--------------------+--------+
| Host      | Port | Availability_Group | NodeId |
+-----------+------+--------------------+--------+
| 127.0.0.1 | 3307 |                  1 |      2 |  ← Leaf 1
| 172.17.0.4| 3309 |                  2 |      3 |  ← Leaf 2
+-----------+------+--------------------+--------+
```
