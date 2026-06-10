# Object Storage & Data Lake — S3 · Cloud Storage · Blob/ADLS Gen2 · PV/MinIO

> **New repository** — added because object storage is the gravitational center of both data-engineering certs (DEA-C01, GCP PDE) and a guaranteed AZ-900 topic. The original layout had no home for it.
>
> Certifications served: **DEA-C01** (S3 + Lake Formation — pervasive), **GCP PDE** (GCS + BigLake), **AZ-900** (storage accounts, tiers, redundancy), **CKA** (PV/PVC contrast).

## 1. Concept-First Blueprint

Object storage = a flat keyspace of immutable blobs behind an HTTP API, with **11-nines-class durability**, lifecycle-tiered economics, and event emission on change. A **data lake** = object storage + open file formats (Parquet/ORC/Avro) + a table format (Iceberg/Delta/Hudi) + a catalog + access governance.

```text
Bucket/Container ─► Objects (key = pseudo-path) ─► Storage classes/tiers ─► Lifecycle rules
        └─► Change events ─► (EventRouting) ─► pipelines
        └─► Table format + Catalog ─► (DataWarehouse / DataProcessing) query in place
```

## 2. Rosetta Stone

| Concept | AWS S3 | GCP Cloud Storage | Azure Blob / ADLS Gen2 | Kubernetes |
| --- | --- | --- | --- | --- |
| Container | Bucket (global name) | Bucket (global name) | Storage Account ➔ Container | PersistentVolume |
| Hot tier | S3 Standard | Standard | Hot | — |
| Warm tiers | Standard-IA, One Zone-IA | Nearline (30d) | Cool (30d), Cold (90d) | — |
| Archive | Glacier IR / Flexible / **Deep Archive** | Coldline (90d) / **Archive** (365d) | Archive (rehydration needed) | — |
| Auto-tiering | **Intelligent-Tiering** | Autoclass | Lifecycle management | — |
| Versioning | Bucket versioning | Object versioning | Blob versioning + snapshots | — |
| Redundancy choices | Region-spread by default; CRR for geo | Regional / dual-region / multi-region | **LRS / ZRS / GRS / RA-GRS** (the AZ-900 ladder) | replicas |
| HDFS-like semantics | — (flat only) | — (flat; FUSE optional) | **ADLS Gen2 hierarchical namespace** (real dirs + POSIX ACLs) | — |
| Access governance | IAM + bucket policy + **Lake Formation** | IAM (uniform bucket-level) + **BigLake** | RBAC data-plane roles + ACLs + SAS tokens | RBAC on PVCs |
| Events | S3 Notifications / EventBridge | Pub/Sub notifications / Eventarc | Event Grid | — |
| Self-hosted analog | — | — | — | **MinIO** (S3-compatible) |

## 3. Per-Provider Deep Dive

### AWS S3
- Consistency: **strong read-after-write** since 2020 (old eventual-consistency lore is obsolete).
- Security stack evaluated together: IAM identity policies + bucket policies + ACLs (legacy, disable) + **Block Public Access** + VPC endpoint policies. "Why can't Glue read this bucket?" = walk this stack.
- Encryption: SSE-S3 (default), SSE-KMS (auditable, `kms:Decrypt` needed — frequent failure), SSE-C, client-side.
- Performance: 3,500 PUT / 5,500 GET **per prefix per second** — shard hot prefixes; multipart upload >100MB; S3 Transfer Acceleration; S3 Select retired in favor of querying via Athena.
- **Lake Formation** adds database/table/column/row grants + LF-tags on top of the Glue Catalog — the DEA-C01 answer for fine-grained lake security; S3 Tables (managed Iceberg) is the new native table layer.

### GCP Cloud Storage
- One bucket type, four classes, **identical millisecond first-byte latency even for Archive** (unique — Azure Archive requires rehydration, Glacier Flexible/Deep require restore jobs).
- Uniform bucket-level access (recommended) disables object ACLs entirely — IAM-only, matching GCP philosophy.
- **BigLake** unifies GCS data under BigQuery-grade governance (row/column security over Parquet/Iceberg in GCS).
- gsutil/gcloud storage CLI parallel composite uploads for large files.

