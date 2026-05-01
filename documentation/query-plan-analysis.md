# Query Execution and Plan Analysis

## Broadcast Query Scenario

A broadcast operation occurs in SingleStore when a sharded table is joined with a reference table. The reference table (`products`) is replicated to all leaf nodes, so when a query joins `orders_good` (sharded) with `products` (reference), SingleStore broadcasts the reference table data locally on each partition — no network shuffle required.

---

## The Query

```sql
SELECT o.order_id, o.product, o.amount, p.category
FROM orders_good o
JOIN products p ON o.product = p.product_name
WHERE p.category = 'Electronics';
```

### Query Result
```
+----------+---------+--------+-------------+
| order_id | product | amount | category    |
+----------+---------+--------+-------------+
|        2 | Phone   | 499.99 | Electronics |
|        1 | Laptop  | 999.99 | Electronics |
|        3 | Tablet  | 299.99 | Electronics |
|        4 | Monitor | 399.99 | Electronics |
+----------+---------+--------+-------------+
4 rows in set (0.08 sec)
```

4 out of 10 orders matched the `Electronics` category filter.

---

## EXPLAIN Plan

```sql
EXPLAIN SELECT o.order_id, o.product, o.amount, p.category
FROM orders_good o
JOIN products p ON o.product = p.product_name
WHERE p.category = 'Electronics';
```

Output:
```
+---------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                               |
+---------------------------------------------------------------------------------------------------------------------------------------+
| WARNING: Histograms have not been collected on the following columns. Consider running the following commands to collect them now:    |
|     ANALYZE TABLE assessment_db.`products` COLUMNS `category` ENABLE;                                                                |
|                                                                                                                                       |
| Gather partitions:all est_rows:4 alias:remote_0 parallelism_level:partition                                                          |
| Project [o.order_id, o.product, o.amount, p.category] est_rows:4                                                                     |
| HashJoin                                                                                                                              |
| |---HashTableProbe [p.product_name = o.product]                                                                                      |
| |   HashTableBuild alias:o                                                                                                            |
| |   Project [o_0.order_id, o_0.product, o_0.amount] est_rows:10                                                                      |
| |   TableScan assessment_db.orders_good AS o_0 table_type:sharded_rowstore est_table_rows:10 est_filtered:10                         |
| ColumnStoreFilter [p.category = 'Electronics']                                                                                       |
| ColumnStoreScan assessment_db.products AS p, SORT KEY __UNORDERED () table_type:reference_columnstore est_table_rows:5 est_filtered:4|
+---------------------------------------------------------------------------------------------------------------------------------------+
14 rows in set (0.00 sec)
```

### EXPLAIN Analysis

| Operator | What it means |
|---|---|
| `Gather partitions:all` | Aggregator collects results from ALL partitions on the leaf node — this is the broadcast gather step |
| `parallelism_level:partition` | Query runs in parallel across all 16 partitions |
| `Project` | Selects only the required columns — order_id, product, amount, category |
| `HashJoin` | Join method used — builds a hash table from one side, probes with the other |
| `HashTableBuild alias:o` | Builds hash table from `orders_good` rows |
| `HashTableProbe` | Probes the hash table using `products.product_name = orders_good.product` |
| `TableScan orders_good` | Full scan of the sharded rowstore table — 10 rows estimated and filtered |
| `ColumnStoreScan products` | Scans the reference table (stored as columnstore) — 5 rows, 4 pass the filter |
| `ColumnStoreFilter` | Applies `category = 'Electronics'` filter on the reference table |
| `table_type:reference_columnstore` | Confirms `products` is a reference table — broadcast join triggered |

**Key observation:** The `table_type:reference_columnstore` on the products scan confirms the broadcast operation. The reference table is available on every partition locally — no data needs to be shuffled across the network.

**Warning noted:** Histograms not collected on `products.category`. This means the optimizer is estimating row counts without statistics. Fixed by running:
```sql
ANALYZE TABLE products COLUMNS category ENABLE;
```

---

## EXPLAIN EXTENDED

```sql
EXPLAIN EXTENDED SELECT o.order_id, o.product, o.amount, p.category
FROM orders_good o
JOIN products p ON o.product = p.product_name
WHERE p.category = 'Electronics';
```

