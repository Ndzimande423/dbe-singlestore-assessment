# Database and Table Creation

## Database Creation with 16 Partitions

```sql
CREATE DATABASE assessment_db PARTITIONS 16;
```

Output:
```
Query OK, 1 row affected (2.83 sec)
```

16 partitions distribute data across 16 buckets on the leaf node enabling parallel query execution.

```sql
USE assessment_db;
```

---

## Table Creation

### Table 1 — orders_good (Good Shard Key)

```sql
CREATE ROWSTORE TABLE orders_good (
    order_id    INT NOT NULL,
    customer_id INT NOT NULL,
    product     VARCHAR(100),
    amount      DECIMAL(10,2),
    status      VARCHAR(20),
    SHARD KEY (order_id)
);
```

**Why order_id is a good shard key:**
- High cardinality — every order has a unique ID
- Rows spread evenly across all 16 partitions
- Queries filtering by order_id go directly to one partition (partition pruning)

---

### Table 2 — orders_bad (Poor Shard Key)

```sql
CREATE ROWSTORE TABLE orders_bad (
    order_id    INT NOT NULL,
    customer_id INT NOT NULL,
    product     VARCHAR(100),
    amount      DECIMAL(10,2),
    status      VARCHAR(20),
    SHARD KEY (status)
);
```

**Why status is a poor shard key:**
- Low cardinality — only 3 distinct values: `shipped`, `pending`, `delivered`
- All rows with the same status land on the same partition — data skew
- Hot partitions — some partitions overloaded while others sit idle

---

### Table 3 — products (Reference Table)

```sql
CREATE REFERENCE TABLE products (
    product_id   INT NOT NULL,
    product_name VARCHAR(100),
    category     VARCHAR(50),
    PRIMARY KEY (product_id)
);
```

**Why a reference table:**
- Small lookup table replicated to ALL leaf nodes
- Eliminates network shuffle when joining with sharded tables
- Triggers a broadcast join — key for the query plan analysis task

---

## Data Insertion

### Insert into orders_good
```sql
INSERT INTO orders_good VALUES
(1,  101, 'Laptop',   999.99, 'shipped'),
(2,  102, 'Phone',    499.99, 'pending'),
(3,  103, 'Tablet',   299.99, 'shipped'),
(4,  104, 'Monitor',  399.99, 'delivered'),
(5,  105, 'Keyboard',  79.99, 'pending'),
(6,  106, 'Mouse',     29.99, 'shipped'),
(7,  107, 'Headset',  149.99, 'delivered'),
(8,  108, 'Webcam',    89.99, 'pending'),
(9,  109, 'Desk',     599.99, 'shipped'),
(10, 110, 'Chair',    799.99, 'delivered');
```

### Insert into orders_bad
```sql
INSERT INTO orders_bad VALUES
(1,  101, 'Laptop',   999.99, 'shipped'),
(2,  102, 'Phone',    499.99, 'pending'),
(3,  103, 'Tablet',   299.99, 'shipped'),
(4,  104, 'Monitor',  399.99, 'delivered'),
(5,  105, 'Keyboard',  79.99, 'pending'),
(6,  106, 'Mouse',     29.99, 'shipped'),
(7,  107, 'Headset',  149.99, 'delivered'),
(8,  108, 'Webcam',    89.99, 'pending'),
(9,  109, 'Desk',     599.99, 'shipped'),
(10, 110, 'Chair',    799.99, 'delivered');
```

### Insert into products
```sql
INSERT INTO products VALUES
(1, 'Laptop',   'Electronics'),
(2, 'Phone',    'Electronics'),
(3, 'Tablet',   'Electronics'),
(4, 'Monitor',  'Electronics'),
(5, 'Keyboard', 'Accessories');
```

---

## Data Verification

### Row counts confirmed
```sql
SELECT COUNT(*) FROM orders_good;
-- Output: 10

SELECT COUNT(*) FROM orders_bad;
-- Output: 10

SELECT COUNT(*) FROM products;
-- Output: 5
```

### Data skew — orders_bad (poor shard key)
```sql
SELECT status, COUNT(*) FROM orders_bad GROUP BY status;
```

Output:
```
+-----------+----------+
| status    | COUNT(*) |
+-----------+----------+
| shipped   |        4 |
| delivered |        3 |
| pending   |        3 |
+-----------+----------+
3 rows in set (0.05 sec)
```

All 10 rows land in only 3 partition buckets. In a large dataset this creates severe hot partitions.

### Even distribution — orders_good (good shard key)
```sql
SELECT order_id % 4 AS bucket, COUNT(*) FROM orders_good GROUP BY bucket;
```

Output:
```
+--------+----------+
| bucket | COUNT(*) |
+--------+----------+
|      1 |        3 |
|      3 |        2 |
|      2 |        3 |
|      0 |        2 |
+--------+----------+
4 rows in set (0.05 sec)
```

Rows spread evenly across buckets — all leaf nodes share the workload equally.

---

## Verify Tables
```sql
SHOW TABLES;
```

Output:
```
+-------------------------+
| Tables_in_assessment_db |
+-------------------------+
| orders_bad              |
| orders_good             |
| products                |
+-------------------------+
3 rows in set (0.00 sec)
```

---

## Schema Summary

```
assessment_db (16 partitions per leaf)
│
├── orders_good  [ROWSTORE]  SHARD KEY (order_id)   ← even distribution
├── orders_bad   [ROWSTORE]  SHARD KEY (status)     ← data skew
└── products     [REFERENCE]                        ← replicated to all nodes
```

Both orders tables join to products via product/product_name. Because products is a reference table, this join triggers a broadcast operation — the reference table is available locally on every leaf node without network transfer.
