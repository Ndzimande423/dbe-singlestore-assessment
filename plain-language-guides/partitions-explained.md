# Understanding Partitions — Plain Language Guide

## What is a Partition?

Think of a partition like a drawer in a filing cabinet.

Imagine your company has 1 million customer order records stored in a single filing cabinet with one drawer. Every time someone needs to find an order, they have to search through all 1 million records one by one. That takes a long time.

Now imagine splitting those same 1 million records across 16 drawers — roughly 62,500 records per drawer. When someone needs to find an order, instead of one person searching 1 million records, you have 16 people each searching their own drawer at the same time. The answer comes back 16 times faster.

That is exactly what a partition does in SingleStore.

---

## Why More Partitions is a Good Thing

### 1. Speed Through Parallelism

With 16 partitions, SingleStore runs 16 workers simultaneously — one per partition. All 16 workers search their portion of the data at the same time and report back. The more partitions you have, the more parallel workers you get, and the faster your queries run.

```
1 partition  → 1 worker searches 1,000,000 rows  → slow
16 partitions → 16 workers each search 62,500 rows → 16x faster
```

### 2. Handles More Data Without Slowing Down

As your data grows, more partitions means the growth is spread across more workers. A database with 16 partitions can handle significantly more data before performance starts to degrade compared to one with 2 partitions.

### 3. More Efficient Use of Hardware

Modern servers have multiple CPU cores. Partitions allow SingleStore to use all available CPU cores at the same time. Without partitions, most of your CPU cores sit idle while one core does all the work.

On the m6a.2xlarge instance used in this assessment (8 vCPUs), 16 partitions means the workload is spread across all available cores — no CPU is left idle.

### 4. Scales as the Cluster Grows

When a second leaf node is added to the cluster, SingleStore automatically splits the 16 partitions between the two leaves — 8 partitions per leaf. Now you have two machines each running 8 parallel workers simultaneously. The performance scales with the hardware.

```
1 leaf  × 16 partitions = 16 parallel workers
2 leaves × 8 partitions = 16 parallel workers across 2 machines
                        = 2× storage capacity
                        = 2× memory available
                        = better fault tolerance
```

---

## The Filing Cabinet Analogy — Full Picture

| Concept | Real World Analogy |
|---|---|
| Database | The entire office filing system |
| Table | One filing cabinet |
| Partition | One drawer in the cabinet |
| Row (record) | One file inside a drawer |
| Shard key | The rule that decides which drawer a file goes into |
| Query | Someone looking for specific files |
| Parallel execution | Multiple people searching different drawers at the same time |

---

## What Happens With a Bad Partition Setup

If you only have 2 partitions (the default restriction in newer SingleStore developer images), it is like having a filing cabinet with only 2 drawers. All 1 million records are split between just 2 workers. You are leaving 14 potential workers idle.

This is why version 8.1 was used in this assessment — to unlock the full 16-partition capability and demonstrate the performance benefits of proper partition configuration.

---

## Summary

More partitions = more parallel workers = faster queries = better use of hardware = scales with data growth.

The 16 partitions configured in this assessment means queries run across 16 simultaneous workers. As the dataset grows or more leaf nodes are added, this foundation scales efficiently without requiring a redesign of the database.
