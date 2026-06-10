# Identity & Security — AWS IAM · GCP IAM · Entra ID · k8s RBAC

> Certifications served: **AZ-900** (Entra ID, RBAC, Zero Trust, Defender — heavily tested), **CKA** (RBAC, ServiceAccounts, certificates), **DEA-C01/PDE** (least-privilege data access, lake permissions).

## 1. Concept-First Blueprint

Every platform answers the same equation:

```text
WHO (principal) + CAN DO WHAT (permissions) + ON WHICH RESOURCE (scope) [+ UNDER WHICH CONDITIONS]
```

The platforms differ in **where identities live**, **how permissions are expressed**, and **how scope inherits**.

## 2. Rosetta Stone

| Concept | AWS | GCP | Azure | Kubernetes |
| --- | --- | --- | --- | --- |
| Top boundary | AWS Organizations | Organization | Management Groups (under an Entra **Tenant**) | Cluster |
| Logical box | Account | Project | Subscription ➔ Resource Group | Namespace |
| Human identity | IAM User / Identity Center user | Google Account (Workspace / Cloud Identity) | Entra ID User | External (certs, OIDC) — *no User object exists* |
| Machine identity | IAM Role | Service Account | Managed Identity / Service Principal | ServiceAccount |
| Permission bundle | IAM Policy (JSON document) | IAM Role (curated permission list) | RBAC Role definition | Role / ClusterRole |
| Attachment action | Attach policy | IAM **binding** (member ↔ role ↔ resource) | Role **assignment** (principal ↔ role ↔ scope) | RoleBinding / ClusterRoleBinding |
| Guardrails | SCPs (Organizations) | Organization Policy | Azure Policy | Admission controllers / OPA Gatekeeper / Kyverno |
| Cross-boundary access | `sts:AssumeRole` | Service-account impersonation | Cross-tenant guest (B2B) | — |
| Federation | IAM OIDC/SAML providers, Identity Center | Workload Identity Federation | Entra ID federation, Workload identity federation | OIDC API-server flags |

## 3. The Three-and-a-Half Philosophies

### AWS — policy-document centric, historically flat
- Permissions are **JSON policy documents** (`Effect/Action/Resource/Condition`). Evaluation: **explicit Deny > explicit Allow > implicit deny**. Identity-based + resource-based policies are unioned (within one account).
- Accounts are isolation islands; cross-account = resource policy *or* AssumeRole. **Roles deliver temporary STS credentials** — the universal AWS machine-auth mechanism (EC2 instance profile, Lambda execution role, Glue job role…).
- Data-engineering specifics: **Lake Formation** layers table/column/row-level grants on top of IAM for S3 data lakes; Redshift uses both IAM (`GetClusterCredentials`) and internal DB grants.

### GCP — curated roles, inherent hierarchy
- **Organization ➔ Folders ➔ Projects ➔ Resources**; policy bindings inherit *downward* and are **additive only** (no deny in classic IAM; Deny Policies exist but are a separate, newer construct).
- No native user database — identities always come from Google (Workspace/Cloud Identity). You bind `roles/bigquery.dataViewer` to `user:ana@corp.com` on a project or dataset.
- Roles: **Basic** (Owner/Editor/Viewer — avoid), **Predefined** (use these), **Custom**.
- **Service accounts are both identity and resource**: a human can hold `roles/iam.serviceAccountUser` to act as one. Watch for privilege-escalation via `serviceAccountTokenCreator`.

### Azure — two interlocking systems (the part everyone confuses)
1. **Entra ID** (the directory): users, groups, app registrations, service principals, Conditional Access, MFA. Lives at the *tenant* level, outside any subscription.
2. **Azure RBAC** (resource authorization): role definitions + assignments at Management Group / Subscription / Resource Group / Resource scope, inheriting downward.
- AZ-900 trap: *Entra ID roles* (e.g., Global Administrator) govern the directory; *Azure RBAC roles* (e.g., Contributor) govern resources. A Global Admin has **no resource access** until they elevate.
- **Managed Identity** = a service principal whose credentials Azure rotates invisibly. System-assigned (lifecycle tied to resource) vs user-assigned (shareable).
- Zero-trust toolkit tested on AZ-900: Conditional Access, MFA, PIM (just-in-time roles), Defender for Cloud, Azure Policy.

