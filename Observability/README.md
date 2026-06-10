# Observability — CloudWatch · Google Cloud Observability · Azure Monitor · Prometheus/Grafana

> **New repository** — promoted from four rows of `Important_Table.md` (unified suite, metrics engine, log shippers, visualization) into one coherent repo, plus the billing/budget-alert knowledge from `preliminary_chat.md`.
>
> Certifications served: **AZ-900** (Monitor, Advisor, Service Health — tested), **CKA** (logs/metrics troubleshooting workflow), **DEA-C01/PDE** (pipeline monitoring & cost observability).

## 1. Concept-First Blueprint

Three signals + one wallet:

```text
METRICS (numeric time series)  ─► alerting & autoscaling
LOGS    (structured events)    ─► search, audit, debugging
TRACES  (request causality)    ─► latency decomposition across services
COST    (billing telemetry)    ─► budgets & (DIY) kill-switches
```

Cloud-native twist: agents/sidecars ship signals; managed backends store them; **alerting is passive by default — action requires wiring** (see §5).

## 2. Rosetta Stone

| Concept | AWS | GCP | Azure | k8s/OSS |
| --- | --- | --- | --- | --- |
| Unified suite | **CloudWatch** | **Google Cloud Observability** (ex-Stackdriver) | **Azure Monitor** | none built-in (assemble your own) |
| Metrics store | CloudWatch Metrics | Cloud Monitoring | Monitor Metrics | **Prometheus** |
| Managed Prometheus | **AMP** | **GMP** | Azure Monitor managed Prometheus | kube-prometheus-stack |
| Logs store + query | CloudWatch Logs + **Logs Insights** | Cloud Logging + Log Analytics | **Log Analytics workspace + KQL** | Loki / Elasticsearch |
| Node/VM agent | CloudWatch Agent / FireLens | Ops Agent (Fluent Bit-based) | Azure Monitor Agent (AMA) | **Fluent Bit / FluentD** DaemonSet |
| Tracing | **X-Ray** | Cloud Trace | **Application Insights** | Jaeger / Tempo + **OpenTelemetry** |
| Dashboards | CloudWatch / **Amazon Managed Grafana** | Cloud dashboards | Workbooks / **Azure Managed Grafana** | **Grafana** |
| Alert ➔ action | Alarm ➔ SNS / EventBridge / Auto Scaling | Alerting ➔ notification channels / Pub/Sub | Alert rule ➔ **Action Group** | Alertmanager ➔ webhook/pager |
| Audit trail | **CloudTrail** | **Cloud Audit Logs** | **Activity Log** | API server audit logs |
| Cost watch | Cost Explorer + **Budgets (+ Budget Actions)** | Billing reports + Budgets (alert-only) | Cost Management + Budgets | Kubecost / OpenCost |

## 3. Per-Provider Deep Dive

### AWS CloudWatch + X-Ray
- **CloudWatch Metrics**: namespaces/dimensions; standard 5-min vs detailed 1-min vs high-res 1-s custom; percentile statistics (p99 — use for latency, never averages).
- **CloudWatch Logs**: groups/streams, retention (default: forever = silent cost), metric filters (log pattern ➔ metric ➔ alarm), Logs Insights query language; subscription filters stream logs to Kinesis/Firehose/Lambda (the log-export pipeline pattern).
- **CloudWatch Alarms**: threshold/anomaly-detection; composite alarms reduce noise; actions = SNS, Auto Scaling, EC2 stop/terminate, EventBridge.
- **CloudTrail vs CloudWatch**: *who did what API call* vs *how the system performs* — separated on purpose; exam staple. Data-engineering hooks: Glue job metrics, Kinesis `IteratorAge` (consumer lag!), Redshift query monitoring rules, MWAA environment metrics.

#### Amazon X-Ray — distributed tracing
X-Ray is AWS's **distributed tracing** service: it follows a request as it fans out across microservices, Lambda functions, API Gateway, DynamoDB, S3, SQS, and any HTTP/SDK call that is instrumented.

