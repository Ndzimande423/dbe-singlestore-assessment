# Cluster Expansion

## Objective
Add a child aggregator and a second leaf node to the existing SingleStore cluster.

---

## Step 1 — Deploy Additional Containers

Two additional Docker containers were deployed to act as the child aggregator and second leaf node.

### Child aggregator container
```bash
sudo docker run -d --name singlestore-child-agg \
  -e ROOT_PASSWORD="Assessment@2025!" \
  -e SINGLESTORE_VERSION="8.1" \
  -p 3308:3306 \
  --restart unless-stopped \
  ghcr.io/singlestore-labs/singlestoredb-dev:latest
```

### Second leaf container
```bash
sudo docker run -d --name singlestore-leaf2 \
  -e ROOT_PASSWORD="Assessment@2025!" \
  -e SINGLESTORE_VERSION="8.1" \
  -p 3309:3306 \
  --restart unless-stopped \
  ghcr.io/singlestore-labs/singlestoredb-dev:latest
```

### Verify all 3 containers healthy
```bash
sudo docker ps
```

Output:
```
CONTAINER ID   IMAGE                                              STATUS        PORTS                    NAMES
b11308d92168   ghcr.io/singlestore-labs/singlestoredb-dev:latest Up (healthy)  0.0.0.0:3309->3306/tcp   singlestore-leaf2
ab87d3808420   ghcr.io/singlestore-labs/singlestoredb-dev:latest Up (healthy)  0.0.0.0:3308->3306/tcp   singlestore-child-agg
d99c7bad51a3   ghcr.io/singlestore-labs/singlestoredb-dev:latest Up (healthy)  0.0.0.0:3306->3306/tcp   singlestoredb-dev
```

### Get container IPs
```bash
sudo docker inspect singlestore-child-agg --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# Output: 172.17.0.3

sudo docker inspect singlestore-leaf2 --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# Output: 172.17.0.4
```

---

## Step 2 — Attempt to Add Nodes via SQL

### Attempt 1 — colon syntax
```sql
ADD AGGREGATOR '172.17.0.3':3306 IDENTIFIED BY 'Assessment@2025!';
```
Result:
```
ERROR 1064 (42000): You have an error in your SQL syntax near '3306 IDENTIFIED BY'
```

### Attempt 2 — comma syntax
```sql
ADD AGGREGATOR '172.17.0.3', 3306 IDENTIFIED BY 'Assessment@2025!';
```
Result:
```
ERROR 1064 (42000): You have an error in your SQL syntax near ', 3306 IDENTIFIED BY'
```

### Attempt 3 — no port
```sql
ADD AGGREGATOR '172.17.0.3';
```
Result:
```
ERROR 1064 (42000): You have an error in your SQL syntax
```

### Attempt 4 — leaf node
```sql
ADD LEAF '172.17.0.4' PORT 3306;
```
Result:
```
ERROR 1064 (42000): You have an error in your SQL syntax near 'PORT 3306'
```

---

## Step 3 — Attempt via sdb-admin

sdb-admin is installed on the EC2 host at `/usr/bin/sdb-admin`.

```bash
sudo sdb-admin add-aggregator \
  --host 172.17.0.3 \
  --port 3306 \
  --password 'Assessment@2025!'
```

Result:
```
Toolbox does not support running with sudo. Please add 'user = "root"' to
/root/.config/singlestoredb-toolbox/toolbox.hcl
```

Fixed by running as root:
```bash
sudo su -
mkdir -p /root/.config/singlestoredb-toolbox
echo 'user = "root"' > /root/.config/singlestoredb-toolbox/toolbox.hcl

sdb-admin add-aggregator \
  --host 172.17.0.3 \
  --port 3306 \
  --password 'Assessment@2025!'
```

Result:
```
failed to find host with ma
```

---

## Root Cause Analysis

| Layer | Finding |
|---|---|
| Docker dev image | Sealed single-node environment — cannot register external nodes |
| sdb-admin | Manages bare-metal/VM SingleStore installations only |
| Docker containers | Not visible to sdb-admin as valid SingleStore nodes |
| Self-managed license | Required for sdb-deploy which supports proper multi-node clusters |

The Docker dev image runs a complete Master Aggregator + Leaf Node inside a single container. It is designed for development and testing on a single machine — not for building distributed clusters.

To properly add a child aggregator and second leaf node, the following is required:

1. A SingleStore self-managed license key
2. Install SingleStore on separate hosts/VMs using `sdb-deploy`
3. Bootstrap each node individually
4. Register them with the master aggregator

---

## Commands That Would Work With a Proper License

Once SingleStore is installed via sdb-deploy on separate nodes:

```bash
# On the child aggregator host — bootstrap it first
sdb-admin bootstrap-aggregator \
  --host <child-agg-ip> \
  --port 3308 \
  --password 'Assessment@2025!'

# On the master aggregator — add the child aggregator
sdb-admin add-aggregator \
  --host <child-agg-ip> \
  --port 3308

# On the master aggregator — add the second leaf
sdb-admin add-leaf \
  --host <leaf2-ip> \
  --port 3307
```

Expected SHOW AGGREGATORS output after successful expansion:
```
+-----------+------+--------+-------------------+
| Host      | Port | State  | Master_Aggregator |
+-----------+------+--------+-------------------+
| 127.0.0.1 | 3306 | online |                 1 |  ← Master Aggregator
| 172.17.0.3| 3308 | online |                 0 |  ← Child Aggregator
+-----------+------+--------+-------------------+
```

Expected SHOW LEAVES output after successful expansion:
```
+-----------+------+--------------------+--------+
| Host      | Port | Availability_Group | State  |
+-----------+------+--------------------+--------+
| 127.0.0.1 | 3307 |                  1 | online |  ← Leaf 1
| 172.17.0.4| 3309 |                  2 | online |  ← Leaf 2
+-----------+------+--------------------+--------+
```
