# Streaming & Messaging — Kinesis/MSK · Pub/Sub · Event Hubs · Kafka

> **New repository** — added because real-time ingestion is a top-weighted domain on both DEA-C01 and GCP PDE, and the original layout only had the *router* layer (`EventRouting`), not the *broker* layer.
>
> Certifications served: **DEA-C01** (Kinesis family — heavily tested), **GCP PDE** (Pub/Sub — core), **AZ-900** (Event Hubs/Service Bus awareness), **CKA** (Strimzi/StatefulSets pattern).

## 1. Concept-First Blueprint

Two distinct species — never conflate them:

```text
STREAM (log):  producers ─► partitioned, ordered, REPLAYABLE log ─► many consumer groups, each with offsets
QUEUE (task):  producers ─► buffer ─► competing consumers ─► message DELETED after ack
```

Streams = analytics/event sourcing/replay. Queues = work distribution/decoupling. Exam questions hinge on picking the right species.

## 2. Rosetta Stone

| Concept | AWS | GCP | Azure | OSS/k8s |
| --- | --- | --- | --- | --- |
| Partitioned stream | **Kinesis Data Streams** | **Pub/Sub** (partitions hidden) | **Event Hubs** | **Kafka topic** |
| Managed Kafka proper | **MSK** | Managed Service for Apache Kafka | Event Hubs **Kafka endpoint** / HDInsight | Strimzi operator |
| Simple queue | **SQS** (standard/FIFO) | Pub/Sub (pull) / Cloud Tasks | **Storage Queues** / **Service Bus** | RabbitMQ, NATS |
| Pub/sub fan-out | **SNS** | Pub/Sub (multi-subscription) | Service Bus topics / Event Grid | Kafka consumer groups |
| Throughput unit | Shard (1MB/s in, 2MB/s out) | Auto (quota-based) | Throughput Unit / Processing Unit | Partition count + broker spec |
| Ordering scope | Per shard (partition key) | Per **ordering key** (must enable) | Per partition | Per partition |
| Replay / retention | 24h default ➔ 365d max | 7d default ➔ 31d ack'd via topic retention; **seek** to timestamp/snapshot | 1d ➔ 90d (Premium) | Disk-bound, infinite-ish |
| Delivery guarantee | At-least-once (KDS) | At-least-once (exactly-once *within* Dataflow) | At-least-once | At-least-once / EOS with transactions |
| Firehose-style delivery | **Data Firehose** (zero-code ➔ S3/Redshift/OpenSearch) | Pub/Sub **BigQuery / GCS subscriptions** | Event Hubs **Capture** ➔ ADLS | Kafka Connect sinks |
| Dead-letter | SQS DLQ / consumer DLQs | Dead-letter topics | Service Bus DLQ (built-in) | DLQ topics (manual) |

## 3. Per-Provider Deep Dive

### AWS — a family you must tell apart (DEA-C01 obsession)
- **Kinesis Data Streams (KDS)**: shard math is the exam: need 5 MB/s ingest ➔ ≥5 shards; partition-key skew ➔ hot shards. Provisioned vs **on-demand** capacity. Enhanced Fan-Out gives each consumer dedicated 2MB/s push (vs shared polling). KCL handles checkpointing (in DynamoDB!).
- **Data Firehose**: zero-shard, fully managed delivery with buffering (size/time), inline Lambda transform, format conversion to Parquet — *near*-real-time (≥60s/≥1MB buffer at best, plus the transform hop), never sub-second. "Easiest way to land a stream in S3/Redshift" ➔ Firehose, always.
- **SQS vs SNS vs KDS**: SQS = decouple work (no replay, 14d max, visibility timeout, FIFO ≤ ~300-3000 msg/s); SNS = push fan-out; KDS = ordered replayable analytics log. SNS➔SQS fan-out is the classic durable-fan-out pattern.
- **MSK**: real Kafka when you need the protocol/ecosystem (Connect, Streams, exact offset semantics); MSK Serverless removes broker sizing.

