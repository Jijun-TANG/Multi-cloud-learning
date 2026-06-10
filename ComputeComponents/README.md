# Compute Components — EC2 · Compute Engine · Azure VMs · Pods

> Certifications served: **AZ-900** (core domain), **CKA** (Pods/Nodes), **DEA-C01 & GCP PDE** (compute that backs data services: EMR nodes, Dataproc workers, Redshift RA3 nodes, etc.)

## 1. Concept-First Blueprint

A compute unit is a slice of CPU + memory + ephemeral disk + network identity, scheduled onto physical hardware by a placement engine, and governed by a **reconciliation loop** that maintains a desired replica count.

```text
Image/Template ─► Placement Engine ─► Running Unit ─► Health Check ─► Replace/Scale
```

All four platforms implement exactly this pipeline. Only the unit size and the lifecycle manager differ.

## 2. Rosetta Stone

| Concept | AWS | GCP | Azure | Kubernetes |
| --- | --- | --- | --- | --- |
| Base unit | EC2 Instance | Compute Engine VM | Virtual Machine | **Pod** (1+ containers) |
| Machine image | AMI | Custom Image / Image Family | Managed Image / Azure Compute Gallery | Container Image (OCI) |
| Launch template | Launch Template | Instance Template | VM Scale Set Model | PodSpec (in Deployment) |
| Self-healing group | Auto Scaling Group (ASG) | Managed Instance Group (MIG) | VM Scale Set (VMSS) | ReplicaSet / Deployment |
| Startup script | User Data (cloud-init) | startup-script metadata | Custom Script Extension / cloud-init | `command`/`args`, initContainers |
| Instance metadata | IMDSv2 (`169.254.169.254`) | Metadata server (same IP) | IMDS (same IP) | Downward API |
| Spot/preemptible | Spot Instances | Spot VMs (formerly Preemptible) | Spot VMs | Node pools of spot VMs + taints |
| Placement constraint | Placement Groups, AZs | Zones, sole-tenant nodes | Availability Sets/Zones, Proximity Groups | nodeSelector, affinity, topologySpreadConstraints |
| Vertical sizing | Instance families (m5, c7g, r6i…) | Machine families (e2, n2, c3…) custom shapes | VM Series (B, D, E, F…) | resources.requests/limits |
| PaaS wrapper | **Elastic Beanstalk** | **App Engine** | **App Service** | Helm chart / Operator |

### Key sizing differences worth memorizing

- **GCP is the only provider with custom machine shapes** (arbitrary vCPU:RAM ratios). AWS and Azure force you into predefined sizes.
- **AWS bills per second (60s min) on Linux; Azure per second; GCP per second (60s min)** — effectively identical now, but sustained-use discounts are *automatic only on GCP*.
- Kubernetes Pods are *not* billed; the **Nodes** (which are EC2/GCE/Azure VMs underneath in EKS/GKE/AKS) are.

## 3. Per-Provider Deep Dive

### AWS EC2 — the un-opinionated construction kit
- **Lifecycle**: pending ➔ running ➔ stopping ➔ stopped ➔ terminated. *Stopped* instances don't bill compute but **still bill EBS volumes** (classic exam trap).
- **IMDSv2** requires a session token (PUT then GET) — mitigates SSRF credential theft. DEA-C01 expects you to know instance roles deliver temporary credentials via IMDS.
- **EBS vs Instance Store**: EBS is network-attached and persistent; instance store is physically attached NVMe, wiped on stop/terminate. Kafka/Spark shuffle-heavy workloads (EMR) often use instance store for throughput.
- Tenancy options: shared, dedicated instance, dedicated host (licensing/BYOL scenarios — AZ-900 analog is Azure Dedicated Host).

### GCP Compute Engine — automated and forgiving
- **Live migration**: GCE transparently moves running VMs off failing hardware *without reboot* — unique among the three. AWS/Azure require a stop/start or maintenance event.
- Regional MIGs spread instances across zones automatically; instance templates are immutable (you roll a new template version, like a new Launch Template version).
- Default service account is attached to every VM unless you specify otherwise — **a security anti-pattern**; PDE expects you to attach a minimal custom service account.

### Azure Virtual Machines — enterprise-integrated
- VM = required NIC + OS disk + optional data disks, all separate ARM resources in a **Resource Group**. Deleting a VM does *not* automatically delete its disks/NICs unless you opt in (AZ-900 favorite).
- **Availability Set** (fault/update domains within one DC) vs **Availability Zone** (separate DCs) vs **VMSS** (elastic fleet) — know the SLA ladder: single VM with Premium SSD 99.9% ➔ Availability Set 99.95% ➔ Zones 99.99%.
- Azure Hybrid Benefit: reuse on-prem Windows/SQL licenses — frequent AZ-900 cost question.

