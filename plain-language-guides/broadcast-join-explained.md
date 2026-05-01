# Understanding Broadcast Joins — Plain Language Guide

## What is a Broadcast Join?

Imagine two teams working in different offices need to collaborate on a project.

**Option 1 — Send everyone to one office:**
You move all 1,000 people from Team A to Team B's office. That is expensive, slow, and disruptive. This is what happens in a bad join — data moves across the network.

**Option 2 — Give everyone a copy of the small document:**
Team B has a 1-page reference sheet. Instead of moving anyone, you photocopy that sheet and put one copy on every desk in every office. Now everyone has what they need locally. This is a broadcast join.

In SingleStore, a broadcast join happens when a small reference table is copied to every partition in the cluster. When a query needs to join data, each partition already has its own local copy of the reference table — no data needs to travel across the network.

---

## Is a Broadcast Join Good or Bad?

### It Depends on the Size of the Table Being Broadcast

**Broadcast joins are GOOD when:**

The table being broadcast is small — like a product catalogue, a list of categories, a country list, or any other lookup table that rarely changes and has a limited number of rows.

In this assessment, the `products` table has 5 rows. Broadcasting 5 rows to 16 partitions costs almost nothing. The result was confirmed by the PROFILE output:

```
network_traffic: 0.112 KB
```

0.112 KB is essentially zero. The join completed with near-zero network overhead because every partition already had its local copy of the products data.

**Broadcast joins are BAD when:**

The table being broadcast is large — millions of rows, gigabytes of data. Broadcasting a large table means copying all of that data to every single partition in the cluster. This consumes significant network bandwidth, memory on every node, and time. It defeats the purpose of having a distributed database.

---

## The Broadcast Join in This Assessment

The query joined `orders_good` (a sharded table — data split across 16 partitions) with `products` (a reference table — replicated to all partitions).

```
Without broadcast join (bad design):
Each partition would need to ask: "Where is the products data?"
Data would travel across the network to answer the join.
Network traffic: HIGH

With broadcast join (good design — what happened):
Each partition already has its own copy of products.
No data travels across the network.
Network traffic: 0.112 KB ✅
```

The EXPLAIN output confirmed this with:
```
table_type: reference_columnstore
```

This tells us SingleStore recognised `products` as a reference table and executed the join locally on each partition — a broadcast join.

---

## When to Use a Broadcast Join (Reference Table)

| Situation | Use Reference Table? |
|---|---|
| Small lookup table (categories, countries, products) | ✅ Yes |
| Table that is read frequently but rarely updated | ✅ Yes |
| Table with fewer than a few million rows | ✅ Yes |
| Large transaction table with millions of rows | ❌ No — use shard key instead |
| Table that changes constantly | ❌ No — replication overhead too high |

---

## The Simple Rule

> If the table is small and used for lookups — make it a reference table and let SingleStore broadcast it. You get fast joins with zero network cost.

> If the table is large — shard it properly and let SingleStore distribute the work across partitions instead.

---

## Summary

A broadcast join is a **good thing** when used correctly. It means:

- No data moves across the network during the join
- Every partition works independently with its local copy
- Query performance is fast regardless of how large the other table grows
- Network traffic stays near zero

The key is to only broadcast **small** tables. Broadcasting large tables would be expensive and slow — the opposite of what you want.

In this assessment, broadcasting the 5-row `products` table to 16 partitions was the correct design decision. It resulted in 0.112 KB of network traffic for the entire query — essentially free.
