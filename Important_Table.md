Here is the updated Multi-Cloud & Kubernetes Master Rosetta Stone table. A new row has been added for the **Serverless Event Bus / Event Router** layer, detailing how **AWS EventBridge** behaves along with its exact architectural equivalents across GCP, Azure, and Kubernetes.

---

## The Master Multi-Cloud & Kubernetes Rosetta Stone

| Core Requirement | Amazon Web Services (AWS) | Google Cloud (GCP) | Microsoft Azure | Kubernetes (k8s) Standard |
| --- | --- | --- | --- | --- |
| **Compute Units** | EC2 Instances | Compute Engine VMs | Virtual Machines | **Pods** (on Nodes) |
| **Private Networking** | VPC | VPC Network | Virtual Network (VNet) | **Cluster Network (CNI)** |
| **Traffic Router** | Application Load Balancer | Cloud Load Balancing | Azure Application Gateway | **Ingress Controller** |
| **Internal Routing** | Route Tables | VPC Routes | Route Tables | **Service** (ClusterIP) |
| **Identity & Security** | IAM | IAM | Entra ID | **RBAC** |
| **Key/Password Vault** | Secrets Manager | Secret Manager | Key Vault | **Secrets** |
| **Managed k8s Service** | **EKS** | **GKE** | **AKS** | *Upstream itself* |
| **Workflow Orchestration** | **MWAA** (Managed Airflow) | **Cloud Composer** | **Azure Data Factory** | **Apache Airflow** |
| **Unified Processing** | **AWS Glue** / **Managed Flink** | **Dataflow** (Apache Beam) | **Azure Databricks** | **Apache Spark** / **Flink** |
| **Serverless Warehouse** | **Amazon Redshift** | **BigQuery** | **Azure Synapse** / **Fabric** | **Trino** / **ClickHouse** |
| **Wide-Column NoSQL** | **Amazon DynamoDB** | **Bigtable** | **Azure Cosmos DB** | **Apache Cassandra** |
| **Real-time Event Broker** | **Amazon Kinesis** / **MSK** | Cloud Pub/Sub | Azure Event Hubs | **Apache Kafka** |
| **Unified Observability** | **Amazon CloudWatch** | **Google Cloud Observability** | **Azure Monitor** | *No built-in native suite* |
| **Metrics Engine** | Managed Prometheus (AMP) | Managed Prometheus (GMP) | Azure Managed Prometheus | **Prometheus** |
| **Log Shippers / Agents** | CloudWatch Agent / FireLens | Ops Agent (Fluent Bit) | Azure Monitor Agent (AMA) | **FluentD** / **Fluent Bit** |
| **Visualization Layer** | Amazon Managed Grafana | Google Cloud Dashboards | Azure Managed Grafana | **Grafana** |
| **Serverless Event Bus / Event Router** | **Amazon EventBridge** <br>

<br>

<br>*Receives events and can immediately trigger targets such as:*<br>

<br>• AWS Lambda functions<br>

<br>• Amazon SNS notifications<br>

<br>• Amazon SQS queues<br>

<br>• AWS Step Functions workflows | **Eventarc** <br>

<br>

<br>*Receives events and can immediately trigger targets such as:*<br>

<br>• Cloud Run functions<br>

<br>• Cloud Pub/Sub topics<br>

<br>• Cloud Workflows orchestrations<br>

<br>• GKE services | **Azure Event Grid** <br>

<br>

<br>*Receives events and can immediately trigger targets such as:*<br>

<br>• Azure Functions<br>

<br>• Azure Service Bus / Event Hubs<br>

<br>• Azure Storage queues<br>

<br>• Logic Apps workflows | **Knative Eventing** / **Argo Events** <br>

<br>

<br>*Receives events and can immediately trigger targets such as:*<br>

<br>• Native Cluster Pods / Services<br>

<br>• Autoscaling workloads (KEDA)<br>

<br>• Argo Workflows pipelines<br>

<br>• Kafka Event Broker |

---

## 💡 The Parallel Concept: The "Event-Driven Router" Pattern

When learning these in parallel, look at how the **Source ➔ Router ➔ Target** pattern functions exactly the same way across all three clouds and Kubernetes:

1. **The Source:** A state change occurs (e.g., an image file is uploaded, a database row is updated, or an external system sends a webhook).
2. **The Router (EventBridge / Eventarc / Event Grid / Knative):** Evaluates the incoming event against a rule or filter that you created (e.g., *"Is this message an error?"* or *"Is the file type a .pdf?"*).
3. **The Target (The Handlers):** If the event matches the rule, the router passes the event payload directly into a serverless compute service to run code, onto a messaging queue to buffer it, or into a workflow pipeline to begin a long-running corporate orchestration.

By mapping the routing architecture this way, you can build identical, decoupled, event-driven applications on any infrastructure you choice.