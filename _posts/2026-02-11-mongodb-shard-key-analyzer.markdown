---
layout: post
title:  "Choosing the Right MongoDB Shard Key: A Data-Driven Approach"
date:   2026-02-11 10:00:00 +0800
categories: mongodb sharding atlas automation
---

Sharding is how MongoDB scales horizontally — splitting data across multiple servers so no single machine has to handle everything. It works well when done right, but the entire strategy hinges on one critical decision: choosing the shard key.

I built a [shard key analyzer](https://github.com/tzehon/research/tree/main/shard-key-analyzer) that wraps MongoDB's native analysis commands in an interactive UI, letting you evaluate and compare candidate shard keys before committing. This post covers the concepts behind sharding, what makes a good shard key, the MongoDB commands that help you decide, and how the analyzer ties it all together.

## What Is Sharding?

A standalone MongoDB instance or replica set stores all your data on one set of machines. That works until it doesn't — disk fills up, queries slow down, write throughput hits a ceiling. Sharding splits a collection's documents across multiple **shards** (each shard being its own replica set), so reads and writes can be distributed.

MongoDB uses the **shard key** — one or more fields present in every document — to decide which shard owns each document. The shard key divides the data into **chunks**, and MongoDB's balancer moves chunks between shards to keep things even.

```
        +------------------------------+
        |       mongos (router)        |
        |  Routes queries by shard key |
        +-------+----------+----------+
                |          |          |
           +----+---+ +----+---+ +----+----+
           |Shard A | |Shard B | |Shard C  |
           |ch 1-4  | |ch 5-8  | |ch 9-12  |
           +--------+ +--------+ +---------+
```

When a query includes the shard key, `mongos` routes it directly to the right shard (**targeted query**). When it doesn't, `mongos` broadcasts the query to every shard and merges results (**scatter-gather**). Targeted queries are fast; scatter-gather queries get slower as you add shards.

## Why the Shard Key Matters

The shard key determines three things:

1. **How data is distributed** — whether documents spread evenly or pile up on one shard
2. **How queries are routed** — whether reads hit one shard or all of them
3. **How writes are distributed** — whether inserts fan out or create hot spots

MongoDB supports [resharding](https://www.mongodb.com/docs/manual/core/sharding-reshard-a-collection/) and [refining a shard key](https://www.mongodb.com/docs/manual/core/sharding-refine-a-shard-key/) if you need to change course after sharding, so you're not permanently locked in. That said, resharding involves copying data across shards, so it's worth putting in the effort upfront to choose a good key. Here are the three failure modes a poor shard key can lead to:

**Hot spots** — If your shard key has low cardinality (few distinct values) or is monotonically increasing (like a timestamp), all new writes land on the same shard. The other shards sit idle while one is overloaded.

**Scatter-gather queries** — If your application mostly queries by a field that isn't the shard key, every query hits every shard. Adding more shards makes queries slower, not faster.

**Jumbo chunks** — If one shard key value appears in millions of documents, MongoDB can't split that chunk further. You end up with an immovable chunk that keeps growing.

## Shard Key Best Practices

A good shard key has three properties:

### High Cardinality

The key should have many distinct values — ideally as many as there are documents. This gives MongoDB room to split data into small, balanced chunks.

| Key | Cardinality | Verdict |
|-----|------------|---------|
| `userId` (UUID) | Millions | Good |
| `email` | Millions | Good |
| `region` | 4 | Bad — only 4 chunks possible |
| `status` | 5 | Bad — data piles into a few chunks |
| `isActive` | 2 | Bad — two chunks, one probably huge |

### Low Frequency (Even Distribution)

Even with many distinct values, the distribution matters. If 80% of documents have the same `customerId`, that customer's chunk becomes a hot spot.

### Non-Monotonic

Values shouldn't increase or decrease over time. Monotonically increasing keys (timestamps, auto-incrementing IDs, ObjectIds) cause all inserts to go to the shard that owns the maximum chunk range.

| Key | Monotonic? | Problem |
|-----|-----------|---------|
| `createdAt` | Yes | All inserts go to one shard |
| `_id` (ObjectId) | Yes | ObjectId starts with a timestamp |
| `UUID` | No | Random distribution |
| `{ region: 1, orderId: 1 }` | Partially | Compound key can help — random within each region |

### Match Your Query Patterns

The shard key should appear in your most common queries. If 90% of your reads filter by `customerId`, then `customerId` in the shard key means 90% of reads target a single shard.

### Compound Shard Keys

When no single field checks all the boxes, a compound shard key combines multiple fields. For example, `{ customerId: 1, createdAt: 1 }` gives you:

- **Targeted reads** — queries filtering by `customerId` hit one shard
- **Range queries within a customer** — date range scans within a customer stay on one shard
- **Better distribution** — the combination has higher cardinality than either field alone

### Common Patterns by Use Case

| Use Case | Good Shard Key | Why |
|----------|---------------|-----|
| E-commerce orders | `{ customerId: 1 }` | Most queries are per-customer |
| Multi-tenant SaaS | `{ tenantId: 1, entityId: 1 }` | Isolates tenants; entityId prevents jumbo chunks for large tenants |
| IoT / time-series | `{ deviceId: 1, timestamp: 1 }` | Distributes by device; enables time range queries |
| Social media posts | `{ userId: 1, createdAt: 1 }` | User timeline queries are most common |

## The Two MongoDB Commands That Help

Starting in MongoDB 7.0, two database commands let you evaluate shard keys with real data before committing. They work on both replica sets and sharded clusters, so you can analyze before sharding.

### `configureQueryAnalyzer`

This command turns on **query sampling** for a collection. While active, MongoDB records a sample of the reads and writes hitting that collection into `config.sampledQueries`.

```javascript
db.adminCommand({
  configureQueryAnalyzer: "mydb.orders",
  mode: "full",
  samplesPerSecond: 5
})
```

- **mode** — `"full"` to start, `"off"` to stop
- **samplesPerSecond** — how many queries per second to capture (1–50)

Each sampled query looks like this:

```json
{
  "cmdName": "find",
  "ns": "mydb.orders",
  "cmd": {
    "filter": { "customerId": "c45fc6a3-89e1-46a1-861a-64ff578e2d31" }
  },
  "expireAt": "2026-03-11T04:14:21.618Z"
}
```

You want sampling running long enough to capture representative traffic. For production, 20–30 minutes at a low rate (1–5/sec) is a reasonable starting point. Sampled queries expire automatically via a TTL index (default ~27 days).

### `analyzeShardKey`

This command evaluates a candidate shard key by examining two data sources:

```javascript
db.adminCommand({
  analyzeShardKey: "mydb.orders",
  key: { customerId: 1 },
  keyCharacteristics: true,
  readWriteDistribution: true
})
```

**Key characteristics** (from the documents themselves):
- **Cardinality** — how many distinct values the key has
- **Frequency** — how evenly those values are distributed
- **Monotonicity** — whether values increase/decrease over time

**Read/write distribution** (from sampled queries):
- **Read targeting** — what percentage of reads would target a single shard
- **Write targeting** — what percentage of writes would target a single shard
- **Scatter-gather ratio** — how many queries would need to hit all shards

The analysis is read-only and doesn't modify your data. You can run it repeatedly with different candidates. The more sampled queries available, the more accurate the read/write distribution metrics.

### The Catch

These commands are powerful but not easy to use interactively. `analyzeShardKey` returns raw JSON with nested objects, and comparing multiple candidates means running the command multiple times and manually diffing the output. There's no scoring, no visualization, and no side-by-side comparison. That's the gap the shard key analyzer fills.

## How the Shard Key Analyzer Helps

The [shard key analyzer](https://github.com/tzehon/research/tree/main/shard-key-analyzer) is a web application (React + Express) that wraps both commands in a guided workflow with scoring and visualization.

### Workflow

```
1. Connect   -- enter your Atlas connection string
2. Explore   -- browse databases, collections, schemas, indexes
3. Sample    -- start configureQueryAnalyzer (runs in background)
4. Traffic   -- let your app run, or use the workload simulator
5. Analyze   -- submit candidate shard keys (runs analyzeShardKey)
6. Report    -- compare candidates with scores and charts
```

### Scoring System

The analyzer translates the raw `analyzeShardKey` output into scores on five metrics, each weighted:

| Metric | Weight | What It Measures |
|--------|--------|-----------------|
| Cardinality | 25% | Number of distinct shard key values |
| Frequency | 20% | Evenness of value distribution |
| Monotonicity | 15% | Whether values increase/decrease over time |
| Read Targeting | 20% | Percentage of reads targeting a single shard |
| Write Targeting | 20% | Percentage of writes targeting a single shard |

Each metric scores 0–100, and the weighted average gives an overall score with a letter grade:

| Grade | Score | Meaning |
|-------|-------|---------|
| A | 90–100 | Excellent candidate |
| B | 80–89 | Good candidate |
| C | 70–79 | Fair — consider alternatives |
| D | 60–69 | Poor — likely problematic |
| F | <60 | Avoid — will cause issues |

For example, `{ status: 1 }` on an e-commerce orders collection might score:

- Cardinality: 15/100 (only 5 values)
- Frequency: 30/100 (most orders are "delivered")
- Monotonicity: 100/100 (not monotonic)
- Read Targeting: 20/100 (few queries filter by status alone)
- Write Targeting: 25/100 (updates scatter across shards)
- **Overall: 36/100 (F)**

While `{ customerId: 1 }` on the same collection:

- Cardinality: 95/100 (millions of customers)
- Frequency: 85/100 (fairly even distribution)
- Monotonicity: 100/100 (UUIDs aren't monotonic)
- Read Targeting: 90/100 (most queries filter by customer)
- Write Targeting: 80/100 (writes naturally target by customer)
- **Overall: 90/100 (A)**

### Warnings

The analyzer generates warnings when it detects patterns that indicate problems:

- **Low cardinality** — fewer than 100 distinct values
- **Monotonic key** — values increase or decrease over time
- **Hot spot detected** — one value appears far more often than average
- **High scatter-gather** — more than 50% of reads hit all shards
- **Shard key updates** — writes that modify the shard key field (causes document migration between shards)

### Candidate Recommendations

Before you even run the analysis, the explorer page generates candidate suggestions by sampling 100 documents and analyzing field characteristics:

- Fields named `customerId`, `userId`, or `tenantId` get a boost
- Fields named `timestamp` or `createdAt` get a penalty
- High-cardinality fields score higher
- Fields with many nulls are penalized
- Compound key combinations are generated from the top candidates

This gives you a starting point — you're not guessing at random field combinations.

### Workload Simulator

If you don't have real application traffic yet (maybe you're designing a new system), the built-in workload simulator generates realistic query patterns. Three profiles are available:

- **E-commerce** — customer lookups, order placement, status updates, regional aggregations
- **Social Media** — user feeds, post creation, likes, comments, trending queries
- **IoT** — device data writes, time-range reads, device lookups

The simulator runs real reads and writes against your database, so use it only on test data. For production collections, use your actual application traffic instead.

## Getting Started

### Prerequisites

| Requirement | Details |
|------------|---------|
| MongoDB | 7.0+ on Atlas (M30+ for sharding) or any replica set |
| Node.js | 18+ |
| Database User | `clusterManager` role (covers all needed commands) |
| Network | Your IP in the Atlas whitelist |

### Setup

```bash
git clone https://github.com/tzehon/research.git
cd research/shard-key-analyzer
npm install
npm run dev
```

This starts the Express backend on port 3001 and the Vite dev server on port 5173. Open `http://localhost:5173`.

### Step-by-Step Walkthrough

**1. Connect** — Enter your MongoDB connection string:

```
mongodb+srv://<username>:<password>@<cluster>.xxxxx.mongodb.net/
```

The app verifies your MongoDB version (7.0+), deployment type (replica set or sharded), and permissions.

**2. Explore** — Browse your databases and collections. Select a collection to see its schema, document count, indexes, and field-level statistics. The explorer also generates initial shard key candidate suggestions based on field analysis.

**3. Start Sampling** — Go to the Sampling page, set a sampling rate (start with 1–5/sec for production), and click Start. This runs `configureQueryAnalyzer` in the background. Leave it running while you generate traffic.

**4. Generate Traffic** — Either let your real application run its normal workload (recommended), or use the Workload page to run the simulator against test data. The sampling captures queries as they flow in. More queries means better read/write distribution data.

**5. Analyze Candidates** — Go to the Analysis page. Pick candidates from the suggestions or add your own (e.g., `{ customerId: 1 }`, `{ customerId: 1, createdAt: 1 }`). Click Analyze. The tool runs `analyzeShardKey` for each candidate and calculates scores.

**6. Review the Report** — The report page shows:
- Overall recommendation with the highest-scoring candidate
- Radar charts comparing all candidates across the five metrics
- Detailed score breakdowns with explanations
- Warnings about potential issues
- Side-by-side comparison bars

### Supporting Indexes

Before you actually shard a collection, MongoDB requires a **supporting index** that matches or prefixes your shard key. The analyzer checks for this and shows you the exact `createIndex` command if one is needed:

```javascript
db.orders.createIndex({ customerId: 1 })
```

If you're evaluating `{ customerId: 1, createdAt: 1 }`, you need an index on `{ customerId: 1, createdAt: 1 }` (or an index where those fields are a prefix).

## Production Safety

If you're running this against a production cluster, be aware of the overhead each operation introduces:

| Operation | Impact | Recommendation |
|-----------|--------|----------------|
| `configureQueryAnalyzer` | CPU/IO overhead for intercepting queries | Low sampling rate (1–5/sec) |
| `analyzeShardKey` | Read-only, reads documents proportional to sample size | Default 10,000 is fine; avoid during peak hours |
| Workload Simulator | **Runs real reads and writes** | **Never use on production** |
| Sample Data Loading | Inserts documents | Test databases only |

The safest production workflow: connect, start sampling at a low rate, let your real application generate traffic for 20–30 minutes, then analyze.

## Permissions Quick Reference

| Command | Minimum Role |
|---------|-------------|
| `configureQueryAnalyzer` | `dbAdmin` on the target database, or `clusterManager` |
| `analyzeShardKey` | `enableSharding` on the collection, or `clusterManager` |
| Delete sampled queries | `clusterManager` |

The `clusterManager` role covers everything. On Atlas, the **Atlas Admin** built-in role includes it.

## Resources

- [Choose a Shard Key](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/) — MongoDB's guide to shard key traits
- [`analyzeShardKey` command reference](https://www.mongodb.com/docs/manual/reference/command/analyzeshardkey/)
- [`configureQueryAnalyzer` command reference](https://www.mongodb.com/docs/manual/reference/command/configurequeryanalyzer/)
- [Resharding a collection](https://www.mongodb.com/docs/manual/core/sharding-reshard-a-collection/)
- [Refining a shard key](https://www.mongodb.com/docs/manual/core/sharding-refine-a-shard-key/)
- [Sharding overview](https://www.mongodb.com/docs/manual/sharding/)
- [Shard key analyzer source](https://github.com/tzehon/research/tree/main/shard-key-analyzer)