### Azure Blob & ADLS Gen2
- Hierarchy: **Storage Account** (the billable, named resource — also hosts Files/Queues/Tables) ➔ container ➔ blob. AZ-900 tests this nesting and the four services.
- **ADLS Gen2 = Blob + hierarchical namespace flag**: real directories (atomic renames — critical for Spark commit protocols) + POSIX ACLs. Analytics workloads should always enable it.
- Redundancy ladder (AZ-900 favorite): LRS (3 copies, 1 DC) ➔ ZRS (3 zones) ➔ GRS (secondary region) ➔ RA-GRS (readable secondary).
- Access: account keys (avoid) ➔ **SAS tokens** (scoped, expiring — unique to Azure) ➔ Entra ID RBAC (`Storage Blob Data Reader/Contributor` — note these *data-plane* roles differ from control-plane Contributor!).

### Kubernetes contrast
- PV/PVC is **block/file** storage for pods, not object storage. Pods consume S3/GCS/Blob via SDKs + workload identity, not mounts (CSI mounts exist but are niche). **MinIO** gives you an in-cluster S3 API for portable dev/test.

## 4. Differences & Identical Links

**Identical:** flat-namespace objects, 11-nines durability, versioning, lifecycle tiering, signed URLs (presigned / signed / SAS), event-on-change, "the lake is just objects + catalog."

**Different:**
- **Namespace**: only ADLS Gen2 has a true hierarchical namespace; S3/GCS fake folders with delimiters (rename = copy+delete ➔ Spark output committers matter).
- **Archive latency**: GCS Archive is instant-read; Glacier Deep Archive and Azure Archive are not.
- **Account model**: Azure interposes the Storage Account; AWS/GCP buckets are top-level with globally unique names.
- **Delegated access**: Azure SAS tokens have no exact AWS/GCP twin (presigned URLs are per-object; SAS can scope whole containers with start/expiry/IP).

## 5. Cross-Component Operating Notes

- **The arrival-trigger pattern** (memorize all three): S3 PUT ➔ EventBridge ➔ Lambda/Step Functions/Glue; GCS finalize ➔ Eventarc (Pub/Sub) ➔ Cloud Run function ➔ Dataflow/Composer; Blob created ➔ Event Grid ➔ Function/ADF event trigger.
- **Engine ➔ lake auth**: EMR/Glue use job/instance roles to S3 (plus Lake Formation grants if enabled); Dataproc/Dataflow workers use their service account to GCS; Databricks on Azure reaches ADLS via Unity Catalog managed identities or service principals + `abfss://` URIs.
- **Warehouse ↔ lake**: Spectrum external schemas (Glue Catalog), BigQuery external/BigLake tables, Synapse serverless `OPENROWSET` — covered in `DataWarehouse`.
- **File hygiene for every exam**: Parquet > CSV/JSON for analytics; compress (Snappy); target 128MB–1GB files (small-files problem); partition paths `dt=YYYY-MM-DD/`; table formats (Iceberg/Delta/Hudi) add ACID, time travel, compaction.
- Private access paths: S3 Gateway VPC Endpoint (free), Private Google Access, ADLS Private Endpoints (see `Networking`).

## 6. Certification Checkpoints

- **DEA-C01**: storage classes & lifecycle math, Intelligent-Tiering, multipart/prefix performance, SSE-KMS pitfalls, Lake Formation grants, S3 event ➔ pipeline triggers.
- **GCP PDE**: class transitions, Autoclass, dual/multi-region for DR, BigLake, GCS ➔ BigQuery load/external decision.
- **AZ-900**: storage account types, four services, tiers, redundancy ladder, SAS vs keys vs RBAC.
- **CKA**: PV/PVC/StorageClass, access modes, reclaim policy — and knowing object storage is *not* that.
