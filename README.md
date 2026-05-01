# DBE Technical Assessment - SingleStore
**Candidate:** Lunga Ndzimande  
**Account:** 281814941186 | **Region:** eu-west-1  
**EC2 Instance:** m6a.2xlarge | **IP:** 34.242.222.87 (bastion)  
**Repository:** https://github.com/Ndzimande423/dbe-singlestore-assessment

---

## Repository Structure

```
├── documentation/
│   ├── cluster-setup.md          # Cluster setup, Docker deployment, connection
│   ├── database-tables.md        # Database creation, tables, shard keys, data distribution
│   ├── query-plan-analysis.md    # Broadcast query, EXPLAIN, PROFILE, DEBUG PROFILE
│   └── cluster-expansion.md     # Child aggregator and leaf node expansion attempts
│
├── issues-and-fixes/
│   └── deployment-issues.md     # Blockers encountered and resolutions
│
└── README.md
```

---

## Task 1 — Cluster Setup and Configuration

### EC2 Instance
| Item | Value |
|---|---|
| Instance class | m6a.2xlarge |
| vCPUs | 8 |
| RAM | 32GB |
| OS | Ubuntu |
| Region | eu-west-1 |

### SingleStore Deployment
SingleStore was deployed using the official Docker dev image (`singlestoredb-dev-image`).

**Note on licensing:** The assessment requires 16 partitions per leaf. The current SingleStore dev image (v0.2.40+) restricts databases to 2 partitions. To work around this, SingleStore version 8.1 was used which predates the restriction.

```bash
sudo docker run -d --name singlestoredb-dev \
  -e ROOT_PASSWORD="Assessment@2025!" \
  -e SINGLESTORE_VERSION="8.1" \
  -e SINGLESTORE_SET_GLOBAL_DEFAULT_PARTITIONS_PER_LEAF=16 \
  -p 3306:3306 -p 8080:8080 -p 9000:9000 \
  --restart unless-stopped \
  ghcr.io/singlestore-labs/singlestoredb-dev:latest
```

**Cluster topology:**
| Node | Port | Role | Status |
|---|---|---|---|
| singlestoredb-dev | 3306 | Master Aggregator | ✅ Online |
| singlestoredb-dev | 3307 | Leaf Node 1 | ✅ Online |
| singlestore-child-agg | 3308 | Child Aggregator (attempted) | ⚠️ Blocked |
| singlestore-leaf2 | 3309 | Leaf Node 2 (attempted) | ⚠️ Blocked |

### Cluster Connection
```bash
sudo docker exec -it singlestoredb-dev singlestore -pAssessment@2025!
```

---

## Task 2 — Database and Table Creation

### Database with 16 Partitions
```sql
CREATE DATABASE assessment_db PARTITIONS 16;
```

### Tables Created
| Table | Type | Shard Key | Distribution |
|---|---|---|---|
| orders_good | ROWSTORE | order_id | Even — high cardinality |
| orders_bad | ROWSTORE | status | Skewed — only 3 distinct values |
| products | REFERENCE | N/A | Replicated to all nodes |

**Full documentation:** `documentation/database-tables.md`

---

## Task 3 — Query Execution and Plan Analysis

### Broadcast Query
A JOIN between a sharded table (`orders_good`) and a reference table (`products`) triggers a broadcast operation — the reference table is broadcast to all leaf nodes.

```sql
SELECT o.order_id, o.product, o.amount, p.category
FROM orders_good o
JOIN products p ON o.product = p.product_name
WHERE p.category = 'Electronics';
```

**Outputs captured:**
- `EXPLAIN` — logical query plan
- `PROFILE` — execution statistics per operator
- `DEBUG PROFILE` — detailed node-level execution breakdown

**Full analysis:** `documentation/query-plan-analysis.md`

---

## Task 4 — Cluster Expansion

Two additional Docker containers were deployed to simulate child aggregator and second leaf node:

```bash
# Child aggregator container
sudo docker run -d --name singlestore-child-agg \
  -e ROOT_PASSWORD="Assessment@2025!" \
  -e SINGLESTORE_VERSION="8.1" \
  -p 3308:3306 --restart unless-stopped \
  ghcr.io/singlestore-labs/singlestoredb-dev:latest

# Second leaf container  
sudo docker run -d --name singlestore-leaf2 \
  -e ROOT_PASSWORD="Assessment@2025!" \
  -e SINGLESTORE_VERSION="8.1" \
  -p 3309:3306 --restart unless-stopped \
  ghcr.io/singlestore-labs/singlestoredb-dev:latest
```

**Blocker:** The Docker dev image is a sealed single-node environment. `sdb-admin` manages bare-metal installations only and cannot register Docker-based nodes. A self-managed license is required for a proper multi-node cluster via `sdb-deploy`.

**Full details:** `issues-and-fixes/deployment-issues.md`
