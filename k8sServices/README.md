# Kubernetes Services — EKS · GKE · AKS · Upstream (CKA Core)

> Certifications served: **CKA** (this is the primary repo), **AZ-900** (AKS awareness), **DEA-C01/PDE** (Spark/Kafka/Airflow run on k8s; Composer literally *is* GKE).

## 1. Concept-First Blueprint

Kubernetes is a **declarative reconciliation engine for containers**: you submit desired state (YAML) to an API server; controllers loop forever closing the gap between desired and observed state, with all state persisted in **etcd**.

```text
kubectl ─► API Server ─► etcd (desired state)
                ▲              │
        controllers & scheduler watch ──► kubelet on Nodes ──► containers (CRI) ──► status back
```

## 2. Cluster Anatomy (memorize for CKA)

| Component | Runs on | Job | Failure symptom |
| --- | --- | --- | --- |
| kube-apiserver | Control plane | Sole gateway; authN ➔ authZ ➔ admission | `kubectl` hangs entirely |
| etcd | Control plane | Raft-consistent KV store of ALL state | Cluster frozen; reads may work, writes fail |
| kube-scheduler | Control plane | Assigns Pods ➔ Nodes (filter, then score) | Pods stuck `Pending` with no node assigned |
| kube-controller-manager | Control plane | Runs reconciliation loops (ReplicaSet, Node, Job…) | Deployments don't create pods, nodes never marked NotReady |
| kubelet | Every node | Realizes PodSpecs via CRI; reports status | Node `NotReady` |
| kube-proxy | Every node | Programs Service VIPs (iptables/IPVS) | Service IPs unreachable |
| CoreDNS | Workload pods | Cluster DNS | name resolution failures |

In **EKS/GKE/AKS the entire control plane (including etcd) is managed and invisible** — you cannot SSH to it; the cloud backs up etcd for you. You manage only worker nodes (and sometimes not even those: Autopilot/Fargate).

## 3. Managed-Offering Rosetta Stone

| Metric | EKS (AWS) | GKE (GCP) | AKS (Azure) |
| --- | --- | --- | --- |
| Control-plane price | ~$0.10/hr per cluster | ~$0.10/hr (free-tier credit for one zonal/Autopilot) | **Free** (paid tier only for uptime SLA) |
| Philosophy | Un-opinionated assembly kit | Most automated, most mature (Google ≈ Borg lineage) | Enterprise/dev-tooling integration |
| Nodeless option | Fargate profiles | **Autopilot** (whole-cluster mode) | Virtual Nodes (ACI) |
| Node fleet primitive | Managed Node Groups (ASGs) | Node Pools (MIGs) | Node Pools (VMSS) |
| Cluster identity ➔ cloud | IRSA / Pod Identity | Workload Identity | Workload Identity (OIDC) |
| Default CNI | AWS VPC CNI (pods get VPC IPs) | GKE dataplane (alias IPs; Dataplane V2 = Cilium) | kubenet or Azure CNI |
| Ingress controller | AWS Load Balancer Controller ➔ ALB | GKE Ingress ➔ Cloud LB | AGIC ➔ App Gateway, or managed nginx |
| Directory integration | IAM access entries / aws-auth | Google Groups for RBAC | **Entra ID-native RBAC** (its killer feature) |
| Upgrade experience | You drive (control plane then node groups) | Release channels, hands-free | Auto-upgrade channels |

## 4. CKA Domain Survival Map

1. **Workloads & Scheduling**: Deployments/rolling updates/rollbacks (`kubectl rollout undo`), DaemonSets, Jobs/CronJobs, requests/limits, taints+tolerations vs nodeAffinity (taints *repel*, affinity *attracts* — you usually need both to dedicate nodes).
2. **Storage**: PV/PVC/StorageClass, access modes (RWO/ROX/RWX), `volumeBindingMode: WaitForFirstConsumer`, reclaim policies. In cloud: EBS CSI (RWO), EFS CSI (RWX) / PD CSI, Filestore / Azure Disk, Azure Files.
3. **Services & Networking**: see `Networking` repo — Services, Ingress, NetworkPolicy, CoreDNS debugging, Gateway API.
4. **RBAC & Security**: see `IdentitySecurity` — Roles/Bindings, ServiceAccounts, CSRs for new users.
5. **Cluster lifecycle (kubeadm)**: `kubeadm init/join/upgrade`, **etcd backup/restore** (`etcdctl snapshot save/restore` — near-guaranteed exam task), cert renewal (`kubeadm certs renew`), node drain/cordon for maintenance.
6. **Troubleshooting** (≈30% of CKA): `kubectl describe` events first; `CrashLoopBackOff` (app or probe issue) vs `ImagePullBackOff` (registry/name/secret) vs `Pending` (resources/taints/PVC). Node issues: check kubelet (`systemctl status kubelet`, `journalctl -u kubelet`). Control-plane issues in kubeadm: inspect static-pod manifests in `/etc/kubernetes/manifests/`.

```bash
# etcd snapshot (CKA classic)
ETCDCTL_API=3 etcdctl snapshot save /backup/snap.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

## 5. Differences & Identical Links

**Identical:** `kubectl` and YAML are 100% portable across EKS/GKE/AKS — that is the whole point. Helm charts, operators, CRDs, HPA behavior: identical.

**Different — exactly four seams where the cloud leaks in:**
1. **Authentication** into the cluster (IAM ↔ aws-auth/access entries; gcloud credentials; Entra ID).
2. **CNI/IPAM** behavior and IP exhaustion characteristics.
3. **LoadBalancer/Ingress provisioning** (which cloud LB appears and which annotations drive it).
4. **Storage classes** (which CSI driver, which disk/file service, default class name).

When a workload "works on GKE but not EKS," the cause is almost always one of these four.

## 6. Cross-Component Operating Notes

- **Cluster Autoscaler ➔ cloud compute**: unschedulable pods make the autoscaler call EC2 ASG / GCE MIG / VMSS APIs to add nodes. Karpenter (AWS) skips ASGs and provisions instances directly — faster, bin-packing-aware.
- **Data engineering on k8s**: Spark-on-k8s operator, Strimzi (Kafka), Flink operator, Airflow's **KubernetesExecutor** all schedule pods per task/executor — which is why Composer (Airflow on GKE) tasks are literally GKE pods (see `WorkflowOrchestration`).
- **etcd ≠ CloudFormation** (from your notes): CloudFormation/Deployment Manager/ARM are *provisioning engines* that run and step away; etcd is the *live state database*. The cloud config analogs of etcd are SSM Parameter Store / AppConfig, runtime configs, Azure App Configuration — but in-cluster you mostly just use ConfigMaps/Secrets, which etcd backs.
- **Secrets & identity bridges**: IRSA / Workload Identity / AKS Workload Identity federate pod SAs to cloud IAM (detailed in `IdentitySecurity`); External Secrets / CSI driver pull vault material (detailed in `SecretManagements`).
- **Observability**: managed Prometheus (AMP/GMP/Azure Managed Prometheus) scrape your cluster so you don't operate Prometheus storage (see `Observability`).

## 7. Certification Checkpoints

- **CKA**: everything in section 4; practice under time pressure with `kubectl` aliases and `--dry-run=client -o yaml`.
- **AZ-900**: AKS = managed Kubernetes; containers vs VMs vs serverless positioning; ACI for simple containers.
- **DEA-C01**: EMR on EKS, MSK vs self-managed Kafka on k8s; container-image Lambda vs k8s decision.
- **GCP PDE**: Composer-on-GKE architecture, Dataproc on GKE, GKE Autopilot cost model.
