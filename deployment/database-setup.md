# Database and Table Setup

## Assessment Requirement
> Create a database with 16 partitions per leaf. Create two identical rowstore tables with different shard keys. Insert data — one table with equal distribution, one with data skew.

---

## Database Creation

```sql
CREATE DATABASE assessment_db PARTITIONS 16;
```
```
Query OK, 1 row affected (2.83 sec)
```

16 partitions distribute data across 16 buckets on the leaf node enabling parallel query execution across all partitions simultaneously.

```sql
USE assessment_db;
```

---

## Table 1 — orders_good (Equal Distribution)

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

**Shard key choice — order_id:**
- High cardinality — every order has a unique ID
- SingleStore hashes order_id and distributes rows evenly across all 16 partitions
- Queries filtering by order_id go directly to one partition (partition pruning)
- No hot partitions — all partitions share the workload equally

---

## Table 2 — orders_bad (Data Skew)

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

**Shard key choice — status:**
- Low cardinality — only 3 distinct values: `shipped`, `pending`, `delivered`
- All rows with the same status hash to the same partition
- Creates hot partitions — some partitions overloaded, others idle
- No partition pruning benefit — queries must scan all partitions

---

## Table 3 — products (Reference Table)

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
- Triggers a broadcast join — used in the query plan analysis task

---

## Data Insertion

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

INSERT INTO products VALUES
(1, 'Laptop',   'Electronics'),
(2, 'Phone',    'Electronics'),
(3, 'Tablet',   'Electronics'),
(4, 'Monitor',  'Electronics'),
(5, 'Keyboard', 'Accessories');
```

---

## Verification

```sql
SHOW TABLES;
```
```
+-------------------------+
| Tables_in_assessment_db |
+-------------------------+
| orders_bad              |
| orders_good             |
| products                |
+-------------------------+
```

```sql
SELECT COUNT(*) FROM orders_good;   -- 10
SELECT COUNT(*) FROM orders_bad;    -- 10
SELECT COUNT(*) FROM products;      -- 5
```