Output:
```
+---------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                           |
+---------------------------------------------------------------------------------------------------------------------------------------------------+
| Gather partitions:all est_rows:4                                                                                                                  |
|   query:[SELECT STRAIGHT_JOIN `o`.`order_id`, `o`.`product`, `o`.`amount`, `p`.`category`                                                        |
|   FROM `assessment_db`.`products` as `p`                                                                                                          |
|   STRAIGHT_JOIN (SELECT WITH(NO_MERGE_THIS_SELECT=1) `o_0`.`order_id`, `o_0`.`product`, `o_0`.`amount`                                           |
|   FROM `assessment_db_0`.`orders_good` as `o_0`) AS `o`                                                                                          |
|   WITH (gen_min_max = TRUE, gen_spill = TRUE)                                                                                                     |
|   WHERE ((`p`.`category` = 'Electronics') AND (`o`.`product` = `p`.`product_name`))                                                              |
|   OPTION(NO_QUERY_REWRITE=1, INTERPRETER_MODE=INTERPRET_FIRST)]                                                                                   |
|   alias:remote_0 parallelism_level:partition                                                                                                      |
| Project [o.order_id, o.product, o.amount, p.category] est_rows:4                                                                                 |
| HashJoin                                                                                                                                          |
| |---HashTableProbe [p.product_name = o.product]                                                                                                   |
| |   HashTableBuild alias:o                                                                                                                        |
| |   Project [o_0.order_id, o_0.product, o_0.amount] est_rows:10                                                                                  |
| |   TableScan assessment_db.orders_good AS o_0 table_type:sharded_rowstore est_table_rows:10 est_filtered:10                                      |
| ColumnStoreFilter [p.category = 'Electronics']                                                                                                    |
| ColumnStoreScan assessment_db.products AS p table_type:reference_columnstore est_table_rows:5 est_filtered:4                                      |
+---------------------------------------------------------------------------------------------------------------------------------------------------+
9 rows in set (0.00 sec)
```

### EXPLAIN EXTENDED vs EXPLAIN

EXPLAIN EXTENDED shows the actual SQL that SingleStore sends to the leaf nodes inside the `query:[]` block. Key differences:

- Shows `STRAIGHT_JOIN` — SingleStore controls the join order explicitly
- Shows `assessment_db_0.orders_good` — the internal partition-level database name
- Shows `WITH (gen_min_max = TRUE, gen_spill = TRUE)` — optimizer hints for min/max generation and spill-to-disk support
- Shows `OPTION(NO_QUERY_REWRITE=1, INTERPRETER_MODE=INTERPRET_FIRST)` — execution mode hints sent to the leaf

---

## PROFILE Output

```sql
PROFILE SELECT o.order_id, o.product, o.amount, p.category
FROM orders_good o
JOIN products p ON o.product = p.product_name
WHERE p.category = 'Electronics';
```

Then:
```sql
SHOW PROFILE;
```

Output:
```
+----------------------------------------------------------------------------------------------------------------------------------+
| PROFILE                                                                                                                          |
+----------------------------------------------------------------------------------------------------------------------------------+
| Gather partitions:all est_rows:4 alias:remote_0 parallelism_level:partition                                                     |
|   exec_time:0ms start_time:00:00:00.000 end_time:00:00:00.026                                                                   |
|   network_traffic:0.112000 KB network_time:0ms actual_rows:4                                                                    |
|                                                                                                                                  |
| Project [o.order_id, o.product, o.amount, p.category] est_rows:4 actual_rows:4                                                  |
|   exec_time:0ms start_time:00:00:00.026 network_traffic:0.112000 KB network_time:0ms                                            |
|                                                                                                                                  |
| HashJoin actual_rows:4 exec_time:0ms start_time:00:00:00.026                                                                    |
|                                                                                                                                  |
| |---HashTableProbe [p.product_name = o.product]                                                                                 |
|                                                                                                                                  |
| |   HashTableBuild alias:o actual_rows:10 exec_time:0ms                                                                         |
|       start_time:[00:00:00.025, 00:00:00.026] memory_usage:1,048.576050 KB                                                      |
|                                                                                                                                  |
| |   Project [o_0.order_id, o_0.product, o_0.amount] est_rows:10 actual_rows:10                                                  |
|       exec_time:1ms start_time:[00:00:00.025, 00:00:00.026]                                                                     |
|                                                                                                                                  |
| |   TableScan assessment_db.orders_good AS o_0 table_type:sharded_rowstore                                                      |
|       est_table_rows:10 est_filtered:10 actual_rows:10                                                                          |
|       exec_time:0ms start_time:[00:00:00.025, 00:00:00.026]                                                                     |
|                                                                                                                                  |
| ColumnStoreFilter [p.category = ?] actual_rows:20 exec_time:0ms start_time:00:00:00.026                                         |
|   total_rows_in:25 average_filters_per_row:1.000000                                                                             |
|   average_index_filters_per_row:0.000000 average_bloom_filters_per_row:0.000000                                                 |
|                                                                                                                                  |
| ColumnStoreScan assessment_db.products AS p table_type:reference_columnstore                                                    |
|   est_table_rows:5 est_filtered:4 actual_rows:25                                                                                |
|   exec_time:0ms start_time:[00:00:00.025, 00:00:00.026]                                                                        |
|   memory_usage:2,097.152100 KB segments_scanned:5 segments_skipped:11 segments_fully_contained:0                               |
|                                                                                                                                  |
| Compile Total Time: 23ms                                                                                                        |
+----------------------------------------------------------------------------------------------------------------------------------+
10 rows in set (0.00 sec)
```

