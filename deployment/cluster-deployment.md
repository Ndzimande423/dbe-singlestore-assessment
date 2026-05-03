# Deployment Documentation

## Assessment Requirement
> Create an EC2 instance with m6a.2xlarge instance class. Deploy a SingleStore Cluster-in-a-box with one aggregator (MA) and one leaf node using SingleStore free license.

---

## EC2 Instance

| Item | Value |
|---|---|
| Instance class | m6a.2xlarge |
| vCPUs | 8 |
| RAM | 32GB |
| Storage | 96GB (80GB free) |
| OS | Ubuntu |
| Region | eu-west-1 |
| Connection | AWS SSM Session Manager |

---

## Licensing Decision

Before deployment, the official GitHub README for the SingleStore dev image was reviewed. It states that versions 0.2.40 and later restrict databases to a maximum of 2 partitions. This limitation was identified proactively before attempting deployment.

To meet the 16-partition requirement, SingleStore version 8.1 was selected — it predates the restriction and supports the full partition configuration.

**Result:** 16 partitions were successfully created and confirmed:

```sql
SELECT * FROM information_schema.DISTRIBUTED_DATABASES WHERE DATABASE_NAME = 'assessment_db';
```

```
+-------------+---------------+----------------+--------------------+
| DATABASE_ID | DATABASE_NAME | NUM_PARTITIONS | NUM_SUB_PARTITIONS |
+-------------+---------------+----------------+--------------------+
|           1 | assessment_db |             16 |                 64 |
+-------------+---------------+----------------+--------------------+
```

Full details: `issues-and-fixes/deployment-issues.md`

---

## SingleStore Deployment

### Step 1 — Verify resources
```bash
free -h && df -h
```
```
Mem:   30Gi total   1.3Gi used   28Gi free
/dev/root  96G  16G  80G  17%
```

### Step 2 — Deploy SingleStore Docker container (v8.1)
```bash
sudo docker run -d --name singlestoredb-dev \
  -e ROOT_PASSWORD="Assessment@2025!" \
  -e SINGLESTORE_VERSION="8.1" \
  -e SINGLESTORE_SET_GLOBAL_DEFAULT_PARTITIONS_PER_LEAF=16 \
  -p 3306:3306 \
  -p 8080:8080 \
  -p 9000:9000 \
  --restart unless-stopped \
  ghcr.io/singlestore-labs/singlestoredb-dev:latest
```

### Step 3 — Verify container is healthy
```bash
sudo docker ps
```
```
CONTAINER ID   IMAGE                                              STATUS        PORTS
d99c7bad51a3   ghcr.io/singlestore-labs/singlestoredb-dev:latest Up (healthy)  0.0.0.0:3306->3306/tcp
                                                                                0.0.0.0:8080->8080/tcp
                                                                                0.0.0.0:9000->9000/tcp
```

---

## Cluster Connection

### Connect using the mysql client
```bash
sudo docker exec -it singlestoredb-dev singlestore -pAssessment@2025!
```
```
Welcome to the MySQL monitor.
Your MySQL connection id is 8517
Server version: 5.7.32 SingleStoreDB source distribution
```

### Verify Master Aggregator
```sql
SHOW AGGREGATORS;
```
```
+-----------+------+--------+-------------------+--------+
| Host      | Port | State  | Master_Aggregator | NodeId |
+-----------+------+--------+-------------------+--------+
| 127.0.0.1 | 3306 | online |                 1 |      1 |
+-----------+------+--------+-------------------+--------+
```

### Verify Leaf Node
```sql
SHOW LEAVES;
```
```
+-----------+------+--------------------+--------+--------------------+------------------------------+
| Host      | Port | Availability_Group | NodeId | Opened_Connections | Average_Roundtrip_Latency_ms |
+-----------+------+--------------------+--------+--------------------+------------------------------+
| 127.0.0.1 | 3307 |                  1 |      2 |                 16 | 0.227                        |
+-----------+------+--------------------+--------+--------------------+------------------------------+
```

**Result:** Master Aggregator on port 3306 and Leaf Node on port 3307 — both online ✅

---

## Cluster Expansion

### Assessment Requirement
> Add a child aggregator to the cluster. Add a new leaf node to the cluster.

Two additional Docker containers were deployed:

```bash
# Child aggregator
sudo docker run -d --name singlestore-child-agg \
  -e ROOT_PASSWORD="Assessment@2025!" \
  -e SINGLESTORE_VERSION="8.1" \
  -p 3308:3306 --restart unless-stopped \
  ghcr.io/singlestore-labs/singlestoredb-dev:latest

# Second leaf
sudo docker run -d --name singlestore-leaf2 \
  -e ROOT_PASSWORD="Assessment@2025!" \
  -e SINGLESTORE_VERSION="8.1" \
  -p 3309:3306 --restart unless-stopped \
  ghcr.io/singlestore-labs/singlestoredb-dev:latest
```

```bash
sudo docker ps
```
```
CONTAINER ID   STATUS        PORTS                    NAMES
b11308d92168   Up (healthy)  0.0.0.0:3309->3306/tcp   singlestore-leaf2
ab87d3808420   Up (healthy)  0.0.0.0:3308->3306/tcp   singlestore-child-agg
d99c7bad51a3   Up (healthy)  0.0.0.0:3306->3306/tcp   singlestoredb-dev
```

**Blocker:** The Docker dev image is a sealed single-node environment. sdb-admin manages bare-metal installations only and cannot register Docker-based nodes. A self-managed license is required for a proper multi-node cluster.

Commands that would work with a proper license:
```sql
ADD AGGREGATOR '172.17.0.3' IDENTIFIED BY 'Assessment@2025!';
ADD LEAF '172.17.0.4' IDENTIFIED BY 'Assessment@2025!';
```

Full details: `issues-and-fixes/deployment-issues.md`
