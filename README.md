# Cloud Native Architecture — Multi-Cloud Certification Knowledge Base

A parallel-learning documentation set covering **AWS, Google Cloud, Microsoft Azure, and Kubernetes** through the lens of *cloud-native architecture*. Instead of learning four ecosystems separately, every repository here teaches **one foundational concept once**, then maps the proprietary names, behaviors, and gotchas across all providers (the "Rosetta Stone" method).

## Target Certifications

| Certification | Code | Primary Repositories | Supporting Repositories |
| --- | --- | --- | --- |
| **AWS Certified Data Engineer – Associate** | DEA-C01 | `DataWarehouse`, `DataProcessing`, `StreamingMessaging`, `ObjectStorageDataLake`, `WorkflowOrchestration`, `NoSQLDatabases` | `IdentitySecurity`, `ServerlessFunctions`, `EventRouting`, `Observability` |
| **Google Cloud Professional Data Engineer** | PDE | `DataWarehouse`, `DataProcessing`, `StreamingMessaging`, `WorkflowOrchestration`, `NoSQLDatabases` | `ObjectStorageDataLake`, `IdentitySecurity`, `k8sServices` |
| **Microsoft Azure Fundamentals** | AZ-900 | `ComputeComponents`, `Networking`, `IdentitySecurity`, `SecretManagements` | All others (breadth-level awareness) |
| **Certified Kubernetes Administrator** | CKA | `k8sServices`, `Networking`, `SecretManagements` | `ComputeComponents`, `Observability`, `EventRouting` |

## Repository Map (The Master Rosetta Stone)

| Repository | Core Requirement | AWS | GCP | Azure | Kubernetes |
| --- | --- | --- | --- | --- | --- |
| `ComputeComponents` | Compute units | EC2 / Elastic Beanstalk | Compute Engine / App Engine | Virtual Machines / App Service | Pods (on Nodes) |
| `Networking` | Private networking & routing | VPC, ALB/NLB | VPC Network, Cloud Load Balancing | VNet, Application Gateway | CNI, Service, Ingress |
| `IdentitySecurity` | Identity & access | IAM, Organizations | IAM, Resource Hierarchy | Entra ID, RBAC | RBAC, ServiceAccounts |
| `SecretManagements` | Key/password vault | Secrets Manager, SSM Parameter Store, KMS | Secret Manager, Cloud KMS | Key Vault | Secrets (+ External Secrets Operator) |
| `k8sServices` | Managed Kubernetes | EKS | GKE | AKS | Upstream itself (CKA core) |
| `ServerlessFunctions` | Functions-as-a-Service | Lambda | Cloud Run functions | Azure Functions | Knative Serving |
| `WorkflowOrchestration` | Pipeline orchestration | MWAA, Step Functions | Cloud Composer, Workflows | Data Factory, Logic Apps | Apache Airflow, Argo Workflows |
| `DataWarehouse` | Serverless warehouse | Redshift | BigQuery | Synapse / Fabric | Trino / ClickHouse |
| `ObjectStorageDataLake` | Object storage & lake | S3, Lake Formation | Cloud Storage, BigLake | Blob Storage, ADLS Gen2 | PV/PVC, MinIO |
| `StreamingMessaging` | Event brokers & queues | Kinesis, MSK, SQS/SNS | Pub/Sub | Event Hubs, Service Bus | Apache Kafka (Strimzi) |
| `DataProcessing` | Unified batch/stream processing | Glue, EMR, Managed Flink | Dataflow, Dataproc | Databricks, Synapse Spark, Fabric | Spark on k8s, Flink Operator |
| `NoSQLDatabases` | Wide-column / key-value NoSQL | DynamoDB | Bigtable, Firestore | Cosmos DB | Cassandra (k8ssandra) |
| `EventRouting` | Serverless event bus / router | EventBridge | Eventarc | Event Grid | Knative Eventing, Argo Events |
| `Observability` | Monitoring, logging, tracing | CloudWatch, AMP, X-Ray | Cloud Observability, GMP | Azure Monitor, App Insights | Prometheus, Grafana, FluentBit |

## The Five Universal Cloud-Native Laws

Every document in this knowledge base maps back to these shared philosophies:

1. **Declarative State** — you declare *what* you want (Terraform/CloudFormation/ARM-Bicep/YAML manifests); the platform figures out *how*.
2. **Continuous Reconciliation** — a control loop perpetually compares desired vs. current state (Auto Scaling Groups, Managed Instance Groups, VM Scale Sets, kube-controller-manager).
3. **Software-Defined Networking** — physical cables abstracted to VPC/VNet/CNI with software firewalls (Security Groups, firewall rules, NSGs, NetworkPolicies).
4. **Identity as the Perimeter** — every API call is authenticated and authorized; machine identities (IAM Roles, Service Accounts, Managed Identities, k8s ServiceAccounts) replace hardcoded credentials.
5. **Event-Driven Decoupling** — Source ➔ Router ➔ Target. Producers never call consumers directly; routers (EventBridge/Eventarc/Event Grid/Knative) filter and fan out.

## How Each Repository Is Structured

Every repository contains a `README.md` with:

1. **Concept-first blueprint** — the universal pattern, provider-agnostic.
2. **Rosetta Stone table** — service-by-service mapping with key limits/quotas.
3. **Per-provider deep dive** — philosophy, architecture, exam-relevant details.
4. **Differences & identical links** — what is genuinely the same vs. where the platforms diverge.
5. **Cross-component operating notes** — how this layer touches other layers (e.g., how Elastic Beanstalk bootstraps EC2, how Cloud Composer authenticates to BigQuery, how AKS pods reach Key Vault).
6. **Certification checkpoints** — what each exam (DEA-C01, PDE, AZ-900, CKA) actually tests from this domain.

## Suggested Parallel Study Order

```text
Week 1-2  IdentitySecurity ─► Networking          (foundation for everything)
Week 3    ComputeComponents ─► ServerlessFunctions
Week 4-5  k8sServices (CKA-intensive)
Week 6    ObjectStorageDataLake ─► SecretManagements
Week 7-8  StreamingMessaging ─► DataProcessing
Week 9    DataWarehouse ─► NoSQLDatabases
Week 10   WorkflowOrchestration ─► EventRouting ─► Observability
```

Practice by **Feature Deployment**, not by provider: deploy the same capability on all platforms in the same sitting and note the deltas. That is the entire trick.
