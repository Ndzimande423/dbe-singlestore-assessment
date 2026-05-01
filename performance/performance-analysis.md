# Performance Analysis

## Assessment Requirement
> Document the observed differences and what they indicate about query performance.

---

## Query Performance Summary

| Metric | Value |
|---|---|
| Total execution time | 26ms |
| Compile time | 23ms (88% of total) |
| Actual execution time | ~3ms |
| Rows scanned — orders_good | 10 |
| Rows scanned — products | 25 (across 5 segments) |
| Segments skipped | 11 of 16 |
| Network traffic | 0.112 KB |
| Hash table memory | ~1 MB |
| ColumnStore scan memory | ~2 MB |
| Final rows returned | 4 |

---

## Execution Timeline

```
00:00:00.000  Gather starts — waiting for leaf results
00:00:00.025  Leaf starts: TableScan orders_good + ColumnStoreScan products (parallel)
00:00:00.026  HashTableBuild complete (10 rows from orders_good)
00:00:00.026  HashTableProbe complete (4 matches found)
00:00:00.026  Project + Gather complete — results sent to aggregator
```

Total leaf execution: ~1ms
Network + aggregation: ~25ms (dominated by compile on first run)

---

## Broadcast Join Performance

The broadcast join pattern is optimal for this query because:

- `products` is small (5 rows) — cheap to replicate to all nodes
- No data shuffle required — join happens locally on each partition
- Network traffic is near zero (0.112 KB) — only the final 4 result rows travel the network
- Scales well — as `orders_good` grows, each partition still joins locally with its local copy of `products`

**When broadcast joins are NOT optimal:**
- When the reference table is very large (millions of rows) — replication cost becomes high
- When the reference table changes frequently — replication overhead on every write

---

## Columnstore Performance

The `products` reference table is stored as a columnstore internally. This provides:

- **Segment elimination:** 11 of 16 segments skipped — only 31% of data read
- **Column pruning:** Only `product_name` and `category` columns read — not all columns
- **Compression:** Columnstore compresses data significantly reducing I/O

---

## Shard Key Performance Impact

| Metric | orders_good (order_id) | orders_bad (status) |
|---|---|---|
| Partition distribution | Even across 16 partitions | Skewed to 3 buckets |
| Parallel workers | 16 | 3 effective |
| Hot partition risk | None | High |
| Partition pruning | Yes | No |
| Performance at scale | Linear scaling | Bottlenecked |

---

## Optimization Recommendations

### 1. Collect histograms
```sql
ANALYZE TABLE products COLUMNS category ENABLE;
ANALYZE TABLE orders_good COLUMNS product, status ENABLE;
```
Improves optimizer row count estimates leading to better query plans.

### 2. Fix the shard key on orders_bad
```sql
-- Recreate with a better shard key
CREATE ROWSTORE TABLE orders_good_v2 (
    order_id    INT NOT NULL,
    customer_id INT NOT NULL,
    product     VARCHAR(100),
    amount      DECIMAL(10,2),
    status      VARCHAR(20),
    SHARD KEY (order_id)
);
```
Using `order_id` instead of `status` eliminates data skew entirely.

### 3. Add indexes for frequent filter columns
```sql
ALTER TABLE orders_good ADD INDEX idx_status (status);
ALTER TABLE orders_good ADD INDEX idx_product (product);
```
Speeds up WHERE clause filtering on non-shard-key columns.

### 4. Compiled plan caching
The 23ms compile time only occurs on the first execution. Subsequent runs reuse the cached compiled plan. In production, warm up critical queries after deployment to avoid first-run latency.
