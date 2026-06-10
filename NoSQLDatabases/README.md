# NoSQL Databases — DynamoDB · Bigtable/Firestore · Cosmos DB · Cassandra

> **New repository** — added because operational NoSQL stores appear on DEA-C01 (DynamoDB) and GCP PDE (Bigtable schema design is a classic question family), and AZ-900 expects Cosmos DB recognition.
>
> Certifications served: **DEA-C01** (DynamoDB), **GCP PDE** (Bigtable, Firestore, plus Spanner contrast), **AZ-900** (Cosmos DB), **CKA** (Cassandra/k8ssandra as the StatefulSet exemplar).

## 1. Concept-First Blueprint

Wide-column/key-value NoSQL trades joins and ad-hoc queries for **horizontal scale at predictable single-digit-ms latency**. Everything follows one law:

```text
Partition key ─► hash ─► physical partition/node.
DESIGN THE TABLE FOR THE ACCESS PATTERN, not the data model.
Hot key = hot partition = throttling. On every platform. Always.
```

## 2. Rosetta Stone

| Concept | DynamoDB | Bigtable | Cosmos DB | Cassandra |
| --- | --- | --- | --- | --- |
| Data model | Key-value / document | Wide-column (single `row key`) | Multi-model (Core SQL, Mongo, **Cassandra**, Gremlin, Table APIs) | Wide-column |
| Partition lever | Partition key (+ sort key) | Row-key design (lexicographic ranges) | Partition key (logical ➔ physical) | Partition key (+ clustering cols) |
| Secondary access | **GSI / LSI** | None — design the row key or duplicate the table | Auto-indexes everything (tunable policy) | Materialized views / SAI (use carefully) |
| Capacity model | RCU/WCU provisioned or **on-demand** | Nodes (SSD/HDD) + autoscaling | **Request Units (RU/s)** provisioned/autoscale/serverless | vnodes / cluster size |
| Consistency | Eventual or strong reads (same region); transactions | Single-row atomic; eventually consistent replication | **Five levels**: strong ➔ bounded staleness ➔ session (default) ➔ consistent prefix ➔ eventual | Tunable per query (ONE/QUORUM/ALL) |
| Global replication | Global Tables (multi-active) | Multi-cluster replication | Turnkey multi-region, multi-master | Multi-DC replication strategy |
| Change feed | **DynamoDB Streams** | Bigtable change streams | **Change Feed** | CDC (commit log) |
| TTL | Per-item TTL | Per-column-family GC policy | Per-item TTL | Per-cell TTL |
| Serverless caching | DAX | — | Integrated cache | — |
| k8s deployment | — (managed only) | — | — | **k8ssandra / Cass-operator** |

GCP nuance: **Bigtable** = petabyte wide-column (time-series, ad-tech, IoT); **Firestore** = serverless document DB for apps; **Spanner** = globally consistent *relational*. PDE tests this triage hard.

## 3. Per-Provider Deep Dive

### DynamoDB (DEA-C01)
- Single-table design: PK + SK composite modeling; **GSI** (any keys, eventual, own capacity) vs **LSI** (same PK, created at table creation only, 10GB per-partition cap).
- Capacity math: 1 RCU = one strongly consistent 4KB read (two eventual); 1 WCU = one 1KB write. On-demand for spiky/unknown traffic.
- **Streams ➔ Lambda** is the canonical CDC pipeline (ESM, batch windows); Kinesis Data Streams destination for bigger fan-out; **DynamoDB ➔ S3 export** (point-in-time, no RCU consumption) is the blessed path to the lake — exam favorite.
- Hot-partition mitigation: write sharding (key suffixes), DAX for read-heavy, exponential backoff on `ProvisionedThroughputExceededException`.

### Bigtable (PDE)
- One index: the **row key**. All exam questions are row-key design: avoid monotonic keys (timestamps ➔ hotspot one tablet); prefer `reversed_id#timestamp` or field promotion/salting; tall-narrow over wide rows for time series.
- Scale by adding nodes (linear throughput); separate app profiles for batch vs serving; HBase API compatible.

### Cosmos DB (AZ-900 + data awareness)
- **RU/s** is the universal currency — every operation costs RUs; 429 = RU exhaustion (the DynamoDB-throttle twin).
- The **five consistency levels** are the marquee differentiator — no other managed NoSQL offers the spectrum; session consistency is the default sweet spot.
- Multi-API strategy: speak MongoDB/Cassandra wire protocols for lift-and-shift onto Azure infrastructure.
- Change Feed ➔ Azure Functions (Cosmos trigger) mirrors Streams ➔ Lambda exactly.

### Cassandra / k8ssandra (CKA crossover + the k8s standard)
- Masterless ring, vnodes, gossip; per-query consistency (`QUORUM` reads+writes ➔ strong-ish). Running it on k8s = StatefulSets, headless Services, PVCs, PodDisruptionBudgets, anti-affinity across zones — a complete CKA stateful-workload workout.

## 4. Differences & Identical Links

**Identical:** partition-key physics and hot-key pathology; LSM-tree storage underneath (write-optimized, compaction); change-feed ➔ serverless function CDC pattern; TTL; "model the queries, not the entities."

**Different:**
- **Secondary indexes**: DynamoDB explicit GSIs, Cosmos indexes-everything-by-default (inverted defaults!), Bigtable refuses entirely, Cassandra discourages.
- **Consistency menu**: Cosmos's five levels vs DynamoDB's binary choice vs Cassandra's per-query tunables vs Bigtable's single-row atomicity.
- **Billing**: RCU/WCU vs RU/s are cousins; Bigtable bills *nodes* (capacity, not requests) — different cost-optimization instincts.

## 5. Cross-Component Operating Notes

- **CDC to the lake/warehouse**: DynamoDB Streams ➔ Lambda ➔ Firehose ➔ S3/Redshift, or native S3 export ➔ Athena/Glue; Bigtable change streams ➔ Dataflow ➔ BigQuery; Cosmos Change Feed ➔ Functions/Synapse Link (HTAP — analytical store with zero ETL — Azure's headline answer).
- **Analytics directly against NoSQL is an anti-pattern** everywhere: don't Spark-scan DynamoDB (burns RCUs) — export first; BigQuery *can* federate to Bigtable but PDE prefers Dataflow copies; Synapse Link exists precisely to avoid hammering Cosmos.
- **From k8s**: pods reach DynamoDB/Bigtable/Cosmos via workload-identity patterns (`IdentitySecurity`); connection-heavy clients (Cassandra drivers) need pod readiness gates and graceful shutdown to avoid client storms.
- Throttle-handling is an *application* concern on all platforms: SDK exponential backoff + jitter; idempotent writes (pairs with at-least-once delivery from `StreamingMessaging`).

## 6. Certification Checkpoints

- **DEA-C01**: GSI vs LSI, capacity modes/math, Streams ➔ Lambda, S3 export, hot partitions, DAX, Global Tables.
- **GCP PDE**: Bigtable row-key design scenarios, Bigtable vs Firestore vs Spanner vs BigQuery triage, node scaling.
- **AZ-900**: Cosmos DB = globally distributed multi-model NoSQL; consistency-spectrum awareness.
- **CKA**: StatefulSets, headless services, PDBs, zone anti-affinity (via the Cassandra lens).
