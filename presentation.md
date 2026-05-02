# DBE Technical Assessment — SingleStore
## Presentation Document
**Candidate:** Lunga Ndzimande | **Date:** May 2026

---

## Slide 1 — Who Am I and What Did I Build?

**What is SingleStore?**
SingleStore is a high-performance distributed database that can process millions of records in milliseconds. Think of it as a database that splits your data across multiple workers and searches everything at the same time — instead of one by one.

**What was built in this assessment:**
- A cloud server on AWS running SingleStore
- A database split across 16 parallel workers (partitions)
- Tables designed to show good vs bad data distribution
- A live query demonstration with full performance analysis

---

## Slide 2 — The Infrastructure

**Where it runs:**

| What | Value |
|---|---|
| Cloud Provider | AWS (Amazon Web Services) |
| Server Type | m6a.2xlarge |
| CPU | 8 cores |
| Memory | 32GB RAM |
| Location | Ireland (eu-west-1) |
| Access | AWS SSM Session Manager — no open ports |

**Why this server?**
The assessment specified m6a.2xlarge. It has enough power to run a full SingleStore cluster and demonstrate real performance characteristics.

---

## Slide 3 — The Cluster

**What is a SingleStore cluster?**

Think of it like a restaurant:

```
┌─────────────────────────────────────┐
│         Master Aggregator           │
│         (The Head Waiter)           │
│   Takes orders, routes to kitchen   │
│         Port 3306 — Online ✅        │
└──────────────────┬──────────────────┘
                   │
┌──────────────────▼──────────────────┐
│            Leaf Node 1              │
│            (The Kitchen)            │
│   Stores data, processes queries    │
│         Port 3307 — Online ✅        │
└─────────────────────────────────────┘
```

- The **Head Waiter** receives all requests and coordinates the work
- The **Kitchen** is where data lives and queries are processed
- Both confirmed online with 0.227ms response time

---

## Slide 4 — Cluster Expansion Attempt

**What was required:**
Add a Child Aggregator (second head waiter) and a second Leaf Node (second kitchen)

**What was done:**
Two additional containers were deployed and confirmed healthy:

```
singlestore-child-agg  — port 3308 — healthy ✅
singlestore-leaf2      — port 3309 — healthy ✅
```

**What was blocked:**
Registering these containers with the Master Aggregator requires a self-managed license. The free developer portal no longer provides this directly — it must be requested from SingleStore enterprise.

**Commands ready to run once license is available:**
```sql
ADD AGGREGATOR '172.17.0.3' IDENTIFIED BY '...';
ADD LEAF '172.17.0.4' IDENTIFIED BY '...';
```

---

## Slide 5 — The Database: 16 Partitions

**What is a partition?**

Imagine a filing cabinet with 1 million files. One drawer = slow. 16 drawers searched at the same time = 16x faster.

**Confirmed:**
```
DATABASE_NAME  | NUM_PARTITIONS
assessment_db  |             16  ✅
```

**Why 16 matters:**
- 16 parallel workers process every query simultaneously
- Data is automatically split across all 16 drawers
- As data grows, all 16 workers share the load equally

---

## Slide 6 — Good vs Bad Table Design

**Two identical tables. One key difference — the shard key.**

The shard key decides which drawer (partition) each record goes into.

| | orders_good | orders_bad |
|---|---|---|
| Shard Key | order_id | status |
| Cardinality | High — every order unique | Low — only 3 values |
| Partitions used | 16 of 16 | 3 of 16 |
| Hot partitions | None | Yes — 13 empty |
| At 1 million rows | 62,500 per partition | 400,000 on one partition |

**The lesson:**
A bad shard key wastes 81% of your cluster's capacity. Choosing the right shard key is the single most important design decision in SingleStore.

---

## Slide 7 — The Broadcast Query

**What was demonstrated:**
A query that joins orders with a product catalogue — without moving any data across the network.

