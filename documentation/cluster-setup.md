# Cluster Setup and Configuration

## EC2 Instance

| Item | Value |
|---|---|
| Instance class | m6a.2xlarge |
| vCPUs | 8 |
| RAM | 32GB |
| Storage | 96GB (80GB free) |
| OS | Ubuntu |
| Region | eu-west-1 |
| Connection method | AWS SSM Session Manager |

---

## Initial Setup

### Step 1 — Verify Docker is installed
```bash
sudo docker ps
```

Output confirmed Docker was running and accessible via sudo.

### Step 2 — Check available resources
```bash
free -h && df -h
```

Output:
```
              total        used        free
Mem:           30Gi        1.3Gi       28Gi
Swap:            0B          0B         0B

Filesystem      Size  Used Avail Use%
/dev/root        96G   16G   80G  17% /
```

---

## SingleStore Deployment

### Licensing Decision

The assessment requires a database with 16 partitions per leaf. The current SingleStore dev image (v0.2.40+) restricts databases to a maximum of 2 partitions as documented in the official GitHub README:

> "For singlestoredb-dev-image versions 0.2.40 and later, databases are restricted to two partitions. Using the SINGLESTORE_SET_GLOBAL_DEFAULT_PARTITIONS_PER_LEAF environment variable will not override this value."

A self-managed license was investigated through the SingleStore portal. The portal no longer provides free self-managed license keys directly — they must be requested via the enterprise contact form. As a workaround, SingleStore version 8.1 was used which predates the partition restriction.

### Step 3 — Deploy SingleStore Docker container (v8.1)
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

### Step 4 — Verify container is healthy
```bash
sudo docker ps
```

Output:
```
CONTAINER ID   IMAGE                                              COMMAND               CREATED        STATUS                    PORTS
d99c7bad51a3   ghcr.io/singlestore-labs/singlestoredb-dev:latest "/scripts/start.sh"   36 hours ago   Up (healthy)   0.0.0.0:3306->3306/tcp, 0.0.0.0:8080->8080/tcp, 0.0.0.0:9000->9000/tcp
```

---

## Cluster Connection

### Connect using the mysql client
```bash
sudo docker exec -it singlestoredb-dev singlestore -pAssessment@2025!
```

Output:
```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 58
Server version: 5.7.32 SingleStoreDB source distribution (compatible; MySQL Enterprise & MySQL Commercial)

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

singlestore>
```

### Verify cluster topology
```sql
SHOW AGGREGATORS;
```

Output:
```
+-----------+------+--------+--------------------+------------------------------+-------------------+--------+
| Host      | Port | State  | Opened_Connections | Average_Roundtrip_Latency_ms | Master_Aggregator | NodeId |
+-----------+------+--------+--------------------+------------------------------+-------------------+--------+
| 127.0.0.1 | 3306 | online |                  1 | NULL                         |                 1 |      1 |
+-----------+------+--------+--------------------+------------------------------+-------------------+--------+
```

```sql
SHOW LEAVES;
```

Output:
```
+-----------+------+--------------------+-----------+-----------+--------+--------------------+------------------------------+--------+
| Host      | Port | Availability_Group | Pair_Host | Pair_Port | State  | Opened_Connections | Average_Roundtrip_Latency_ms | NodeId |
+-----------+------+--------------------+-----------+-----------+--------+--------------------+------------------------------+--------+
| 127.0.0.1 | 3307 |                  1 | NULL      | NULL      | online |                 16 | 0.227                        |      2 |
+-----------+------+--------------------+-----------+-----------+--------+--------------------+------------------------------+--------+
```

**Cluster summary:**
| Node | Host | Port | Role | Status |
|---|---|---|---|---|
| Master Aggregator | 127.0.0.1 | 3306 | Receives and routes all queries | Online |
| Leaf Node 1 | 127.0.0.1 | 3307 | Stores and processes data | Online |

---

## Cluster Expansion Attempts

Two additional Docker containers were deployed to simulate a child aggregator and second leaf node.

### Deploy child aggregator container
```bash
sudo docker run -d --name singlestore-child-agg \
  -e ROOT_PASSWORD="Assessment@2025!" \
  -e SINGLESTORE_VERSION="8.1" \
  -p 3308:3306 \
  --restart unless-stopped \
  ghcr.io/singlestore-labs/singlestoredb-dev:latest
```

### Deploy second leaf container
```bash
sudo docker run -d --name singlestore-leaf2 \
  -e ROOT_PASSWORD="Assessment@2025!" \
  -e SINGLESTORE_VERSION="8.1" \
  -p 3309:3306 \
  --restart unless-stopped \
  ghcr.io/singlestore-labs/singlestoredb-dev:latest
```

### Verify all containers running
```bash
sudo docker ps
```

Output:
```
CONTAINER ID   IMAGE                                              STATUS      PORTS                    NAMES
b11308d92168   ghcr.io/singlestore-labs/singlestoredb-dev:latest Up (healthy) 0.0.0.0:3309->3306/tcp   singlestore-leaf2
ab87d3808420   ghcr.io/singlestore-labs/singlestoredb-dev:latest Up (healthy) 0.0.0.0:3308->3306/tcp   singlestore-child-agg
d99c7bad51a3   ghcr.io/singlestore-labs/singlestoredb-dev:latest Up (healthy) 0.0.0.0:3306->3306/tcp   singlestoredb-dev
```

### Get container IPs
```bash
sudo docker inspect singlestore-child-agg --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# Output: 172.17.0.3

sudo docker inspect singlestore-leaf2 --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# Output: 172.17.0.4
```

### Attempt to add child aggregator (SQL)
```sql
ADD AGGREGATOR '172.17.0.3':3306 IDENTIFIED BY 'Assessment@2025!';
```
Result: `ERROR 1064 (42000)` — syntax not supported in this version

### Attempt via sdb-admin
```bash
sudo sdb-admin add-aggregator \
  --host 172.17.0.3 \
  --port 3306 \
  --password 'Assessment@2025!'
```
Result: `failed to find host with ma` — sdb-admin manages bare-metal installations only, not Docker containers

### Root cause
The Docker dev image is a sealed single-node environment. sdb-admin cannot register Docker-based nodes. A proper multi-node cluster requires a self-managed license and bare-metal installation via sdb-deploy.

The commands to add nodes once a proper license is obtained:
```sql
-- Add child aggregator (requires running SingleStore node at target host)
ADD AGGREGATOR '172.17.0.3' IDENTIFIED BY 'Assessment@2025!';

-- Add second leaf node
ADD LEAF '172.17.0.4' IDENTIFIED BY 'Assessment@2025!';
```