### Kubernetes Pods — the cloud-native atom (CKA core)
- A Pod is a shared network namespace + shared volumes around one or more containers. Containers in a pod talk over `localhost`.
- Pods are **mortal and never self-heal alone** — a bare Pod that dies stays dead. Deployments ➔ ReplicaSets give the reconciliation loop.
- Scheduling flow: `kubectl apply` ➔ API server validates ➔ object persisted in etcd ➔ scheduler binds Pod to Node ➔ kubelet pulls image and starts containers ➔ status written back.
- QoS classes (Guaranteed / Burstable / BestEffort) determine eviction order under node pressure — derived purely from requests/limits.
- Static Pods (managed directly by kubelet from `/etc/kubernetes/manifests/`) are how control-plane components themselves run in kubeadm clusters — classic CKA question.

## 4. Differences & Identical Links

**Genuinely identical everywhere:**
- The image ➔ template ➔ group ➔ health-check ➔ replace pipeline.
- Metadata service at `169.254.169.254` on all three clouds (k8s replaces it with the Downward API + projected service account tokens).
- cloud-init is the de facto startup standard on all three clouds' Linux VMs.

**Genuinely different:**
- **Granularity**: Pod (seconds to start, process-level) vs VM (tens of seconds to minutes, OS-level).
- **GCE live migration** vs AWS/Azure maintenance reboots.
- **Azure's decomposed resource model** (NIC, disk, IP as separate objects) vs AWS/GCP's more monolithic instance object.
- Default firewall stance: AWS Security Group denies all inbound by default; GCP default VPC ships permissive default rules (delete them!); Azure NSGs allow VNet-internal by default.

## 5. Cross-Component Operating Notes

### How Elastic Beanstalk initializes EC2 (the canonical PaaS-over-IaaS example)
1. You upload an app bundle (zip/war) or Docker config; Beanstalk stores it in **S3**.
2. Beanstalk synthesizes a **CloudFormation stack** — this is the part most people miss: every Beanstalk environment is just generated CloudFormation.
3. The stack creates a **Launch Template + Auto Scaling Group + (optionally) ALB + Security Groups**, using a Beanstalk **platform AMI** for your runtime (Python/Node/Docker…).
4. Each EC2 boots with Beanstalk **user data** that installs the host manager agent; the agent pulls your bundle from S3, runs `.ebextensions`/`.platform` hooks, wires the app to the reverse proxy (nginx), and reports health to the Beanstalk service.
5. An **instance profile (IAM role)** — typically `aws-elasticbeanstalk-ec2-role` — is attached so instances can read the S3 bundle and push logs to CloudWatch. A separate **service role** lets Beanstalk itself orchestrate.
   - **Gotcha**: you keep full SSH access to the EC2s (it's *not* abstracted away like App Engine), but any manual change is destroyed on the next deployment because the ASG re-provisions from the template. Treat instances as cattle; customize only via `.ebextensions`.

### GCP/Azure analogs
- **App Engine Flexible** similarly drives GCE: it builds a container, creates a regional MIG of VMs running the container, and manages versions/traffic-splitting. App Engine *Standard* instead runs on Google-internal sandboxes — no VMs visible to you at all.
- **Azure App Service** runs your code on a shared or dedicated **App Service Plan** (the plan is the VM fleet; the app is the tenant). Scale the *plan*, not the app — classic AZ-900 distinction.

### Compute ↔ other layers
- **EC2 ➔ IAM**: never store keys on a VM; attach an instance profile. The SDK credential chain checks IMDS automatically. Same pattern: GCE attached service account; Azure VM system-assigned Managed Identity.
- **VM ➔ data services**: EMR/Dataproc clusters are *just* EC2/GCE fleets with Hadoop/Spark bootstrapped — every VM concept (spot, instance roles, startup scripts) applies directly to data-engineering clusters.
- **Nodes ➔ Pods**: in EKS/GKE/AKS, cluster autoscaler manipulates the cloud's ASG/MIG/VMSS to add nodes when Pods are unschedulable. Two loops cooperating: HPA scales Pods, cluster autoscaler scales VMs.

## 6. Certification Checkpoints

- **AZ-900**: VM SLAs, Availability Sets vs Zones, VMSS, App Service vs VMs vs Containers vs Functions spectrum, Hybrid Benefit, pay-as-you-go vs reserved.
- **CKA**: Pod lifecycle, multi-container patterns (sidecar/init), requests/limits & QoS, static pods, taints/tolerations/affinity, draining nodes (`kubectl drain --ignore-daemonsets`).
- **DEA-C01**: EC2 as substrate for EMR/MSK/OpenSearch; spot for transient ETL; instance store vs EBS for shuffle; Graviton cost levers.
- **GCP PDE**: Dataproc on GCE (preemptible secondary workers), machine-type tuning for Dataflow workers.
