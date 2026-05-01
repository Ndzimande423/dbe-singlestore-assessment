# Child Aggregator and Second Leaf — Plain Language Guide

## First, a Quick Recap of What We Have Now

Think of the current setup like a restaurant:

```
Master Aggregator  =  The Head Waiter
Leaf Node 1        =  The Kitchen
```

- The Head Waiter takes orders from customers (queries)
- The Head Waiter sends the order to the Kitchen
- The Kitchen prepares the food (processes the data)
- The Head Waiter brings the result back to the customer

This works fine for now. But what happens when the restaurant gets busy?

---

## What a Second Leaf Node Does

Adding a second leaf is like **opening a second kitchen**.

```
Master Aggregator  =  The Head Waiter
Leaf Node 1        =  Kitchen 1  (handles half the orders)
Leaf Node 2        =  Kitchen 2  (handles the other half)
```

The Head Waiter now splits the work between both kitchens. Both kitchens cook at the same time. Food comes out twice as fast.

In database terms:
- Leaf 1 stores partitions P0 to P7 (half the data)
- Leaf 2 stores partitions P8 to PF (the other half)
- Both leaves process queries simultaneously
- Results are combined and returned to the client

**The data is automatically split between the two leaves when Leaf 2 is added. No manual work needed.**

---

## What a Child Aggregator Does

Adding a child aggregator is like **hiring a second Head Waiter**.

```
Master Aggregator   =  Head Waiter 1  (main entry point)
Child Aggregator    =  Head Waiter 2  (handles extra customers)
Leaf Node 1         =  Kitchen 1
Leaf Node 2         =  Kitchen 2
```

The Child Aggregator does NOT store any data. Its only job is to handle more customer connections and route their orders to the kitchens.

**Why would you need this?**
- If 1,000 customers all arrive at the same time, one Head Waiter gets overwhelmed
- A second Head Waiter means both can take orders simultaneously
- The kitchens (leaf nodes) are never the bottleneck — the waiter is
- Adding a Child Aggregator removes that bottleneck

---

## Can We Add a Second Leaf Without the Child Aggregator?

**Yes — absolutely.**

The Child Aggregator and the second Leaf Node are completely independent of each other. You can add a second leaf without ever adding a child aggregator.

The blocker we hit was specifically about registering nodes with the Master Aggregator — which requires a self-managed license. That blocker applies to BOTH the child aggregator AND the second leaf.

So the situation is:

| Action | Requires License | Status |
|---|---|---|
| Add second leaf node | Yes | ⚠️ Blocked — license needed |
| Add child aggregator | Yes | ⚠️ Blocked — license needed |
| Both are independent | — | Adding one does not require the other |

---

## What Would Happen to the Data With 2 Leaves

Right now with 1 leaf:
```
Leaf 1 owns ALL 16 partitions
└── P0, P1, P2, P3, P4, P5, P6, P7, P8, P9, PA, PB, PC, PD, PE, PF
    └── ALL 10 rows of orders_good live here
    └── ALL 10 rows of orders_bad live here
    └── products replicated here
```

After adding Leaf 2, SingleStore automatically rebalances:
```
Leaf 1 owns 8 partitions          Leaf 2 owns 8 partitions
└── P0, P1, P2, P3, P4, P5, P6, P7   └── P8, P9, PA, PB, PC, PD, PE, PF
    └── ~5 rows of orders_good            └── ~5 rows of orders_good
    └── ~5 rows of orders_bad             └── ~5 rows of orders_bad
    └── products replicated here          └── products replicated here
```

The rebalancing happens automatically in the background while the database stays online. No downtime. No manual data movement.

---

## The Simple Summary

| Component | Role | Like a Restaurant |
|---|---|---|
| Master Aggregator | Main entry point — routes all queries | Head Waiter |
| Child Aggregator | Extra entry point — handles more connections | Second Head Waiter |
| Leaf Node 1 | Stores and processes half the data | Kitchen 1 |
| Leaf Node 2 | Stores and processes the other half | Kitchen 2 |

- **More leaf nodes** = more storage + faster queries (data split across more kitchens)
- **More aggregators** = more client connections handled (more waiters taking orders)
- **They are independent** — you can add a leaf without a child aggregator and vice versa
- **Both are blocked** in our current setup by the same licensing requirement

Once a self-managed license is available, both can be added with two commands:
```sql
ADD LEAF '172.17.0.4' IDENTIFIED BY 'Assessment@2025!';
ADD AGGREGATOR '172.17.0.3' IDENTIFIED BY 'Assessment@2025!';
```
