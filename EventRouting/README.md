# Event Routing — EventBridge · Eventarc · Event Grid · Knative Eventing/Argo Events

> **New repository** — promoted from a row in `Important_Table.md` to its own repo: the Source ➔ Router ➔ Target pattern glues every other repository together and appears on all four exams in some form.
>
> Certifications served: **DEA-C01** (EventBridge in pipelines), **GCP PDE** (Eventarc triggers), **AZ-900** (Event Grid positioning), **CKA** (Knative/Argo Events awareness).

## 1. Concept-First Blueprint (from your Important_Table)

```text
SOURCE (state change) ─► ROUTER (rule/filter match) ─► TARGET (function | queue | workflow | service)
```

The router is **not a broker**: it stores nothing durably for replay-by-consumers, owns no offsets, and exists to *filter and dispatch*. Brokers (Kafka/Kinesis/Event Hubs — see `StreamingMessaging`) hold logs; routers make decisions.

## 2. Rosetta Stone

| Concept | EventBridge (AWS) | Eventarc (GCP) | Event Grid (Azure) | k8s (Knative Eventing / Argo Events) |
| --- | --- | --- | --- | --- |
| Routing object | **Rule** on an event **Bus** | **Trigger** | **Subscription** on a Topic/System Topic | Trigger on a Broker / Sensor |
| Filter language | Event patterns (JSON match: prefix, numeric, anything-but) | CloudEvents attribute filters (exact match mostly) | Subject/event-type + **advanced filters** | CloudEvents attribute filters |
| Native cloud sources | 200+ AWS services via CloudTrail/native | 100+ Google services via Audit Logs + direct | All Azure resources via System Topics | K8s API events, brokers, webhooks |
| SaaS/partner ingestion | **Partner event sources** (Salesforce, Datadog…) | Third-party channels | Partner topics | — |
| Targets | Lambda, SFN, SQS, SNS, Kinesis, API destinations (any HTTP), another bus | Cloud Run (functions), Workflows, GKE services, internal HTTP | Functions, Service Bus, Event Hubs, Storage Queues, Logic Apps, webhooks | Any addressable Service (ksvc, svc) |
| Event format | Proprietary JSON envelope | **CloudEvents** (native) | CloudEvents (configurable) | **CloudEvents** (native) |
| Scheduler twin | EventBridge **Scheduler** | Cloud Scheduler | — (use Logic Apps/Functions timers) | CronJob / Argo Events calendar |
| Replay/archive | **Archive & replay** on the bus | No native replay (underlying Pub/Sub retention) | No native replay (dead-letter only) | Depends on backing channel |
| Failure handling | Retry 24h ➔ DLQ (SQS) | Pub/Sub retry/DLQ semantics | Retry 24h ➔ dead-letter to Storage | Delivery spec + DLQ sink |
| Schema governance | **Schema Registry** (+ discovery) | — (Pub/Sub schemas adjacent) | — | — |

## 3. Per-Provider Deep Dive

