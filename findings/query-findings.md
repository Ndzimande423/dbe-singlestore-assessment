# Findings and Analysis

## Assessment Requirement
> Compare and contrast the outputs of EXPLAIN, PROFILE, and DEBUG PROFILE. Document the observed differences and what they indicate about query performance.

---

## Finding 1 — Broadcast Join Confirmed

The join between `orders_good` (sharded rowstore) and `products` (reference columnstore) triggers a broadcast operation.

**Evidence from EXPLAIN:**
```
ColumnStoreScan assessment_db.products AS p, table_type:reference_columnstore
```

**Evidence from PROFILE:**
```
network_traffic: 0.112000 KB
```

**What this means:**
The reference table is replicated to every leaf node. When the query runs, each partition joins its local shard of `orders_good` with its local copy of `products` — no data moves across the network. The 0.112 KB network traffic is just the final result being sent back to the aggregator, not the join data itself.

---

## Finding 2 — Parallel Execution Across All Partitions

**Evidence from EXPLAIN:**
```
Gather partitions:all parallelism_level:partition
```

**What this means:**
The query runs simultaneously across all 16 partitions. The `Gather` operator at the top collects results from all partitions and merges them. This is the core performance advantage of SingleStore — work is distributed and parallelised automatically.

---

## Finding 3 — Columnstore Segment Elimination

**Evidence from PROFILE:**
```
segments_scanned: 5
segments_skipped: 11
segments_fully_contained: 0
```

**What this means:**
The columnstore for `products` has 16 segments total. 11 were skipped because the optimizer determined they could not contain rows matching `category = 'Electronics'`. Only 5 segments were actually read. This is columnstore pruning — a major performance feature that avoids reading irrelevant data entirely.

---

## Finding 4 — Compile Time Dominates First Execution

**Evidence from PROFILE:**
```
Compile Total Time: 23ms
Total query time: ~26ms
```

**What this means:**
88% of the total query time was spent compiling the query plan, not executing it. On the first run SingleStore compiles the query to native machine code. Subsequent executions reuse the cached compiled plan and will be significantly faster — typically under 5ms for this query size.

---

## Finding 5 — HashJoin Strategy

**Evidence from EXPLAIN:**
```
HashJoin
|---HashTableProbe [p.product_name = o.product]
|   HashTableBuild alias:o
```

**Evidence from PROFILE:**
```
HashTableBuild actual_rows:10 memory_usage:1,048.576050 KB
```

**What this means:**
SingleStore chose to build the hash table from `orders_good` (10 rows) and probe it with `products` (5 rows after filter). This is the correct choice — build from the larger side, probe with the smaller filtered side. The hash table used ~1MB of memory which is well within available RAM.

---

## Finding 6 — Histogram Warning

**Evidence from EXPLAIN:**
```
WARNING: Histograms have not been collected on the following columns.
ANALYZE TABLE assessment_db.`products` COLUMNS `category` ENABLE;
```

**What this means:**
The optimizer is estimating row counts without statistics. The estimated rows for the ColumnStoreScan was 4 (correct) but this was a lucky guess. On larger datasets without histograms, the optimizer may choose a suboptimal join order or strategy.

**Fix:**
```sql
ANALYZE TABLE products COLUMNS category ENABLE;
```

---

## Comparison: EXPLAIN vs PROFILE vs EXPLAIN EXTENDED vs DEBUG PROFILE

| Feature | EXPLAIN | EXPLAIN EXTENDED | PROFILE | DEBUG PROFILE |
|---|---|---|---|---|
| Purpose | Logical query plan | Physical SQL sent to leaves | Actual execution statistics | Node-level execution detail |
| Query executes | No | No | Yes | Yes |
| Row counts | Estimated | Estimated | Estimated + Actual | Estimated + Actual |
| Timing per operator | No | No | Yes | Yes |
| Memory usage | No | No | Yes | Yes |
| Network traffic | No | No | Yes | Yes |
| Leaf SQL shown | No | Yes — in query:[] | No | Yes |
| Segment scan stats | No | No | Yes | Yes |
| Broadcast confirmation | table_type:reference | STRAIGHT_JOIN shown | network_traffic:0.112KB | Full node breakdown |
| Supported in v8.1 | ✅ | ✅ | ✅ | ❌ |
| Best used for | Understanding structure | Distributed execution | Diagnosing bottlenecks | Deep node-level debug |

---

## Shard Key Distribution Comparison

### orders_good — Equal Distribution (order_id shard key)
```sql
SELECT order_id % 4 AS bucket, COUNT(*) FROM orders_good GROUP BY bucket;
```
```
+--------+----------+
| bucket | COUNT(*) |
+--------+----------+
|      0 |        2 |
|      1 |        3 |
|      2 |        3 |
|      3 |        2 |
+--------+----------+
```
Rows spread evenly — no hot partitions. All partitions share the workload equally.

### orders_bad — Data Skew (status shard key)
```sql
SELECT status, COUNT(*) FROM orders_bad GROUP BY status;
```
```
+-----------+----------+
| status    | COUNT(*) |
+-----------+----------+
| shipped   |        4 |
| delivered |        3 |
| pending   |        3 |
+-----------+----------+
```
All rows land in only 3 partition buckets. In a large dataset this creates severe hot partitions — some partitions handle millions of rows while others handle none. Queries slow down because they are bottlenecked by the overloaded partitions.

### Impact of Poor Shard Key at Scale

| Scenario | orders_good (order_id) | orders_bad (status) |
|---|---|---|
| 1 million rows | ~62,500 rows per partition | Up to 400,000 rows on one partition |
| Query parallelism | All 16 partitions work | Only 3 partitions work |
| Hot partition risk | None | High |
| Partition pruning | Yes — for order_id filters | No — must scan all partitions |
| Recommended for | High-cardinality keys | Never recommended |
