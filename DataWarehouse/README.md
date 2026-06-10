# Data Warehouse — Redshift · BigQuery · Synapse/Fabric · Trino/ClickHouse

> Certifications served: **DEA-C01** (Redshift — major domain), **GCP PDE** (BigQuery — the single most-tested service), **AZ-900** (Synapse awareness), **CKA** (Trino/ClickHouse on k8s as a pattern only).

## 1. Concept-First Blueprint

A cloud warehouse = **columnar storage + massively parallel (MPP) SQL compute + a separation-of-storage-and-compute economic model**.

```text
Ingest (batch COPY / streaming) ─► Columnar storage (sorted, compressed, partitioned)
        ─► Distributed query engine (slices/slots/DWUs) ─► BI / ML / federated consumers
```

All three converged on the same end-state from different directions: Redshift *added* storage-compute separation (RA3/Serverless), BigQuery was *born* with it (Dremel + Colossus), Synapse split pools / Fabric re-unified on OneLake.

## 2. Rosetta Stone

| Concept | Redshift | BigQuery | Synapse / Fabric | OSS on k8s |
| --- | --- | --- | --- | --- |
| Compute unit | Node slices / RPU (serverless) | **Slots** | DWU (dedicated pool) / Fabric capacity (CU) | Trino workers |
| Provisioning model | Cluster (RA3) or Serverless | Always serverless | Dedicated pool, serverless pool, or Fabric | Helm-deployed pods |
| Physical layout lever | **DISTKEY / SORTKEY** | **Partitioning + Clustering** | Distribution (HASH/ROUND_ROBIN/REPLICATE) + ordered CCI | Connector-dependent |
| Bulk load | `COPY` from S3 | Load jobs (free!) from GCS | `COPY INTO` / PolyBase from ADLS | external tables |
| Streaming ingest | Streaming ingestion (Kinesis/MSK materialized views) | **Storage Write API** | Event Hubs ➔ Fabric Eventstream/Synapse | Kafka ➔ ClickHouse |
| Query the lake in place | **Redshift Spectrum** | **External/BigLake tables** | Serverless SQL pool over ADLS / OneLake shortcuts | Trino's whole reason to exist |
| Cached/accelerated views | Materialized views + autorefresh | Materialized views | Result-set caching, MVs | MVs (ClickHouse) |
| Cross-org sharing | Datashares | Analytics Hub / authorized datasets | Fabric shortcuts / Power BI | — |
| ML in SQL | Redshift ML (➔ SageMaker) | **BQML** (native) | Fabric/Synapse ML integration | — |
| Pricing axis | Node-hours or RPU-seconds | **Bytes scanned** (on-demand) or slot capacity | DWU-hours / capacity units | Infra cost |

## 3. Per-Provider Deep Dive

### Amazon Redshift (DEA-C01 core)
- **RA3** nodes = managed storage in S3 separate from compute; **Serverless** = pay per RPU-second. Legacy DC2 couples storage+compute (know this evolution).
- **Distribution styles**: `KEY` (co-locate join keys), `ALL` (replicate small dims), `EVEN`, `AUTO`. Wrong DISTKEY ➔ network shuffle ("DS_BCAST_INNER" in EXPLAIN = red flag).
- **SORTKEY** enables zone-map pruning (min/max per block — same idea as BigQuery clustering and Parquet row-group stats).
- `COPY` best practices: multiple files (multiple-of-slice-count), compressed, single manifest; `UNLOAD` to export. **Spectrum** queries S3 via the Glue Data Catalog — exam answer for "query exabytes without loading."
- Ops vocabulary: WLM/auto-WLM, concurrency scaling (burst clusters), short query acceleration, `VACUUM`/`ANALYZE`, Data API (HTTP, no JDBC pools — pairs with Lambda).

### Google BigQuery (PDE core)
- Architecture: **Dremel** execution over **Colossus** storage, shuffled via Google's Jupiter network — compute and storage scale independently and invisibly.
- **Cost control is the exam obsession**: partition (by date/integer) + cluster (up to 4 cols) ➔ prune bytes; `--dry_run` for estimates; `maximum_bytes_billed` guard; **avoid `SELECT *`**; long-term storage discount after 90 untouched days; flat-rate/Editions slot reservations for predictable spend.
- Ingest: load jobs are **free** (batch); Storage Write API for exactly-once streaming; federated queries to Cloud SQL/Spanner; external + **BigLake** tables over GCS.
- Governance: dataset/table IAM, **authorized views**, row-level & column-level (policy-tag) security, CMEK.
- **BQML**: `CREATE MODEL` in SQL — regression, classification, k-means, time-series; PDE tests when BQML suffices vs Vertex AI.