```sql
SELECT o.order_id, o.product, o.amount, p.category
FROM orders_good o
JOIN products p ON o.product = p.product_name
WHERE p.category = 'Electronics';
```

**Result:**
```
Phone   — $499.99
Laptop  — $999.99
Tablet  — $299.99
Monitor — $399.99
```

**Why this is special:**
The `products` table is a reference table — a full copy lives on every partition. When the query runs, each partition joins its own data with its own local copy of products. No data travels across the network.

---

## Slide 8 — What the Tools Revealed

**Four diagnostic tools were used:**

**EXPLAIN** — The plan before execution
> "Here is how I intend to run this query"
> Confirmed: broadcast join, parallel execution across 16 partitions

**PROFILE** — What actually happened
> "Here is exactly what happened when I ran it"
> Network traffic: 0.112 KB — near zero ✅
> 11 of 16 data segments skipped automatically ✅

**EXPLAIN EXTENDED** — The internal SQL sent to leaf nodes
> "Here is the exact instruction I sent to each kitchen"
> Shows SingleStore rewrites queries internally for distributed execution

**SHOW PROFILE JSON** — Per-partition breakdown
> "Here is what each individual drawer did"
> Shows avg, max and which partition handled the most work
> Partition 7 handled most orders | Partition 3 handled most products

---

## Slide 9 — Performance Numbers

| What | Result | What it means |
|---|---|---|
| Total query time | 26ms | Fast for a distributed query |
| Compile time (first run) | 23ms | One-time cost — cached after |
| Execution time (second run) | 4ms | 6x faster after caching |
| Network traffic | 0.112 KB | Broadcast join working perfectly |
| Data segments skipped | 11 of 16 | Database skips irrelevant data automatically |
| Partitions working | 16 | All workers active simultaneously |

**Key insight:**
88% of the first query's time was setup. The second run took 4ms. In production, queries are warmed up after deployment so users never experience the first-run delay.

---

## Slide 10 — Challenges and How They Were Handled

| Challenge | What Happened | How It Was Resolved |
|---|---|---|
| Dev image limits to 2 partitions | Could not create 16 partitions | Used SingleStore v8.1 which predates the restriction — 16 partitions confirmed ✅ |
| Self-managed license unavailable | Could not expand cluster | Containers deployed, commands documented, license request raised |
| DEBUG PROFILE not in v8.1 | Syntax error on DEBUG PROFILE | Used SHOW PROFILE JSON — provides same per-partition detail ✅ |

**What this demonstrates:**
Every blocker was identified, documented, and worked around professionally. No challenge was left undocumented or ignored.

---

## Slide 11 — Key Takeaways

**1. Partitions = Speed**
More partitions means more parallel workers. 16 partitions = 16 workers searching simultaneously instead of 1.

**2. Shard Key = Everything**
The wrong shard key wastes 81% of your cluster. Always choose a column with many unique values.

**3. Reference Tables = Free Joins**
Small lookup tables should be reference tables. The join costs almost nothing — 0.112 KB of network traffic.

**4. SingleStore Optimises Automatically**
11 of 16 data segments were skipped without any manual tuning. The database is smart enough to avoid reading data it doesn't need.

**5. First Run vs Subsequent Runs**
First run: 26ms (includes compilation). Second run: 4ms. Always warm up critical queries in production.

---

## Slide 12 — Repository

All work is documented and available at:

**GitHub:** https://github.com/Ndzimande423/dbe-singlestore-assessment

```
├── deployment/        — Infrastructure setup, database, tables
├── analysis/          — All query plans and outputs
├── findings/          — Comparisons and architecture diagrams
├── performance/       — Performance metrics and optimizations
├── issues-and-fixes/  — All challenges and resolutions
├── plain-language-guides/ — Non-technical explanations
└── final-report.md    — Complete assessment report
```

**Every command executed, every output captured, every challenge documented.**
