# Workflow Orchestration — MWAA · Cloud Composer · Data Factory · Airflow/Argo

> Certifications served: **DEA-C01** (orchestration domain), **GCP PDE** (Composer — core service), **AZ-900** (ADF awareness), **CKA** (Airflow-on-k8s executors as a workload pattern).

## 1. Concept-First Blueprint

An orchestrator is a **dependency-aware scheduler**: it holds a DAG of tasks, decides what runs when (schedule/event/backfill), dispatches tasks to *other* compute, tracks state, retries failures, and alerts.

```text
DAG definition ─► Scheduler ─► Task dispatch (to external engines!) ─► State store ─► Retry/Alert/Backfill
```

**Golden rule: the orchestrator orchestrates; it does not process.** Heavy lifting belongs in Glue/EMR/BigQuery/Dataflow/Databricks. Tasks should be thin API calls + sensors/polls.

## 2. Rosetta Stone

| Concept | AWS | GCP | Azure | Kubernetes/OSS |
| --- | --- | --- | --- | --- |
| Managed Airflow | **MWAA** | **Cloud Composer** | (Marketplace/Astronomer; ADF instead) | Airflow Helm chart |
| Native serverless state machine | **Step Functions** | **Cloud Workflows** | **Logic Apps** / Durable Functions | **Argo Workflows** |
| GUI-first ETL orchestrator | Glue Workflows | (Composer/Data Fusion) | **Azure Data Factory (ADF)** | — |
| DAG/definition language | Python (Airflow) / ASL JSON (SFN) | Python (Airflow) / YAML (Workflows) | JSON pipelines + visual designer | Python / YAML CRs |
| Event-triggered start | EventBridge ➔ SFN/MWAA | Eventarc/Functions ➔ Composer API | ADF event triggers (Event Grid) | Argo Events |
| Cron schedule | EventBridge Scheduler / Airflow | Cloud Scheduler / Airflow | ADF schedule/tumbling-window triggers | CronWorkflow |

## 3. Per-Provider Deep Dive

### Apache Airflow (the substrate — learn once, use thrice)
- Components: **scheduler** (parses DAGs, queues tasks), **executor** (Celery/Kubernetes), **workers**, **webserver**, **metadata DB** (the Airflow "etcd": every task state lives here).
- Core concepts to master: operators vs sensors (and `mode="reschedule"` to free worker slots), hooks & **connections**, XCom (small metadata only — never DataFrames), `catchup`/backfill semantics, `data_interval`/logical date, pools for concurrency limits, deferrable operators.

### AWS MWAA
- Architecture: Airflow on Fargate inside *your* VPC; **DAGs/plugins/requirements come from an S3 bucket** — deployment = `aws s3 cp dag.py s3://bucket/dags/`.
- The **environment execution role** is what operators use to call Glue/EMR/Redshift/Athena. Secrets backend ➔ Secrets Manager (`airflow/connections/*` naming convention).
- Typical DEA-C01 pattern: `GlueJobOperator ➔ GlueCrawlerOperator ➔ RedshiftDataOperator ➔ SnsPublishOperator`.
- **Step Functions vs MWAA decision**: SFN = serverless, per-state-transition billing, tight AWS-service integration (`.sync` integration patterns wait for Glue/EMR completion), JSON ASL; MWAA = Python ecosystem, backfills, complex dependencies, always-on cost. The exam loves this trade-off.

### GCP Cloud Composer
- **Composer is Airflow running on a GKE cluster in a tenant/customer project**, with the metadata DB in Cloud SQL and the webserver behind IAP. DAGs deploy by **copying to a GCS bucket** (`/dags`) — the GCS bucket is FUSE-mounted into the GKE pods.
- Composer 2/3 = autoscaling workers (KubernetesExecutor lineage); each task can run as a pod.
- **How Composer calls BigQuery (the canonical chain, exam-ready):**
  1. DAG uses `BigQueryInsertJobOperator` (google provider package).
  2. The operator's hook obtains credentials via **ADC**: the worker pod's k8s ServiceAccount is mapped through **Workload Identity** to the **Composer environment's GCP service account**.
  3. That service account must hold `roles/bigquery.jobUser` (project) + `roles/bigquery.dataEditor` (dataset).
  4. Hook submits a **BigQuery job** (jobs.insert) and polls `jobs.get` until DONE — *BigQuery slots do the SQL work, not Airflow workers* (deferrable mode releases the worker while waiting).
  5. Job results/errors propagate back to task state; lineage can flow to Dataplex/Data Catalog.
