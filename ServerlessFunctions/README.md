# Serverless Functions — Lambda · Cloud Run functions · Azure Functions · Knative

> Certifications served: **DEA-C01** (Lambda in pipelines — heavily tested), **GCP PDE** (Cloud Run functions as glue), **AZ-900** (serverless positioning), **CKA** (Knative awareness only).

## 1. Concept-First Blueprint

```text
Event occurs ─► Platform provisions a micro-environment ─► Your handler runs ─► Scale to zero
```

Naming alert (from your notes): **Google Cloud Functions is now "Cloud Run functions"** — folded under the container-based Cloud Run umbrella.

## 2. Rosetta Stone

| Metric | AWS Lambda | Cloud Run functions (GCP) | Azure Functions | Knative Serving (k8s) |
| --- | --- | --- | --- | --- |
| Deployment unit | Zip ≤50MB (250MB unzipped) or container ≤10GB | Source folder ➔ auto-built container (Cloud Build/Buildpacks) | **Function App** (group of functions sharing plan/storage) | Container + Service CR |
| Max timeout | **15 min** hard | 60 min (HTTP) | 10 min Consumption / ~unlimited Premium-Dedicated | Configurable |
| Scaling driver | Memory slider (CPU scales with it), 1 request per instance instance* | Explicit CPU+memory; **many concurrent requests per instance** | Hosting plan (Consumption / Flex / Premium / Dedicated) | Concurrency-based (KPA), scale-to-zero |
| Connection style | Event source mappings + SDK calls in code | Triggers + **Eventarc** routing | **Triggers & Bindings** (declarative I/O) | Knative Eventing |
| HTTP exposure | API Gateway or Function URL (extra step) | Native HTTPS URL out of the box | Native URL out of the box | Ingress/Kourier |
| State extension | Step Functions (external) | Workflows (external) | **Durable Functions (built-in)** | Argo Workflows |
| Cold-start mitigations | Provisioned Concurrency, SnapStart (Java/.NET/Python) | Min instances | Premium pre-warmed instances | minScale |

\* Lambda = strict one-request-per-environment isolation; this is *the* architectural divider vs Cloud Run's in-instance concurrency.

## 3. Per-Provider Deep Dive

### AWS Lambda — the isolated event processor
- Runs on **Firecracker microVMs**; every function version is immutable; aliases route traffic (canary 90/10 etc.).
- **Execution role** (IAM) = what the function may call; **resource-based policy** = who may invoke the function. Two different documents — classic exam confusion.
- Invocation modes: synchronous (API GW), asynchronous (S3/SNS/EventBridge; built-in retries ×2 + DLQ/Destinations), **poll-based** (SQS, Kinesis, DynamoDB Streams via *Event Source Mappings* — Lambda polls for you; understand batch size, batch window, `bisectBatchOnError`, parallelization factor per shard for DEA-C01).
- Packaging & dependency rules (full detail in `preliminary_chat.md`): dependencies flat at zip root; Layers use `python/` or `nodejs/node_modules/` structure, `/opt` mount, max 5 layers; container images up to 10GB via ECR for pandas/numpy-class workloads; **native-binary trap** — build on Lambda-matching Linux/arch (`sam build --use-container`, or `pip install --platform manylinux2014_x86_64`).
- VPC-attached Lambdas get ENIs (Hyperplane); they lose internet unless a NAT Gateway exists — recurring gotcha.

### Cloud Run functions — container-native
- Your source is auto-containerized (Cloud Build + Buildpacks) and executed by Cloud Run: hence traffic splitting between revisions, GPU attachment, and concurrency >1 per instance all come free.
- 1st-gen vs 2nd-gen: 2nd-gen *is* Cloud Run — longer timeouts, bigger instances, Eventarc triggers.
- Identity: each function runs as a **service account**; calling BigQuery/GCS requires zero keys — just bind roles.

### Azure Functions — the enterprise Function App
- You deploy a **Function App**, not single functions; all functions inside share plan, storage account, and identity.
- **Triggers & Bindings**: declarative input/output (Cosmos DB, Blob, Service Bus, Event Hubs) injected as handler parameters — no SDK boilerplate. Killer feature; know it conceptually for AZ-900.
- **Durable Functions**: stateful orchestrations (fan-out/fan-in, human interaction, eternal orchestrations) coded in-line — AWS needs Step Functions, GCP needs Workflows for the same.
- A Function App *requires* an Azure Storage account (state, triggers bookkeeping) — common quiz nugget.

### Knative — the k8s standard
- Serving: revision-based, scale-to-zero HTTP workloads. Eventing: see `EventRouting`. KEDA is the pragmatic alternative for event-driven *scaling* of plain Deployments (and powers Azure Container Apps).

## 4. Differences & Identical Links

**Identical:** event ➔ handler ➔ ephemeral compute; per-invocation billing; identity-per-function (role / service account / managed identity); cold starts and the same mitigations (pre-warmed capacity).

**Different:**
- **Concurrency model**: Lambda 1-per-sandbox vs Cloud Run many-per-instance vs Azure plan-dependent.
- **Grouping**: solo functions (AWS/GCP) vs Function App bundle (Azure).
- **HTTP**: AWS needs API GW/Function URL; GCP/Azure give URLs natively (from your notes).
- **State**: only Azure builds orchestration in (Durable); others externalize it.

## 5. Cross-Component Operating Notes

- **S3 ➔ Lambda**: S3 Event Notifications invoke Lambda directly (resource policy allows `s3.amazonaws.com`), or via EventBridge for filtering/fan-out. GCS ➔ Eventarc ➔ Cloud Run function (Eventarc rides Pub/Sub under the hood). Blob ➔ Event Grid ➔ Azure Function (Event Grid trigger beats legacy blob polling trigger for latency).
- **Stream consumption**: Lambda ESM polls Kinesis shards (one batch per shard concurrently); Pub/Sub *pushes* to Cloud Run functions; Event Hubs trigger uses the Functions host's EventProcessor with checkpoints in the storage account. Same job — pull vs push vs host-managed.
- **In data pipelines (DEA-C01 patterns)**: Lambda = lightweight transform/dispatch (file-arrival ➔ start Glue job / Step Functions); NOT for >15-min or shuffle-heavy work — that's Glue/EMR. GCP equivalent: Cloud Run function triggers a Dataflow template launch; Azure: Function activity inside a Data Factory pipeline.
- **Warehouse loading**: Lambda ➔ Redshift Data API (async, no connection pools in functions!); Cloud Run function ➔ BigQuery `insertAll`/load jobs; Azure Function ➔ Synapse via managed identity + output bindings.

## 6. Certification Checkpoints

- **DEA-C01**: ESM tuning, async retries/DLQ, 15-min limit as a disqualifier, Lambda-in-VPC networking, layers/packaging, Redshift Data API.
- **GCP PDE**: Cloud Run functions as Eventarc targets, service-account auth to BigQuery/GCS, when to choose Dataflow instead.
- **AZ-900**: serverless = consumption billing + auto-scale; Functions vs Logic Apps (code vs designer); Function App concept.
- **CKA**: not tested, but Knative/KEDA awareness rounds out the k8s picture.
