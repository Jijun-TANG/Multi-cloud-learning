# Data Processing — Glue/EMR/Flink · Dataflow/Dataproc · Databricks/Synapse Spark · Spark/Flink on k8s

> **New repository** — added because batch/stream transformation engines are the largest scored domain on DEA-C01 and GCP PDE; the original layout listed them only as one row in the master table.
>
> Certifications served: **DEA-C01** (Glue vs EMR — pervasive), **GCP PDE** (Dataflow vs Dataproc — pervasive), **AZ-900** (Databricks/Synapse awareness), **CKA** (Spark-on-k8s pattern).

## 1. Concept-First Blueprint

Distributed processing = **a driver/coordinator planning a DAG of tasks over partitioned data, executed by ephemeral workers, with shuffle as the expensive step**.

```text
Read (lake/stream) ─► Transform DAG (map ➔ SHUFFLE ➔ reduce) ─► Write (lake/warehouse)
```

Two engine families:
- **Spark** (micro-batch heritage, unified batch + Structured Streaming) — Glue, EMR, Dataproc, Databricks, Synapse Spark.
- **Beam/Flink** (true streaming, event-time first) — Dataflow (Beam), Managed Flink, Flink operator.

The universal streaming vocabulary regardless of engine: **event time vs processing time, watermarks, windows (tumbling/sliding/session), triggers, late data, checkpointing, exactly-once sinks**.

## 2. Rosetta Stone

| Concept | AWS | GCP | Azure | k8s/OSS |
| --- | --- | --- | --- | --- |
| Serverless Spark ETL | **AWS Glue** | **Dataproc Serverless** / BigQuery Spark | Synapse Spark pools / Fabric | Spark Operator |
| Managed Hadoop/Spark cluster | **EMR** (EC2/EKS/Serverless) | **Dataproc** | HDInsight (legacy) / **Databricks** | Spark on k8s |
| True-streaming engine | **Managed Service for Apache Flink** | **Dataflow** (Apache Beam) | Stream Analytics / Flink on HDInsight | Flink Operator |
| Premium lakehouse platform | Databricks on AWS | Databricks on GCP | **Azure Databricks** (first-party!) | — |
| Capacity unit | DPU (Glue) / instances (EMR) | workers (auto) | DBU (Databricks) / Spark pool vCores | executor pods |
| Schema/catalog | **Glue Data Catalog** + crawlers | Dataplex / BigQuery metastore / Hive on Dataproc | Unity Catalog / Purview | Hive Metastore |
| Visual/no-code ETL | Glue Studio / DataBrew | Data Fusion (CDAP) | ADF Mapping Data Flows | — |

## 3. Per-Provider Deep Dive

### AWS: the Glue-vs-EMR axis (DEA-C01's favorite decision)
- **Glue** = serverless Spark + the Catalog + crawlers + Data Quality + bookmarks. Choose when: no cluster ops, S3-centric ETL, catalog-first. Know: **job bookmarks** (incremental processing state), DynamicFrames vs DataFrames, worker types (G.1X/G.2X/G.8X), Flex execution (cheaper, interruptible), crawlers infer partitions/schemas.
- **EMR** = full open-source stack (Spark, Hive, Presto/Trino, HBase, Flink) with cluster control. Choose when: custom libs/AMIs, long-running clusters, non-Spark engines, spot-fleet cost engineering (task nodes on spot = canonical pattern). Variants: EMR on EC2, **EMR on EKS** (share k8s clusters), **EMR Serverless** (closes most of the gap with Glue).
- **Managed Service for Apache Flink** (ex-Kinesis Data Analytics): stateful true streaming over KDS/MSK — answer for "sub-second, event-time, complex state" where Spark micro-batch won't do.

### GCP: the Dataflow-vs-Dataproc axis (PDE's favorite decision)
- **Dataflow** = managed **Apache Beam**: one model for batch + streaming; serverless, autoscaling, **dynamic work rebalancing** (kills stragglers); exactly-once with Pub/Sub + BigQuery; Flex Templates for reuse; Streaming Engine offloads state. *Default answer for new GCP pipelines, especially streaming.*
- **Dataproc** = managed Spark/Hadoop: choose for **lift-and-shift of existing Spark/Hive code** or when you need the OSS ecosystem. Exam heuristics: existing Hadoop ➔ Dataproc; new unified pipeline ➔ Dataflow. Ephemeral-cluster pattern: create ➔ run ➔ delete (storage lives in GCS, never HDFS); preemptible secondary workers for cost.
- Beam vocabulary tested directly: `PCollection`, `ParDo`, side inputs, windowing + triggers + allowed lateness.

