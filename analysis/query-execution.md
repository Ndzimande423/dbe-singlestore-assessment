# Query Execution and Plan Analysis

## Assessment Requirement
> Design a scenario to demonstrate a Broadcast query plan. Execute a query that triggers a broadcast operation. Capture EXPLAIN, PROFILE and DEBUG PROFILE outputs.

---

## Broadcast Query Scenario

A broadcast operation occurs when a sharded table is joined with a reference table. The reference table (`products`) is replicated to all leaf nodes. When `orders_good` (sharded) joins `products` (reference), SingleStore executes the join locally on each partition — no data needs to move across the network.

### The Query

```sql
SELECT o.order_id, o.product, o.amount, p.category
FROM orders_good o
JOIN products p ON o.product = p.product_name
WHERE p.category = 'Electronics';
```

### Result
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

---

## EXPLAIN Output

```sql
EXPLAIN SELECT o.order_id, o.product, o.amount, p.category
FROM orders_good o
JOIN products p ON o.product = p.product_name
WHERE p.category = 'Electronics';
```

```
+---------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                               |
+---------------------------------------------------------------------------------------------------------------------------------------+
| WARNING: Histograms have not been collected on the following columns:                                                                 |
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
| ColumnStoreScan assessment_db.products AS p table_type:reference_columnstore est_table_rows:5 est_filtered:4                         |
+---------------------------------------------------------------------------------------------------------------------------------------+
```

---

## EXPLAIN EXTENDED Output

```sql
EXPLAIN EXTENDED SELECT o.order_id, o.product, o.amount, p.category
FROM orders_good o
JOIN products p ON o.product = p.product_name
WHERE p.category = 'Electronics';
```

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
```

---

## PROFILE Output

```sql
PROFILE SELECT o.order_id, o.product, o.amount, p.category
FROM orders_good o
JOIN products p ON o.product = p.product_name
WHERE p.category = 'Electronics';

SHOW PROFILE;
```

```
+----------------------------------------------------------------------------------------------------------------------------------+
| PROFILE                                                                                                                          |
+----------------------------------------------------------------------------------------------------------------------------------+
| Gather partitions:all est_rows:4 alias:remote_0 parallelism_level:partition                                                     |
|   exec_time:0ms start_time:00:00:00.000 end_time:00:00:00.026                                                                   |
|   network_traffic:0.112000 KB network_time:0ms actual_rows:4                                                                    |
| Project [o.order_id, o.product, o.amount, p.category] est_rows:4 actual_rows:4                                                  |
|   exec_time:0ms start_time:00:00:00.026 network_traffic:0.112000 KB network_time:0ms                                            |
| HashJoin actual_rows:4 exec_time:0ms start_time:00:00:00.026                                                                    |
| |---HashTableProbe [p.product_name = o.product]                                                                                 |
| |   HashTableBuild alias:o actual_rows:10 exec_time:0ms                                                                         |
|       start_time:[00:00:00.025, 00:00:00.026] memory_usage:1,048.576050 KB                                                      |
| |   Project [o_0.order_id, o_0.product, o_0.amount] est_rows:10 actual_rows:10                                                  |
|       exec_time:1ms start_time:[00:00:00.025, 00:00:00.026]                                                                     |
| |   TableScan assessment_db.orders_good AS o_0 table_type:sharded_rowstore                                                      |
|       est_table_rows:10 est_filtered:10 actual_rows:10                                                                          |
|       exec_time:0ms start_time:[00:00:00.025, 00:00:00.026]                                                                     |
| ColumnStoreFilter [p.category = ?] actual_rows:20 exec_time:0ms                                                                 |
|   total_rows_in:25 average_filters_per_row:1.000000                                                                             |
|   average_index_filters_per_row:0.000000 average_bloom_filters_per_row:0.000000                                                 |
| ColumnStoreScan assessment_db.products AS p table_type:reference_columnstore                                                    |
|   est_table_rows:5 est_filtered:4 actual_rows:25                                                                                |
|   exec_time:0ms start_time:[00:00:00.025, 00:00:00.026]                                                                        |
|   memory_usage:2,097.152100 KB segments_scanned:5 segments_skipped:11 segments_fully_contained:0                               |
| Compile Total Time: 23ms                                                                                                        |
+----------------------------------------------------------------------------------------------------------------------------------+
```

---

## DEBUG PROFILE — SHOW PROFILE JSON (Equivalent)

`DEBUG PROFILE` syntax is not supported in SingleStore version 8.1:

```sql
DEBUG PROFILE SELECT o.order_id, o.product, o.amount, p.category
FROM orders_good o
JOIN products p ON o.product = p.product_name
WHERE p.category = 'Electronics';
```
```
ERROR 1064 (42000): You have an error in your SQL syntax near 'DEBUG PROFILE SELECT'
```

However `SHOW PROFILE JSON` was successfully used as the equivalent — it provides the same per-partition breakdown that DEBUG PROFILE would show in newer versions:

```sql
PROFILE SELECT o.order_id, o.product, o.amount, p.category
FROM orders_good o
JOIN products p ON o.product = p.product_name
WHERE p.category = 'Electronics';

SHOW PROFILE JSON;
```

Key metrics extracted from the JSON output:

| Metric | Total | Avg per partition | Max partition |
|---|---|---|---|
| Gather network_traffic | 112 bytes | 7 bytes | 29 bytes (P7) |
| Project actual_rows | 4 | 0.25 | 1 (P6) |
| HashTableBuild actual_rows | 10 | 0.625 | 2 (P7) |
| HashTableBuild memory_usage | 1,048,576 bytes | 65,536 bytes | 131,072 bytes (P2) |
| ColumnStoreScan actual_rows | 25 | 1.5625 | 5 (P3) |
| ColumnStoreScan memory_usage | 2,097,152 bytes | 131,072 bytes | 131,072 bytes |
| segments_scanned | 5 | 0.3125 | 1 (P3) |
| segments_skipped | 11 | 0.6875 | 1 |

Additional details from the JSON output:
- SingleStore version confirmed: `8.1.54`
- Total runtime: `4ms`
- Aggregator node: `127.0.0.1`
- Online leaves: `1`
- Online aggregators: `1`
- Query compile time: `0ms` (cached plan reused from previous run)
- Optimizer memory used: `102,576 bytes` (optree) + `31,808 bytes` (dstree)

**What SHOW PROFILE JSON reveals beyond regular SHOW PROFILE:**
- Per-partition statistics with `avg`, `stddev`, `max` and `maxPartition` for every operator
- Which specific partition handled the most work (e.g. P3 handled the most columnstore rows, P7 handled the most orders)
- Memory allocation broken down per partition
- Compile time broken down into individual optimizer phases
- The full leaf-level SQL query sent to partitions
- Confirms the plan was served from cache (`mpl_path` shown)