### Azure Synapse ➔ Microsoft Fabric
- Synapse = umbrella: **Dedicated SQL pool** (classic MPP, DWUs), **Serverless SQL pool** (pay-per-TB over ADLS), Spark pools, pipelines (ADF engine inside).
- **Fabric** is the strategic successor: SaaS-unified Lakehouse/Warehouse/Power BI on **OneLake**, everything stored as **Delta Lake**, billed by capacity units. AZ-900-level: "Synapse/Fabric = Azure analytics warehouse"; data certs need the pool distinctions.
- Dedicated-pool design mirrors Redshift: hash-distribute big facts, `REPLICATE` small dims, round-robin staging.

### Trino / ClickHouse on Kubernetes
- **Trino** = federated MPP SQL *engine without storage* — queries S3/GCS/ADLS/Iceberg/Kafka in place (the self-hosted Spectrum/serverless-pool). **ClickHouse** = blazing real-time columnar store. Both run via operators/Helm with StatefulSets + PVCs — the k8s storage concepts from `k8sServices` apply directly.

## 4. Differences & Identical Links

**Identical:** columnar + MPP + pruning fundamentals; "co-locate joins, replicate small dims"; lake-in-place querying; materialized views; IAM-governed datasets.

**Different:**
- **What you tune**: Redshift = physical layout (DISTKEY/SORTKEY/WLM); BigQuery = bytes scanned (partition/cluster/SQL hygiene); Synapse = distribution + pool sizing. BigQuery has *no* indexes/keys to manage — its serverlessness is real but moves the discipline into query cost.
- **Pricing axis** (the big one): node-hours (Redshift provisioned) vs bytes-scanned (BQ on-demand) vs DWU/capacity (Azure). Identical workloads can differ 10× in cost purely by model fit.
- Streaming ingest maturity: BigQuery Storage Write API is the most native; Redshift streaming MVs newer; Synapse leans on Event Hubs/Fabric.

## 5. Cross-Component Operating Notes

- **Loader identity chains**: Redshift `COPY` assumes an **IAM role attached to the cluster** (`IAM_ROLE` clause) to read S3 — not your user's credentials. BigQuery load jobs run as the *caller's* identity (e.g., Composer's environment SA — see `WorkflowOrchestration`). Synapse `COPY INTO` authenticates via the workspace **Managed Identity** granted `Storage Blob Data Reader` on ADLS.
- **Orchestrator ➔ warehouse**: MWAA `RedshiftDataOperator` (Data API, async); Composer `BigQueryInsertJobOperator` (job poll, deferrable); ADF Copy activity / Script activity into Synapse. In all three, *the warehouse executes; the orchestrator waits*.
- **Spectrum/BigLake/serverless-pool catalogs**: Spectrum requires Glue Data Catalog external schemas; BigLake ties to Dataplex; Synapse serverless uses lake databases — the catalog is always the hinge between lake and warehouse (see `ObjectStorageDataLake`).
- **BI layer**: QuickSight ↔ Looker/Looker Studio (BI Engine caching) ↔ Power BI (DirectLake on Fabric is the headline feature).
- Network privacy: Redshift Enhanced VPC Routing; BigQuery via Private Google Access/PSC; Synapse via Private Endpoints + Managed VNet (see `Networking`).

## 6. Certification Checkpoints

- **DEA-C01**: DISTKEY/SORTKEY scenarios, COPY/UNLOAD, Spectrum vs loading, WLM/concurrency scaling, Serverless RPUs, Data API, datashares, streaming ingestion MVs.
- **GCP PDE**: partitioning vs clustering, slots & reservations, Storage Write API vs legacy streaming, authorized views, BQML model choice, external/BigLake tables, cost-optimization scenarios.
- **AZ-900**: Synapse = analytics service; Fabric awareness; difference from operational SQL Database.
- **CKA**: StatefulSet + PVC mechanics if you ever run ClickHouse/Trino in-cluster.