### Amazon EventBridge
- Three bus kinds: **default** (AWS service events), **custom** (your apps), **partner** (SaaS). Rules fan out to up to 5 targets; input transformers reshape payloads in-flight.
- **Pipes** (point-to-point: source ➔ optional enrichment ➔ target, e.g. DynamoDB Stream ➔ Lambda-enrich ➔ SFN) vs **Rules** (pub/sub fan-out) — newer exam material.
- S3 ➔ EventBridge (bucket-level opt-in) supersedes classic S3 Notifications when you need rich filtering or >1 destination type.
- Latency is ~hundreds of ms and **not** ordered — never use a router where ordering matters (that's Kinesis/FIFO SQS).

### GCP Eventarc
- **A control plane over Pub/Sub**: every trigger transparently provisions a Pub/Sub transport topic+subscription. Debugging Eventarc = debugging Pub/Sub.
- Two source classes: **direct events** (e.g., GCS object finalized) and **Cloud Audit Log events** (any API call — powerful but requires audit logging enabled and matching serviceName/methodName).
- Targets receive **CloudEvents over HTTP** with OIDC-authenticated push — the receiving Cloud Run service must grant the trigger's service account `run.invoker`.

### Azure Event Grid
- **System Topics** = built-in feeds from Azure resources (storage accounts, resource groups, subscriptions); custom topics for your apps; **domains** for multi-tenant fan-out; **namespaces** add MQTT for IoT.
- Push delivery with webhook **validation handshake** (subscription-validation event) — the classic integration stumbling block.
- AZ-900 positioning: Event Grid (reactive routing, discrete events) vs Event Hubs (telemetry streams) vs Service Bus (transactional messages). Three different products; one favorite exam question.

### Kubernetes: Knative Eventing & Argo Events
- Knative: **Broker/Trigger** model, CloudEvents-native, channels backed by Kafka or in-memory; pairs with Knative Serving scale-to-zero targets.
- **Argo Events**: EventSources (webhook, S3, calendar, Kafka…) ➔ Sensors ➔ trigger **Argo Workflows**/k8s resources — the de facto k8s data-pipeline kickoff.
- **KEDA** (mentioned in your table): not a router but an *event-driven autoscaler* — scales Deployments on queue depth/lag (SQS, Pub/Sub, Kafka, Event Hubs). Routers deliver events; KEDA scales the consumers.

## 4. Differences & Identical Links

**Identical:** rule/filter ➔ target dispatch; at-least-once push with retry + dead-letter; native emission from each cloud's own services; the decoupling benefit (producers know nothing about consumers).

**Different:**
- **Standards**: Eventarc/Event Grid/Knative speak CloudEvents; EventBridge keeps its own envelope (CloudEvents only via transformation).
- **Replay**: only EventBridge has first-class archive/replay at the router.
- **Underlying transport**: Eventarc visibly rides Pub/Sub; EventBridge and Event Grid are opaque.
- **Schema tooling**: EventBridge Schema Registry is unique at this layer.

## 5. Cross-Component Operating Notes

- **The canonical data-arrival chains** (one per cloud — learn as muscle memory):
  - `S3 PUT ➔ EventBridge rule (suffix = .parquet) ➔ Step Functions ➔ Glue job ➔ SNS on failure`
  - `GCS finalize ➔ Eventarc ➔ Cloud Run function ➔ launch Dataflow Flex Template / trigger Composer DAG`
  - `Blob created ➔ Event Grid ➔ ADF event trigger ➔ Databricks notebook activity`
- **Router vs broker decision** (ties to `StreamingMessaging`): need replay, ordering, throughput, consumer offsets ➔ broker (Kinesis/PubSub/Event Hubs/Kafka). Need filtered dispatch to heterogeneous targets ➔ router. Common architecture: broker for the firehose, router for control-plane signals.
- **Auth at the seam**: EventBridge target invocation uses a rule's IAM role (or resource policies on Lambda/SQS); Eventarc uses OIDC service-account push; Event Grid uses webhook validation + managed-identity delivery. Misconfigured invoker permissions are the #1 "my trigger fires but nothing happens" cause on all three.
- **Cost-guardrail automation** (from your notes' billing discussion): all three clouds implement kill-switches *through this layer* — Budgets ➔ SNS/EventBridge ➔ Lambda (stop instances / detach policies); GCP Budget ➔ Pub/Sub ➔ Cloud Run function ➔ **disable billing** (nuclear); Azure Budget ➔ Action Group ➔ Automation Runbook (deallocate VMs). The event-router pattern is exactly how you build the missing "stop my spend" button.

## 6. Certification Checkpoints

- **DEA-C01**: EventBridge rules/patterns, Pipes vs Rules, S3-to-EventBridge, DLQs, Scheduler for cron pipelines.
- **GCP PDE**: Eventarc trigger anatomy, Audit-Log sources, Pub/Sub underpinnings, function-launch patterns.
- **AZ-900**: Event Grid vs Event Hubs vs Service Bus triage (near-guaranteed question).
- **CKA**: not tested directly; Argo Events/KEDA round out event-driven k8s literacy.