### Azure: Databricks-first
- **Azure Databricks** is a *first-party* Azure service (unique among clouds — deeper Entra ID/ADLS integration than Databricks on AWS/GCP). Concepts: workspace, all-purpose vs **job clusters** (always job clusters for production — cheaper, ephemeral), DBU billing, Photon engine, **Unity Catalog** governance, Delta Lake + medallion (bronze/silver/gold) architecture, Auto Loader for incremental file ingestion, DLT (Delta Live Tables/Lakeflow) for declarative pipelines.
- **Synapse Spark pools / Fabric Spark**: Microsoft-native Spark when you're already in the Synapse/Fabric estate. **Stream Analytics**: SQL-on-streams over Event Hubs — the lightweight Flink-ish option (AZ-900-level awareness).

### Spark & Flink on Kubernetes (CKA crossover)
- Spark Operator / `spark-submit --master k8s://`: driver pod spawns executor pods; cluster autoscaler adds nodes; spot/preemptible executors need checkpoint-tolerant jobs. Flink Operator manages JobManager/TaskManager pods + savepoints. EMR on EKS and Dataproc on GKE are the managed wrappers of exactly this.

## 4. Differences & Identical Links

**Identical:** Spark is Spark — DataFrame code ports almost verbatim across Glue/EMR/Dataproc/Databricks/Synapse; shuffle/skew/small-files tuning is universal; "ephemeral compute, durable lake storage" is the rule everywhere; all platforms converge on Parquet + a table format.

**Different:**
- **Engine ideology**: AWS gives you many engines and makes you choose; GCP pushes one model (Beam/Dataflow) hard; Azure outsources the premium experience to Databricks.
- **Streaming depth**: Dataflow and Flink are event-time-native; Spark Structured Streaming is micro-batch (fine for seconds, not milliseconds).
- **Catalog gravity**: Glue Catalog (AWS) vs BigQuery/Dataplex (GCP) vs Unity Catalog (Databricks/Azure) — whoever owns the catalog owns governance.

## 5. Cross-Component Operating Notes

- **Trigger chains**: S3 ➔ EventBridge ➔ Glue workflow / SFN ➔ EMR steps; GCS ➔ Eventarc ➔ Cloud Run function ➔ Dataflow Flex Template launch; Blob ➔ Event Grid ➔ ADF ➔ Databricks job. (Router layer in `EventRouting`, orchestration in `WorkflowOrchestration`.)
- **Identity at the worker level**: Glue job role / EMR instance profile (or IRSA on EKS) ➔ S3 + Lake Formation; Dataproc/Dataflow worker **service account** ➔ GCS/BigQuery (workers, not your user, must hold the roles — classic failure); Databricks ➔ ADLS via Unity Catalog storage credentials/managed identities.
- **Engine ➔ warehouse handoffs**: Glue/EMR write Parquet to S3 then `COPY`/Spectrum into Redshift; **Dataflow writes BigQuery via Storage Write API natively** (the blessed path); Databricks writes Delta that Fabric/Synapse read directly (DirectLake).
- **Shuffle hardware**: EMR likes instance-store NVMe (see `ComputeComponents`); Dataflow offloads shuffle to its service; Databricks uses local SSDs — same physics, three packagings.
- Streaming checkpoint stores: Spark ➔ DBFS/S3/GCS path; Flink ➔ savepoints in object storage; Dataflow ➔ internal. Losing the checkpoint = reprocessing or data loss; protect those buckets.

## 6. Certification Checkpoints

- **DEA-C01**: Glue vs EMR vs EMR Serverless vs Managed Flink selection, bookmarks, crawlers, DPU sizing, spot task nodes, Glue Data Quality, small-files mitigation.
- **GCP PDE**: Dataflow vs Dataproc selection, Beam windows/watermarks/triggers, dynamic rebalancing, ephemeral Dataproc + GCS, Serverless Spark.
- **AZ-900**: Databricks and Synapse exist and what they're for; ADF Data Flows as no-code option.
- **CKA**: driver/executor pods, operator pattern, resource requests for batch jobs, node autoscaling under burst.
