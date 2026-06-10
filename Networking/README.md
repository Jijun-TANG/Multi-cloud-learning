# Networking — VPC · VPC Network · VNet · CNI/Service/Ingress

> Certifications served: **AZ-900** (VNet basics), **CKA** (Services, Ingress, NetworkPolicy, DNS — heavily tested), **DEA-C01/PDE** (private connectivity to data services).

## 1. Concept-First Blueprint

All cloud networking is **Software-Defined Networking**: an address space you carve into subnets, a routing layer, a firewall layer, a NAT/egress layer, and an L4/L7 entry layer.

```text
Address space ─► Subnets ─► Routes ─► Firewall ─► NAT (egress) ─► Load Balancer (ingress) ─► DNS
```

## 2. Rosetta Stone

| Concept | AWS | GCP | Azure | Kubernetes |
| --- | --- | --- | --- | --- |
| Private network | VPC (regional) | VPC Network (**global**) | VNet (regional) | Cluster network via **CNI** |
| Subnet scope | One AZ | One region | One region (spans zones) | Pod CIDR per node |
| Firewall (stateful) | Security Group (per-ENI) | VPC Firewall Rules (network-wide, tag-targeted) | Network Security Group (subnet or NIC) | **NetworkPolicy** (needs CNI support: Calico/Cilium) |
| Firewall (stateless) | Network ACL (subnet) | — | — | — |
| Routes | Route Tables | VPC Routes | Route Tables (UDR) | kube-proxy iptables/IPVS, CNI routes |
| Egress NAT | NAT Gateway | Cloud NAT | NAT Gateway | Node SNAT (default) |
| L7 entry | Application Load Balancer | Global External Application LB | Application Gateway | **Ingress** / Gateway API |
| L4 entry | Network Load Balancer | Network/Passthrough LB | Azure Load Balancer | Service `type: LoadBalancer` |
| Internal discovery | Route 53 private zones, Cloud Map | Cloud DNS private zones | Azure Private DNS | **CoreDNS** (`svc.ns.svc.cluster.local`) |
| Net-to-net link | VPC Peering, Transit Gateway | VPC Peering, NCC | VNet Peering, vWAN | Cluster mesh (Cilium/Istio) |
| Private path to PaaS | VPC Endpoints / PrivateLink | Private Service Connect / Private Google Access | Private Endpoint / Service Endpoints | — |
| Hybrid link | Direct Connect / Site-to-Site VPN | Cloud Interconnect / Cloud VPN | ExpressRoute / VPN Gateway | — |

## 3. Per-Provider Deep Dive

### AWS VPC
- **Regional VPC, zonal subnets.** "Public subnet" = its route table has `0.0.0.0/0 ➔ Internet Gateway`. Private subnets egress via NAT Gateway (which itself lives in a public subnet — exam favorite).
- Security Groups are **stateful and reference-able** (allow "from sg-xxxx") — this is the idiomatic AWS micro-segmentation. NACLs are stateless and rarely the right answer except for explicit subnet-level denies.
- **Gateway endpoints (S3, DynamoDB) are free**; Interface endpoints (PrivateLink) bill per-hour + per-GB. DEA-C01 loves "keep Glue/EMR ➔ S3 traffic off the internet" ➔ Gateway VPC Endpoint.

### GCP VPC Network
- **Global VPC** is GCP's superpower: one VPC, subnets in many regions, no peering needed between regions. Firewall rules target instances by **network tags or service accounts**.
- **Private Google Access**: lets VM/Dataproc/Composer nodes without external IPs reach Google APIs (BigQuery, GCS). PDE expects this for locked-down data clusters.
- Shared VPC: host project owns the network; service projects attach workloads — GCP's answer to multi-account networking.

### Azure VNet
- Subnets span zones (regional), so "zonal subnet design" doesn't exist here. NSGs attach to subnet *or* NIC; both apply, most restrictive wins.
- **Service Endpoints** (keeps PaaS public IP but routes over backbone, restricts to your VNet) vs **Private Endpoint** (injects a private IP for the PaaS resource into your subnet — the stronger, modern answer). AZ-900 only needs the concept; data certs need the distinction.
- Hub-and-spoke with Azure Firewall in the hub is the canonical enterprise topology.

### Kubernetes networking (CKA core)
Four invariants of the k8s network model:
1. Every Pod gets its own IP; no NAT pod-to-pod.
2. **Service (ClusterIP)** = stable virtual IP + DNS name load-balancing over Pod endpoints (EndpointSlices). kube-proxy programs iptables/IPVS to make the VIP real.
3. **NodePort** exposes a service on every node's IP at a high port (30000-32767); `LoadBalancer` builds on NodePort and asks the *cloud* for an NLB/ALB-equivalent.
4. **Ingress** is an L7 routing *spec*; it does nothing without an **Ingress Controller** (nginx, ALB controller, etc.). Gateway API is the successor.

- DNS: `my-svc.my-ns.svc.cluster.local` via CoreDNS. Debugging DNS (`kubectl exec ... -- nslookup`) is a recurring CKA task.
- **NetworkPolicy is default-allow until any policy selects a pod, then default-deny for that direction** — the single most misunderstood k8s networking fact.

## 4. Differences & Identical Links

**Identical:** subnetting/CIDR math, stateful firewall mental model, NAT-for-private-egress pattern, L4 vs L7 split, private DNS zones.

**Different:**
- Scope philosophy: GCP global VPC vs AWS/Azure regional; AWS zonal subnets vs GCP/Azure regional subnets. (Quoted directly in your notes: *GCP subnets are regional, AWS subnets live in one AZ*.)
- Firewall attachment point: AWS per-ENI groups, GCP network-wide tag rules, Azure subnet/NIC NSGs, k8s label-selector policies.
- k8s adds an entire **east-west** discovery layer (Services + CoreDNS) that clouds only approximate with Cloud Map / internal DNS.

## 5. Cross-Component Operating Notes

- **EKS + AWS VPC CNI**: pods get *real VPC IPs* from the subnet — beautiful for security-group-per-pod, but **exhausts subnet IPs fast**; plan large CIDRs or prefix delegation. GKE uses alias IP ranges (similar); AKS offers kubenet (NATed) vs Azure CNI (VPC-routable) — choose Azure CNI for enterprise.
- **ALB ➔ EKS**: the AWS Load Balancer Controller watches Ingress objects and *provisions an actual ALB*; annotations control target-type (`ip` = direct-to-pod). Same pattern: GKE Ingress ➔ Google Cloud LB; AKS AGIC ➔ Application Gateway.
- **Data-plane privacy patterns** (exam gold):
  - AWS: Glue connections + Gateway endpoint for S3, Interface endpoints for Kinesis/Secrets Manager from private subnets.
  - GCP: Dataproc/Composer with internal-IP-only nodes + Private Google Access to reach BigQuery/GCS.
  - Azure: Databricks VNet injection + Private Endpoints to ADLS Gen2.
- **Beanstalk/App Engine/App Service** all hide LB provisioning, but underneath they create ALB / Google LB / built-in front-end exactly as you would manually.

## 6. Certification Checkpoints

- **AZ-900**: VNet, NSG, peering, VPN Gateway vs ExpressRoute, Azure Firewall vs NSG, DNS basics.
- **CKA**: Service types, Ingress + controller, NetworkPolicy authoring, CoreDNS debugging, CNI role, kube-proxy modes.
- **DEA-C01**: VPC endpoints for S3/DynamoDB, Lambda in VPC (ENI behavior), Redshift enhanced VPC routing, MSK private connectivity.
- **GCP PDE**: Private Google Access, Shared VPC for data platforms, Dataflow worker networking.