### GCP Pub/Sub — the global no-ops broker
- **No shards, no partitions to manage, global by default** — the most serverless of the three. Per-message billing.
- Push vs pull subscriptions; ack deadlines (extendable); **ordering keys** must be explicitly enabled and trade throughput per key; **seek** to timestamp/snapshot = replay.
- **Exactly-once exists only end-to-end with Dataflow** (or EOS pull subscriptions in one region) — PDE's favorite nuance. BigQuery subscriptions write straight to a table (no Dataflow needed — know when that suffices vs when transformation forces Dataflow).
- Pub/Sub **Lite** (zonal, capacity-based, cheap) is being retired — answer "Lite" questions with caution/migration framing.

### Azure — two products by design
- **Event Hubs** = the stream (Kafka-wire-compatible: repoint Kafka clients at it with config only). Throughput Units (standard) / Processing Units (Premium); **Capture** auto-lands Avro into ADLS — the Firehose analog.
- **Service Bus** = the enterprise queue/topic: sessions (ordering), transactions, dedupe, scheduled delivery, built-in DLQ — features SQS lacks.
- AZ-900 framing: Event Hubs = big-data ingestion; Service Bus = enterprise messaging; Event Grid = reactive event routing (see `EventRouting`); Storage Queues = simple/cheap.

### Kafka on k8s (Strimzi)
- Brokers = StatefulSet + PVCs; KRaft replaces ZooKeeper; operators handle rolling broker restarts. Running it teaches *why* managed offerings price what they price. CKA crossover: this is the canonical stateful-workload exercise.

## 4. Differences & Identical Links

**Identical:** partition/key ➔ ordering scope; consumer-group offset model; at-least-once default (so **consumers must be idempotent** — universal exam answer); DLQ pattern; buffer-then-deliver "firehose" tier on every platform.

**Different:**
- **Capacity model**: AWS makes you do shard math; Azure sells throughput units; GCP hides it entirely.
- **Replay**: native and central in KDS/Event Hubs/Kafka; in Pub/Sub it's snapshot/seek (and gone once unretained).
- Azure uniquely splits stream (Event Hubs) from queue (Service Bus) into separate products; AWS splits into three (KDS/SQS/SNS); GCP collapses everything into Pub/Sub.

## 5. Cross-Component Operating Notes

- **Stream ➔ functions**: Lambda ESM polls KDS per-shard (batch size, parallelization factor, bisect-on-error); Pub/Sub *pushes* to Cloud Run functions (HTTP, auto-retry with exponential backoff, OIDC-signed); Event Hubs trigger checkpoints via the Function App's storage account. Same job, three couplings (see `ServerlessFunctions`).
- **Stream ➔ warehouse**: KDS ➔ Firehose ➔ Redshift (COPY under the hood) or Redshift streaming materialized views directly on KDS/MSK; Pub/Sub ➔ BigQuery subscription or Dataflow ➔ Storage Write API; Event Hubs ➔ Capture ➔ Synapse / Fabric Eventstream.
- **Stream ➔ processing**: Managed Flink reads KDS natively; **Dataflow is *the* Pub/Sub consumer** (watermarks, exactly-once); Databricks Structured Streaming reads Event Hubs via its Kafka endpoint.
- **Identity**: producers/consumers authenticate by IAM role (KDS), service account (Pub/Sub), Managed Identity or SAS (Event Hubs), SASL/mTLS (Kafka) — the workload-identity patterns from `IdentitySecurity` apply unchanged.
- Throughput planning trap: resharding KDS mid-stream disrupts ordering; Event Hubs partition count is **fixed at creation** (standard tier); Kafka allows adding partitions but breaks key ➔ partition mapping. Plan partitions up front, everywhere.

## 6. Certification Checkpoints

- **DEA-C01**: KDS vs Firehose vs MSK vs SQS/SNS selection (multiple questions guaranteed), shard math, Enhanced Fan-Out, Firehose buffering/transform, KCL checkpointing.
- **GCP PDE**: push vs pull, ordering keys, seek/snapshot replay, exactly-once-with-Dataflow, BigQuery subscriptions.
- **AZ-900**: position Event Hubs vs Service Bus vs Event Grid vs Storage Queues.
- **CKA**: StatefulSets, headless Services, PVC retention — via the Strimzi lens.