- Same pattern for `DataflowStartFlexTemplateOperator`, `DataprocSubmitJobOperator` — Composer is a thin, authenticated API client.

### Azure Data Factory (different beast: not Airflow)
- Visual/JSON pipelines of **activities**; "DAG" = activity dependency graph with success/failure/completion conditions.
- Key vocabulary: **Linked Service** (connection ≈ Airflow connection), **Dataset** (pointer to data shape/location), **Integration Runtime** (the compute doing movement: Azure IR = managed, Self-Hosted IR = your VM for on-prem/private sources — AZ-900-level fact), **Triggers** (schedule, tumbling window — uniquely supports stateful, catch-up-style intervals like Airflow backfill — and event-based via Event Grid).
- Heavy transforms delegate to **Mapping Data Flows** (managed Spark), Databricks notebook activities, Synapse, or HDInsight. Also hosts a **Managed Airflow** ("Workflow Orchestration Manager") for parity.
- Auth: ADF's **Managed Identity** is granted RBAC on Storage/Synapse/Key Vault — zero stored credentials.

### Kubernetes-native: Argo Workflows
- Workflows as CRDs; each step is a pod; artifacts pass via S3/GCS; Argo Events triggers from queues/webhooks. The CKA-adjacent mental model: an orchestrator built *entirely* from reconciliation loops.

## 4. Differences & Identical Links

**Identical:** DAG dependency semantics, retries/backoff, schedule+event+manual triggering, connections/credentials externalized to secret stores, "orchestrate-don't-process."

**Different:**
- **Authoring**: Python-as-code (Airflow everywhere) vs JSON state machines (SFN) vs visual designer (ADF/Logic Apps).
- **Runtime substrate**: Fargate (MWAA) vs GKE (Composer) vs fully abstracted multitenant service (ADF/SFN).
- **DAG deployment**: S3 bucket (MWAA) vs GCS bucket (Composer) vs Git/ARM (ADF) — same gesture, different store.
- **Billing**: MWAA/Composer bill for always-on environments (idle costs!); SFN/Workflows/Logic Apps bill per transition/execution — a recurring cost-scenario question.

## 5. Cross-Component Operating Notes

- **MWAA ➔ Glue/EMR/Redshift**: operators call AWS APIs with the environment execution role; use deferrable operators or SFN `.sync` to avoid paying workers to sleep. EMR pattern: `EmrCreateJobFlowOperator ➔ EmrAddStepsOperator ➔ EmrStepSensor ➔ terminate`.
- **Composer ➔ Dataflow/BigQuery/Dataproc**: see the credential chain above; for cross-project calls grant the *environment SA* roles in the target project.
- **ADF ➔ Databricks/Synapse**: linked service with managed identity or Databricks PAT from Key Vault; Data Flow debug clusters bill while idle — turn off.
- **Event-driven kickoff** (ties to `EventRouting`): S3 upload ➔ EventBridge ➔ SFN/MWAA REST API; GCS object ➔ Eventarc ➔ Cloud Run function ➔ Composer DAG-trigger API; Blob created ➔ Event Grid ➔ ADF event trigger.
- **Airflow on k8s (CKA crossover)**: KubernetesExecutor spawns one pod per task — debugging a stuck task = debugging a Pending pod (resources, taints, image pulls).

## 6. Certification Checkpoints

- **DEA-C01**: MWAA vs Step Functions vs Glue Workflows selection, EventBridge integration, retry/error handling in SFN (Catch/Retry blocks).
- **GCP PDE**: Composer architecture (GKE+GCS+Cloud SQL), BigQuery/Dataflow operators, environment SA roles, Composer 2 autoscaling.
- **AZ-900**: what ADF is (cloud ETL/data integration), Logic Apps vs Functions distinction.
- **CKA**: not directly tested; Argo/KubernetesExecutor reinforce pod lifecycle skills.
