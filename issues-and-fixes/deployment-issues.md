# Deployment Issues and Resolutions

## Issue 1 — SingleStore Dev Image Partition Limit

**Error encountered:**
```
ERROR: databases are restricted to two partitions
```

**Root cause:**
SingleStore dev image versions 0.2.40 and later restrict databases to a maximum of 2 partitions. The assessment requires 16 partitions per leaf.

**Resolution:**
Used SingleStore version 8.1 which predates the restriction:
```bash
sudo docker run -d --name singlestoredb-dev \
  -e SINGLESTORE_VERSION="8.1" \
  -e SINGLESTORE_SET_GLOBAL_DEFAULT_PARTITIONS_PER_LEAF=16 \
  ...
```

**Result:** Database created successfully with 16 partitions ✅

---

## Issue 2 — Self-Managed License Not Available

**Problem:**
The assessment requires a self-managed license key for cluster expansion (child aggregator + second leaf). The SingleStore portal no longer provides free self-managed license keys directly.

**What was tried:**
- Portal → Install Self-Managed → only shows Dev Image (2 partition limit)
- Portal → API Keys → management API only, not a license
- Multiple email registrations → same result
- Contact Us → Self-Managed Licenses → requires enterprise contact form

**Resolution:**
- Used Docker version 8.1 as workaround for the partition requirement
- Cluster expansion documented with commands that would work with a proper license
- Issue raised with assessors

---

## Issue 3 — Cluster Expansion Blocked (Docker Limitation)

**Problem:**
Cannot add child aggregator or second leaf node to a Docker dev image container.

**Attempts made:**

SQL syntax attempts — all failed with ERROR 1064:
```sql
ADD AGGREGATOR '172.17.0.3':3306 IDENTIFIED BY 'Assessment@2025!';
ADD AGGREGATOR '172.17.0.3', 3306 IDENTIFIED BY 'Assessment@2025!';
ADD AGGREGATOR '172.17.0.3';
ADD LEAF '172.17.0.4' PORT 3306;
```

sdb-admin attempt:
```bash
sdb-admin add-aggregator --host 172.17.0.3 --port 3306 --password 'Assessment@2025!'
# Result: failed to find host with ma
```

**Root cause:**
The Docker dev image is a sealed single-node environment. sdb-admin manages bare-metal installations only and cannot register Docker-based nodes.

**Resolution:**
- Deployed 2 additional Docker containers (singlestore-child-agg, singlestore-leaf2) to demonstrate the attempt
- Documented the correct commands that would work with a self-managed license
- Raised with assessors for license provision

---

## Issue 4 — sdb-admin sudo Permission

**Error:**
```
Toolbox does not support running with sudo. Please add 'user = "root"' to
/root/.config/singlestoredb-toolbox/toolbox.hcl
```

**Resolution:**
```bash
sudo su -
mkdir -p /root/.config/singlestoredb-toolbox
echo 'user = "root"' > /root/.config/singlestoredb-toolbox/toolbox.hcl
```

**Result:** sdb-admin ran as root but still could not register Docker nodes ✅ (issue 3 above)

---

## Issue 5 — Docker API Permission Denied

**Error:**
```
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```

**Resolution:**
All Docker commands prefixed with `sudo`:
```bash
sudo docker ps
sudo docker exec -it singlestoredb-dev singlestore -pAssessment@2025!
```

**Result:** Docker commands executed successfully ✅

---

## Summary

| Issue | Status | Resolution |
|---|---|---|
| Dev image 2 partition limit | ✅ Resolved | Used SingleStore v8.1 |
| Self-managed license unavailable | ⚠️ Workaround | Docker v8.1 used, expansion documented |
| Cluster expansion blocked | ⚠️ Documented | Commands documented, raised with assessors |
| sdb-admin sudo permission | ✅ Resolved | Added root config |
| Docker API permission denied | ✅ Resolved | Used sudo prefix |
