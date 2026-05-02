# DBE Technical Assessment - SingleStore
**Candidate:** Lunga Ndzimande  
**Account:** 281814941186 | **Region:** eu-west-1  
**EC2 Instance:** m6a.2xlarge  
**Repository:** https://github.com/Ndzimande423/dbe-singlestore-assessment

---

## Repository Structure

```
├── deployment/
│   ├── cluster-deployment.md     # EC2 setup, Docker deployment, cluster topology, expansion attempts
│   └── database-setup.md         # Database creation, tables, shard keys, data insertion
│
├── analysis/
│   └── query-execution.md        # Broadcast query, EXPLAIN, EXPLAIN EXTENDED, PROFILE, DEBUG PROFILE
│
├── findings/
│   ├── query-findings.md         # EXPLAIN vs PROFILE comparison, shard key distribution analysis
│   └── cluster-architecture.md   # Current vs expanded cluster, data distribution across 2 leaves, query flow
│
├── performance/
│   └── performance-analysis.md   # Execution timeline, broadcast join performance, optimization recommendations
│
├── issues-and-fixes/
│   └── deployment-issues.md      # Blockers encountered and resolutions
│
├── final-report.md               # Comprehensive report covering all deliverables
└── README.md
```

---

## Task 1 — Cluster Setup and Configuration

### Evaluation Criteria: Initial Setup + Cluster Connection + Cluster Expansion

| Requirement | Status | Detail |
|---|---|---|
| EC2 m6a.2xlarge | ✅ | 8 vCPU, 32GB RAM, eu-west-1 |
| SingleStore Cluster-in-a-box | ✅ | Docker v8.1, MA + 1 Leaf |
| Connect via mysql client | ✅ | `singlestore -pAssessment@2025!` |
| Child aggregator | ⚠️ | Blocked — Docker sealed environment, documented with commands |
| Second leaf node | ⚠️ | Blocked — requires self-managed license, documented with commands |

**Full documentation:** `deployment/cluster-deployment.md`

---

## Task 2 — Database and Table Creation

### Evaluation Criteria: 16 partitions + Shard key variations + Data distribution

| Requirement | Status | Detail |
|---|---|---|
| Database with 16 partitions | ✅ | `CREATE DATABASE assessment_db PARTITIONS 16` |
| orders_good — equal distribution | ✅ | SHARD KEY (order_id) — high cardinality, even spread |
| orders_bad — data skew | ✅ | SHARD KEY (status) — 3 distinct values, hot partitions |
| Reference table | ✅ | products — replicated to all nodes, triggers broadcast |
| Data inserted | ✅ | 10 rows each table, 5 products |

**Full documentation:** `deployment/database-setup.md`

---

## Task 3 — Query Execution and Plan Analysis

### Evaluation Criteria: Broadcast query + EXPLAIN + PROFILE + DEBUG PROFILE + Analysis

| Requirement | Status | Detail |
|---|---|---|
| Broadcast query designed | ✅ | orders_good JOIN products (sharded + reference) |
| Query executed | ✅ | 4 rows returned — Electronics category |
| EXPLAIN captured | ✅ | Shows HashJoin, ColumnStoreScan, Gather |
| EXPLAIN EXTENDED captured | ✅ | Shows leaf-level SQL with STRAIGHT_JOIN |
| PROFILE captured | ✅ | exec_time, memory_usage, network_traffic, segments |
| DEBUG PROFILE | ✅ | Not supported in v8.1 — SHOW PROFILE JSON used, provides per-partition breakdown |
| Comparison documented | ✅ | Full table comparing all 4 outputs |

**Full documentation:** `analysis/query-execution.md`

---

## Key Results

**Broadcast join confirmed:**
```
table_type:reference_columnstore   ← products is broadcast
network_traffic: 0.112 KB          ← near-zero network overhead
parallelism_level:partition        ← runs across all 16 partitions
```

**Shard key impact:**
| Table | Shard Key | Distribution | Hot Partitions |
|---|---|---|---|
| orders_good | order_id | Even across 16 partitions | None |
| orders_bad | status | 3 buckets only | Yes |

**Performance:**
| Metric | Value |
|---|---|
| Total query time | 26ms |
| Compile time (first run) | 23ms |
| Segments skipped | 11 of 16 (columnstore pruning) |
| Network traffic | 0.112 KB |

**Full findings:** `findings/query-findings.md`  
**Cluster architecture:** `findings/cluster-architecture.md`  
**Full performance analysis:** `performance/performance-analysis.md`  
**Issues and blockers:** `issues-and-fixes/deployment-issues.md`  
**Final report:** `final-report.md`

---

## Cluster Architecture — Current vs Expanded

**Current setup (1 MA + 1 Leaf):**
```
Master Aggregator (port 3306)
        │
   Leaf Node 1 (port 3307)
   └── ALL 16 partitions (P0–PF)
       └── All data stored here
```

**Expanded setup (1 MA + 1 CA + 2 Leaves):**
```
Master Aggregator (port 3306)
        │
Child Aggregator (port 3308)
        │
   ┌────┴────┐
Leaf 1      Leaf 2
P0–P7       P8–PF
~5 rows     ~5 rows  ← data split automatically
```

| Aspect | 1 Leaf (current) | 2 Leaves (expanded) |
|---|---|---|
| Partitions per leaf | 16 | 8 each |
| Storage capacity | 1× | 2× |
| Query throughput | Single leaf | Both leaves parallel |
| Fault tolerance | None | Partial — each leaf owns its partitions |

**Full architecture details:** `findings/cluster-architecture.md`
