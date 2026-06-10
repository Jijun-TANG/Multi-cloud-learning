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

### AWS CloudWatch
- Metrics: namespaces/dimensions; standard 5-min vs detailed 1-min vs high-res 1-s custom; percentile statistics (p99 — use for latency, never averages).
- Logs: groups/streams, retention (default: forever = silent cost), metric filters (log pattern ➔ metric ➔ alarm), Logs Insights query language; subscription filters stream logs to Kinesis/Firehose/Lambda (the log-export pipeline pattern).
- Alarms: threshold/anomaly-detection; composite alarms reduce noise; actions = SNS, Auto Scaling, EC2 stop/terminate, EventBridge.
- **CloudTrail vs CloudWatch**: *who did what API call* vs *how the system performs* — separated on purpose; exam staple. Data-engineering hooks: Glue job metrics, Kinesis `IteratorAge` (consumer lag!), Redshift query monitoring rules, MWAA environment metrics.

### Google Cloud Observability
- Cloud Monitoring metric **scopes** can aggregate many projects into one pane — pairs with GCP's project-per-environment philosophy.
- Cloud Logging: **_Required / _Default** buckets, **log sinks** route to GCS (archive), **BigQuery (SQL over logs — the PDE-relevant move)**, or Pub/Sub (stream to SIEM); exclusion filters tame cost.
- Log-based metrics ➔ alerting policies; Cloud Trace/Profiler built-in; Error Reporting groups exceptions automatically.

### Azure Monitor
- Two stores: **Metrics** (fast time-series) and **Log Analytics workspaces** queried with **KQL** (`requests | summarize count() by bin(timestamp, 5m)`) — KQL literacy is the real skill.
- **Application Insights** = APM (requests, dependencies, exceptions, distributed traces) — now a Monitor feature on workspaces.
- Alert rules (metric, log, activity-log) fire **Action Groups** (email/SMS/webhook/Function/Runbook/ITSM).
- AZ-900 trio to disambiguate: **Azure Monitor** (telemetry) vs **Service Health** (Microsoft's platform incidents) vs **Advisor** (recommendations: cost/security/reliability).

### Kubernetes observability (CKA workflow)
- No built-in suite (your table says it best). Standard stack: **metrics-server** (powers `kubectl top` + HPA) ➔ Prometheus Operator (ServiceMonitors scrape `/metrics`) ➔ Alertmanager ➔ Grafana; logs via Fluent Bit DaemonSet ➔ backend; traces via OpenTelemetry.
- **Exam-day debugging order**: `kubectl get events --sort-by=.lastTimestamp` ➔ `kubectl describe pod` ➔ `kubectl logs (-p for previous crash)` ➔ `kubectl exec` ➔ node-level `journalctl -u kubelet`. Container logs are just stdout/stderr files under `/var/log/pods` — kubelet keeps them only while the pod exists; shipping them off-node is why FluentBit exists.
- On EKS/GKE/AKS, control-plane logs are opt-in cloud logs (API server, audit, scheduler) — enable them before you need them.

## 4. Differences & Identical Links

**Identical:** agent ➔ store ➔ query ➔ alert ➔ action pipeline; Prometheus has won metrics (all three sell managed Prometheus + managed Grafana); OpenTelemetry is converging tracing; logs-retention is the universal silent cost bomb.

**Different:**
- **Query languages**: Logs Insights syntax vs Logging filter syntax vs **KQL** (the deepest of the three) vs PromQL/LogQL.
- **Architecture**: Azure splits metric store from log workspace explicitly; AWS folds everything under one brand; GCP organizes by scopes/sinks.
- k8s gives you *primitives only* — the assembly burden the clouds remove.

## 5. Cross-Component Operating Notes

- **Budget alerts are passive everywhere** (your `preliminary_chat.md` deep-dive, operationalized):
  - AWS: Budgets ➔ **Budget Actions** can natively stop tagged EC2/RDS or attach deny-policies — the only semi-native kill-switch.
  - GCP: Budget ➔ Pub/Sub ➔ Cloud Run function ➔ *unlink billing account* (nuclear: kills the whole project) — there is **no** native cap.
  - Azure: free/trial subscriptions have a real **Spending Limit**; pay-as-you-go does not — wire Budget ➔ Action Group ➔ Automation Runbook to deallocate.
  - Sandbox rules: alert at $5/$10, destroy resources before closing the laptop, never trust automation alone.
- **Pipeline health wiring** (data certs): Airflow ➔ `on_failure_callback`/SLA-miss ➔ SNS (MWAA) or Cloud Monitoring alert (Composer metrics); Glue job failure ➔ EventBridge rule ➔ SNS; Dataflow ➔ job metrics (system lag, watermark age) ➔ alerting policy; ADF ➔ Monitor alerts on pipeline-failed events.
- **Lag is the metric that matters in streaming**: Kinesis `GetRecords.IteratorAgeMilliseconds`, Pub/Sub `oldest_unacked_message_age`, Event Hubs consumer lag, Kafka consumer group lag — same signal, four names; alert on it everywhere.
- **Autoscaling is observability-driven**: CloudWatch alarms drive ASGs; HPA reads metrics-server/custom metrics; KEDA reads queue depth — the reconciliation loops from `ComputeComponents`/`k8sServices` are consumers of this repo's data.

## 6. Certification Checkpoints

- **AZ-900**: Monitor vs Service Health vs Advisor, Application Insights purpose, Cost Management & budgets, TCO calculator.
- **CKA**: the kubectl debugging sequence cold; metrics-server for `kubectl top`/HPA; where logs live on nodes.
- **DEA-C01**: CloudWatch alarms on pipeline metrics, IteratorAge, Logs Insights, CloudTrail for data-access auditing, Budget Actions.
- **GCP PDE**: log sinks to BigQuery, log-based metrics, monitoring Dataflow/Composer, audit logs for data access.