### Kubernetes RBAC (CKA core)
- Subjects: `User` (from certs/OIDC — not stored in the cluster), `Group`, `ServiceAccount` (namespaced object).
- `Role` (namespaced) / `ClusterRole` (cluster-wide) define verbs (`get,list,watch,create,update,patch,delete`) on resources/apiGroups; bindings attach them. **RBAC is allow-only; there is no deny rule.**
- Check access: `kubectl auth can-i get pods --as system:serviceaccount:dev:ci-bot -n dev`.
- Pod ➔ API-server auth: projected ServiceAccount token automounted at `/var/run/secrets/kubernetes.io/serviceaccount/`.

## 4. Differences & Identical Links

**Identical:** least privilege, role-based grouping of permissions, scope inheritance (GCP folders ≍ Azure management groups ≍ k8s namespace vs cluster scope), machine identities issuing short-lived credentials.

**Different:**
- **Deny semantics**: AWS has first-class explicit Deny; GCP classic IAM and k8s RBAC are additive-only; Azure has deny assignments only via locks/Blueprints edge cases.
- **Where users live**: inside the account (legacy AWS), outside in Google (GCP), in a global directory predating the cloud (Entra), or nowhere (k8s).
- **Expression format**: raw JSON policies (AWS) vs curated role catalogs (GCP/Azure) vs verbs-on-resources YAML (k8s).

## 5. Cross-Component Operating Notes (workload ➔ cloud identity)

The most important modern pattern: **eliminate static keys by federating pod/VM identity to cloud IAM**.

| Pattern | Mechanism |
| --- | --- |
| **EKS: IRSA / Pod Identity** | k8s ServiceAccount annotated with an IAM role ARN; API server issues an OIDC token; AWS STS exchanges it (`AssumeRoleWithWebIdentity`) for role credentials. A Spark-on-EKS pod reads S3 with zero stored keys. |
| **GKE: Workload Identity** | k8s SA bound to a GCP service account (`iam.workloadIdentityUser`); metadata server inside the pod returns GCP tokens. This is *exactly* how a Composer/GKE pod calls BigQuery. |
| **AKS: Workload Identity** | k8s SA federated to an Entra application via OIDC issuer; pod gets Entra tokens to reach Key Vault / ADLS. |
| **VMs** | EC2 instance profile ↔ GCE attached service account ↔ Azure Managed Identity — identical concept, different name. |

Other cross-layer notes:
- **Beanstalk** needs *two* IAM roles: the instance profile (app's permissions) and the service role (Beanstalk's own orchestration rights). Most managed services replicate this split — e.g., Glue job role vs the Glue service itself, Composer environment SA vs the Composer service agent.
- **Lake permissions stack**: S3 bucket policy ➔ Lake Formation grants ➔ Glue Catalog resource policy can all gate the same table — know the evaluation interplay for DEA-C01. GCP analog: dataset-level IAM + BigQuery row/column policies. Azure analog: ADLS POSIX ACLs + RBAC data-plane roles (`Storage Blob Data Reader`).
- Hands-on parallel drill (from your notes): grant a dummy user read on one bucket in all three clouds — JSON policy (AWS) vs role binding (GCP) vs scoped role assignment (Azure).

## 6. Certification Checkpoints

- **AZ-900**: Entra ID vs Azure RBAC distinction, MFA/Conditional Access, Zero Trust, defense-in-depth, Azure Policy vs RBAC ("what you *can* do" vs "what you *may* deploy").
- **CKA**: write Role/RoleBinding YAML fast, ServiceAccounts, `kubectl auth can-i`, certificate signing requests for new users, kubeconfig anatomy.
- **DEA-C01**: Lake Formation, cross-account S3/Glue access, Redshift IAM auth, Secrets-vs-roles decisions.
- **GCP PDE**: BigQuery dataset/row/column-level security, service-account impersonation, authorized views/datasets.