### PROFILE Analysis

| Metric | Value | What it means |
|---|---|---|
| Total query time | ~26ms | End-to-end execution time |
| Compile time | 23ms | Most time spent compiling the query plan — first execution overhead |
| Gather network_traffic | 0.112 KB | Tiny — reference table broadcast is local, no heavy network transfer |
| HashTableBuild memory | 1,048 KB (~1MB) | Memory used to build hash table from orders_good |
| ColumnStoreScan memory | 2,097 KB (~2MB) | Memory used to scan the reference table |
| TableScan actual_rows | 10 | All 10 orders_good rows scanned |
| ColumnStoreScan actual_rows | 25 | 25 rows scanned across segments (5 rows × 5 segments) |
| ColumnStoreFilter actual_rows | 20 | 20 rows passed to filter (before category check) |
| Final actual_rows | 4 | 4 rows returned — matched Electronics filter |
| segments_scanned | 5 | 5 column segments read |
| segments_skipped | 11 | 11 segments skipped — columnstore pruning working |

**Key observation:** `segments_skipped:11` shows the columnstore is efficiently skipping segments that cannot contain matching rows. This is columnstore segment elimination — a major performance feature.

**Key observation:** `network_traffic:0.112 KB` is extremely low. This confirms the broadcast join is working correctly — the reference table data is local on each partition, so no significant data moves over the network.

---

## DEBUG PROFILE

`DEBUG PROFILE` syntax is not supported in SingleStore version 8.1. The command was attempted:

```sql
DEBUG PROFILE SELECT o.order_id, o.product, o.amount, p.category
FROM orders_good o
JOIN products p ON o.product = p.product_name
WHERE p.category = 'Electronics';
```

Result:
```
ERROR 1064 (42000): You have an error in your SQL syntax near 'DEBUG PROFILE SELECT'
```

`DEBUG PROFILE` was introduced in later versions of SingleStore. In version 8.1, the equivalent detailed profiling is obtained through `SHOW PROFILE` after running `PROFILE`, which provides the same operator-level execution statistics including timing, memory usage, row counts and segment scan details.

---

## Comparison: EXPLAIN vs PROFILE vs EXPLAIN EXTENDED

| Feature | EXPLAIN | PROFILE | EXPLAIN EXTENDED |
|---|---|---|---|
| Purpose | Shows the logical query plan before execution | Shows actual execution statistics after running | Shows the physical SQL sent to leaf nodes |
| Execution required | No — plan only | Yes — query runs | No — plan only |
| Row counts | Estimated only | Both estimated and actual | Estimated only |
| Timing | Not shown | exec_time per operator | Not shown |
| Memory usage | Not shown | memory_usage per operator | Not shown |
| Network traffic | Not shown | network_traffic shown | Not shown |
| Leaf-level SQL | Not shown | Not shown | Shown in query:[] block |
| Broadcast confirmation | table_type:reference_columnstore | network_traffic:0.112 KB | STRAIGHT_JOIN with reference table |
| Best used for | Understanding query structure | Diagnosing performance bottlenecks | Understanding distributed execution |

---

## Key Findings

1. **Broadcast join confirmed** — `products` is scanned as `table_type:reference_columnstore` meaning it is replicated to all partitions. The join happens locally on each partition without network shuffle.

2. **Parallel execution** — `parallelism_level:partition` means the query runs across all 16 partitions simultaneously.

3. **Columnstore segment elimination** — 11 out of 16 segments were skipped during the products scan, showing the columnstore is efficiently pruning irrelevant data.

4. **Compile time dominates** — 23ms of the 26ms total was compilation. On repeated executions the compiled plan is cached and subsequent runs will be significantly faster.

5. **Low network overhead** — only 0.112 KB transferred, confirming the broadcast join design is optimal for this query pattern.

6. **Histogram warning** — the optimizer warned that statistics had not been collected on `products.category`. Running `ANALYZE TABLE products COLUMNS category ENABLE` improves the optimizer's row count estimates.