**How it works:**
1. **Instrumentation**: the X-Ray SDK (or the AWS Distro for OpenTelemetry — ADOT, the modern path) wraps outgoing HTTP/SDK calls and records a **segment** (this service's work) with **subsegments** (each downstream call). For Lambda, the X-Ray SDK is built into the runtime — just enable active tracing.
2. **Sampling**: X-Ray applies a **sampling rule** (default: 1 req/sec + 5% of additional) to avoid flooding the service. You can tune per-service or by response status / URL. For data pipelines, increase the sample rate on critical paths.
3. **Trace assembly**: the X-Ray service receives segments from every instrumented hop, assembles them into a **trace** (a tree of segments with timing, errors, and annotations), and stores them for 30 days.
4. **Service map**: X-Ray auto-generates a **service map** (a live topology diagram showing nodes, edges, latency percentiles, and error rates). This is the "where is the bottleneck" view — invaluable when a Glue job calls Lambda which calls DynamoDB and one hop is slow.
5. **Annotations & metadata**: key-value pairs you attach to segments (e.g., `customer_id`, `job_id`). They are indexed and searchable via `GetTraceSummaries` — use them to trace a single data-pipeline execution end-to-end.
6. **Trace ID propagation**: the `_X_AMZN_TRACE_ID` header carries the trace context across HTTP calls. AWS SDKs propagate it automatically; for cross-cloud or on-prem calls you must forward it manually.

**X-Ray vs CloudWatch vs CloudTrail (the three-way disambiguate):**
- **CloudWatch** = *how the system performs* (metrics, logs, alarms).
- **CloudTrail** = *who did what API call* (audit, compliance).
- **X-Ray** = *how a single request flowed through the system* (latency decomposition, error root cause across services).

**X-Ray integrations relevant to data engineering:**
- Lambda (enable "Active tracing" — captures cold start duration as a separate subsegment).
- API Gateway ➔ Lambda ➔ DynamoDB: see the full fan-out latency.
- Step Functions: trace a workflow execution across all state transitions.
- EKS/ECS: deploy the **X-Ray daemon** as a sidecar DaemonSet; the SDK sends segments to it via UDP (port 2000), the daemon batches and forwards to the X-Ray API.
- **Pricing**: first 100k traces/month free; $5/million records recorded, $0.50/million records retrieved. Traces are short-lived (30 days) — export via CloudWatch Logs or the X-Ray API if you need longer retention.

### Google Cloud Observability + Cloud Trace
- **Cloud Monitoring metric scopes** can aggregate many projects into one pane — pairs with GCP's project-per-environment philosophy.
- **Cloud Logging**: **_Required / _Default** buckets, **log sinks** route to GCS (archive), **BigQuery (SQL over logs — the PDE-relevant move)**, or Pub/Sub (stream to SIEM); exclusion filters tame cost.
- **Log-based metrics** ➔ alerting policies; **Cloud Trace/Profiler** built-in; Error Reporting groups exceptions automatically.

#### Google Cloud Trace — the X-Ray equivalent
Cloud Trace is GCP's distributed tracing system. It is **the direct architectural equivalent of AWS X-Ray** and works on the same segment/trace model.

**How it works:**
1. **Instrumentation**: the OpenTelemetry SDK (recommended) or the legacy Cloud Trace client library instruments outgoing calls. On GKE, the **Cloud Trace agent** (a DaemonSet) auto-instruments many Google-service calls. Cloud Run functions and App Engine have built-in tracing — no SDK needed.
2. **Trace context propagation**: uses the `X-Cloud-Trace-Context` header (W3C Trace Context compatible). Google services propagate it automatically; your code must forward it for external calls.
3. **Sampling**: per-project default is low; you can configure per-service via the Cloud Trace console or Terraform. Latency-based sampling captures slow traces preferentially.
4. **Analysis**: the Trace viewer shows a waterfall timeline per trace, with latency per span. **Trace analytics** (filter by latency percentile, error rate, method) and **trace-based metrics** (latency distributions exported to Cloud Monitoring for alerting).
5. **Service-to-service topology**: like X-Ray's service map, Cloud Trace auto-generates a dependency graph.

**Cloud Trace vs X-Ray — key differences:**
- **Standard**: Cloud Trace adopted W3C Trace Context early; X-Ray uses its own `_X_AMZN_TRACE_ID` header (though ADOT now bridges both).
- **Agent model**: Cloud Trace on GKE uses a DaemonSet agent; X-Ray uses a sidecar daemon on EKS/ECS. Both batch-and-forward over UDP.
- **GCP-native depth**: Cloud Trace has deeper auto-instrumentation for Google services (Cloud Run, GKE, Pub/Sub, BigQuery, Spanner) out of the box. X-Ray has deeper AWS-native depth (Lambda, Step Functions, DynamoDB).
- **Pricing**: first 2.5 million spans/month free (very generous); after that ~$0.20/million spans. Cheaper than X-Ray at scale.

### Azure Monitor + Application Insights
- Two stores: **Metrics** (fast time-series) and **Log Analytics workspaces** queried with **KQL** (`requests | summarize count() by bin(timestamp, 5m)`) — KQL literacy is the real skill.
- **Application Insights** = APM (requests, dependencies, exceptions, distributed traces) — now a Monitor feature on workspaces.
- Alert rules (metric, log, activity-log) fire **Action Groups** (email/SMS/webhook/Function/Runbook/ITSM).
- AZ-900 trio to disambiguate: **Azure Monitor** (telemetry) vs **Service Health** (Microsoft's platform incidents) vs **Advisor** (recommendations: cost/security/reliability).

#### Azure Application Insights — the X-Ray/Cloud Trace equivalent
Application Insights is Azure's **APM + distributed tracing** service. It is broader than X-Ray or Cloud Trace alone because it combines metrics, logs, and traces in one store — but its **distributed tracing** capability is the direct equivalent.

**How it works:**
1. **Instrumentation**: the Application Insights SDK (or **OpenTelemetry** via the Azure Monitor OpenTelemetry Distro — the modern path) auto-collects HTTP requests, dependencies (SQL, HTTP, Azure SDK calls), exceptions, and custom telemetry. For Azure Functions, App Service, and AKS, enable via a single app-setting (`APPINSIGHTS_INSTRUMENTATIONKEY` or `APPLICATIONINSIGHTS_CONNECTION_STRING`).
2. **Distributed tracing / Application Map**: Application Insights auto-correlates telemetry across services using the `traceparent` W3C header. It builds a live **Application Map** (topology diagram with latency, failure rates, call counts) — the Azure equivalent of X-Ray's service map and Cloud Trace's dependency graph.
3. **Live Metrics Stream**: real-time latency, request-rate, and failure-rate dashboard — unique to Azure; great for watching a deployment or a pipeline run.
4. **Smart Detection**: ML-driven anomaly detection on failure rates and latency — fires automatically, no threshold configuration needed.
5. **Profiler & Snapshot Debugger**: captures call stacks during slow requests and exception snapshots — deeper than X-Ray/Cloud Trace alone (they need separate services for profiling).
6. **Log Analytics integration**: all trace data lands in the same Log Analytics workspace as your logs and metrics, so you can KQL-join traces with logs (`requests | join kind=inner (traces) on operation_Id`) — a unified query surface the other two clouds split across services.

**Application Insights vs X-Ray vs Cloud Trace — key differences:**
- **Breadth**: Application Insights is APM + tracing + logging + metrics in one. X-Ray and Cloud Trace are tracing-only; you pair them with CloudWatch/Cloud Monitoring for the rest.
- **Standard**: Application Insights adopted W3C Trace Context (`traceparent`) natively; X-Ray uses its own header (ADOT bridges); Cloud Trace uses `X-Cloud-Trace-Context` (W3C-compatible).
- **Auto-instrumentation depth**: Application Insights has the richest auto-instrumentation for Azure-native services (Functions, App Service, AKS, Service Bus, Cosmos DB). X-Ray wins on AWS-native depth; Cloud Trace wins on GCP-native depth.
- **Pricing**: pay per GB of data ingested (same Log Analytics pricing). First 5 GB/month free. Can be high-volume if you trace everything — use sampling aggressively.
- **Retention**: 90 days by default (configurable). X-Ray: 30 days. Cloud Trace: 30 days. Azure wins on retention out of the box.

### Kubernetes observability (CKA workflow)
- No built-in suite (your table says it best). Standard stack: **metrics-server** (powers `kubectl top` + HPA) ➔ Prometheus Operator (ServiceMonitors scrape `/metrics`) ➔ Alertmanager ➔ Grafana; logs via Fluent Bit DaemonSet ➔ backend; traces via OpenTelemetry.
- **Exam-day debugging order**: `kubectl get events --sort-by=.lastTimestamp` ➔ `kubectl describe pod` ➔ `kubectl logs (-p for previous crash)` ➔ `kubectl exec` ➔ node-level `journalctl -u kubelet`. Container logs are just stdout/stderr files under `/var/log/pods` — kubelet keeps them only while the pod exists; shipping them off-node is why FluentBit exists.
- On EKS/GKE/AKS, control-plane logs are opt-in cloud logs (API server, audit, scheduler) — enable them before you need them.

## 4. Differences & Identical Links

**Identical:** agent ➔ store ➔ query ➔ alert ➔ action pipeline; Prometheus has won metrics (all three sell managed Prometheus + managed Grafana); **OpenTelemetry is converging tracing** (all three clouds now offer an OTel-based distro — ADOT, Azure Monitor OTel Distro, OpenTelemetry on GKE); logs-retention is the universal silent cost bomb.

**Different:**
- **Query languages**: Logs Insights syntax vs Logging filter syntax vs **KQL** (the deepest of the three) vs PromQL/LogQL.
- **Architecture**: Azure splits metric store from log workspace explicitly; AWS folds everything under one brand; GCP organizes by scopes/sinks.
- k8s gives you *primitives only* — the assembly burden the clouds remove.

### Tracing-specific comparison (X-Ray vs Cloud Trace vs Application Insights)

| Dimension | X-Ray (AWS) | Cloud Trace (GCP) | Application Insights (Azure) |
| --- | --- | --- | --- |
| Primary role | Distributed tracing only | Distributed tracing only | APM + tracing + metrics + logs in one |
| Trace context header | `_X_AMZN_TRACE_ID` (proprietary; ADOT adds W3C) | `X-Cloud-Trace-Context` (W3C-compatible) | `traceparent` (W3C native) |
| Service topology | **Service map** (auto-generated) | **Dependency graph** (auto-generated) | **Application Map** (auto-generated + richer) |
| Agent model | Sidecar DaemonSet on EKS/ECS; SDK built into Lambda | DaemonSet agent on GKE; auto-agent for Cloud Run/App Engine | Agent SDK / OTel distro; auto-instrumentation for Functions/App Service |
| Profiling | Separate service (CloudProfiler on GCP, CodeGuru on AWS) | Built-in (**Cloud Trace Profiler**) | Built-in (**Profiler + Snapshot Debugger**) |
| Log ↔ trace join | Separate services (X-Ray + CloudWatch Logs) | Separate services (Trace + Logging) | **Same workspace** (KQL join on `operation_Id`) |
| Pricing free tier | 100k traces/month recorded + 1M retrieved | 2.5M spans/month | 5 GB telemetry/month |
| Retention | 30 days | 30 days | 90 days (default) |
| Best on | AWS-native services (Lambda, Step Functions, DynamoDB, API Gateway) | GCP-native services (Cloud Run, GKE, Pub/Sub, BigQuery) | Azure-native services (Functions, App Service, AKS, Cosmos DB) |
| k8s deployment | EKS: X-Ray daemon DaemonSet as sidecar | GKE: Cloud Trace agent DaemonSet (built into GKE with Operations suite) | AKS: Azure Monitor AMA + OTel DaemonSet or auto-instrumentation extension |

**Key insight for multi-cloud architects**: all three are converging on **OpenTelemetry** as the vendor-neutral instrumentation layer. If you instrument once with OTel, you can export traces to X-Ray, Cloud Trace, Application Insights, Jaeger, or any other backend — this is the strategic answer for "which tracing tool do I learn?" in 2025+.

## 5. Cross-Component Operating Notes

- **Budget alerts are passive everywhere** (your `preliminary_chat.md` deep-dive, operationalized):
  - AWS: Budgets ➔ **Budget Actions** can natively stop tagged EC2/RDS or attach deny-policies — the only semi-native kill-switch.
  - GCP: Budget ➔ Pub/Sub ➔ Cloud Run function ➔ *unlink billing account* (nuclear: kills the whole project) — there is **no** native cap.
  - Azure: free/trial subscriptions have a real **Spending Limit**; pay-as-you-go does not — wire Budget ➔ Action Group ➔ Automation Runbook to deallocate.
  - Sandbox rules: alert at $5/$10, destroy resources before closing the laptop, never trust automation alone.
- **Pipeline health wiring** (data certs): Airflow ➔ `on_failure_callback`/SLA-miss ➔ SNS (MWAA) or Cloud Monitoring alert (Composer metrics); Glue job failure ➔ EventBridge rule ➔ SNS; Dataflow ➔ job metrics (system lag, watermark age) ➔ alerting policy; ADF ➔ Monitor alerts on pipeline-failed events.
- **Lag is the metric that matters in streaming**: Kinesis `GetRecords.IteratorAgeMilliseconds`, Pub/Sub `oldest_unacked_message_age`, Event Hubs consumer lag, Kafka consumer group lag — same signal, four names; alert on it everywhere.
- **Autoscaling is observability-driven**: CloudWatch alarms drive ASGs; HPA reads metrics-server/custom metrics; KEDA reads queue depth — the reconciliation loops from `ComputeComponents`/`k8sServices` are consumers of this repo's data.
- **Tracing the data pipeline end-to-end** (cross-component): instrument the full chain — EventBridge/Eventarc/Event Grid ➔ Lambda/Cloud Run function/Function ➔ Glue/EMR/Dataflow/Databricks ➔ S3/GCS/ADLS ➔ Redshift/BigQuery/Synapse. X-Ray, Cloud Trace, and Application Insights each show a different slice; OTel is the only way to get one unified trace across clouds. For DEA-C01/PDE, know which service map / application map to consult when a pipeline stage is slow.
- **Tracing on k8s**: deploy the OTel collector as a DaemonSet + the appropriate cloud exporter (X-Ray exporter, Cloud Trace exporter, Application Insights exporter). The collector receives spans from instrumented pods via OTLP and forwards to the cloud backend. This is the standard pattern for EKS/GKE/AKS workloads that need cloud-native tracing without per-sidecar daemons.

## 6. Certification Checkpoints

- **AZ-900**: Monitor vs Service Health vs Advisor, Application Insights purpose, Cost Management & budgets, TCO calculator.
- **CKA**: the kubectl debugging sequence cold; metrics-server for `kubectl top`/HPA; where logs live on nodes; OTel collector DaemonSet for tracing.
- **DEA-C01**: CloudWatch alarms on pipeline metrics, IteratorAge, Logs Insights, CloudTrail for data-access auditing, Budget Actions, X-Ray service maps for pipeline latency, sampling rules, X-Ray daemon on EKS.
- **GCP PDE**: log sinks to BigQuery, log-based metrics, monitoring Dataflow/Composer, audit logs for data access, Cloud Trace for Dataflow/Composer latency analysis, W3C trace context propagation.
- **Tracing (all certs)**: know the three-way X-Ray / Cloud Trace / Application Insights comparison table above; know that OpenTelemetry is the convergence path; know that Azure Application Insights is the broadest (APM + tracing + logs in one) while X-Ray and Cloud Trace are tracing-only.
