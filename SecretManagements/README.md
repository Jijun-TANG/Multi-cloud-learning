# Secret Management — Secrets Manager · Secret Manager · Key Vault · k8s Secrets

> Certifications served: **AZ-900** (Key Vault), **CKA** (Secrets/ConfigMaps — guaranteed exam tasks), **DEA-C01/PDE** (DB credentials for pipelines, encryption keys for lakes).

## 1. Concept-First Blueprint

Two related but distinct stores that beginners conflate:

```text
SECRETS  = small sensitive strings (passwords, API keys, tokens) — read at runtime, versioned, rotated
KEYS     = cryptographic material (KMS) — never leaves the HSM; you send data/requests TO the key
CONFIG   = non-sensitive runtime parameters — feature flags, endpoints (the etcd-like layer)
```

## 2. Rosetta Stone

| Concept | AWS | GCP | Azure | Kubernetes |
| --- | --- | --- | --- | --- |
| Secret store | **Secrets Manager** | **Secret Manager** | **Key Vault** (secrets) | **Secret** object |
| Cheap/simple param store | SSM Parameter Store | — (use Secret Manager) | App Configuration | ConfigMap |
| Crypto keys (KMS) | AWS KMS | Cloud KMS | Key Vault (keys) / Managed HSM | — (use cloud KMS for envelope encryption of etcd) |
| Native rotation | Built-in (Lambda-driven, RDS/Redshift turnkey) | Rotation reminders + Pub/Sub triggers (you implement) | Key Vault rotation policies (keys); Functions for secrets | None native |
| Versioning | Staging labels (AWSCURRENT/AWSPREVIOUS) | Numbered versions + aliases (`latest`) | Version GUIDs | resourceVersion only |
| Access control | IAM + resource policy | IAM per-secret bindings | **Access Policies (legacy) or Azure RBAC** + network rules | RBAC on Secret objects |
| Audit | CloudTrail | Cloud Audit Logs | Key Vault diagnostics ➔ Monitor | API server audit logs |
| Price model | ~$0.40/secret/month + API calls | Per active secret-version + access ops | Per operation | Free (etcd) |
| etcd-equivalent config layer | SSM Parameter Store / AppConfig | Runtime config patterns | App Configuration | **etcd itself** (via ConfigMaps) |

## 3. Per-Provider Deep Dive

### AWS
- **Secrets Manager vs Parameter Store** — recurring exam decision:
  - Secrets Manager: paid, native rotation, cross-account resource policies, RDS/Redshift/DocumentDB turnkey rotation.
  - Parameter Store (SecureString): free tier, no native rotation, great for config + modest secrets.
- **KMS envelope encryption** underpins everything: S3 SSE-KMS, EBS, Redshift, Glue catalog encryption. Know `kms:Decrypt` must be granted *in addition to* `secretsmanager:GetSecretValue` when a CMK is used.
- Glue/Lambda/MWAA all fetch DB credentials from Secrets Manager by ARN — the idiomatic DEA-C01 answer for "no hardcoded credentials."

### GCP
- Secret Manager is deliberately minimal: secret ➔ versions ➔ `latest` alias. Grant `roles/secretmanager.secretAccessor` per secret to a service account.
- Rotation is **notification-based**: a schedule publishes to Pub/Sub, a Cloud Run function performs the actual rotation — fits GCP's composable philosophy.
- Cloud KMS is separate; CMEK (customer-managed encryption keys) on BigQuery/GCS/Dataflow is the PDE-relevant feature.

### Azure Key Vault — three stores in one
- Holds **secrets, keys, and certificates** in a single resource — the only provider that bundles them.
- Two permission models: legacy **Access Policies** (vault-wide grants) vs **Azure RBAC** (recommended, scoped, PIM-compatible). Know both exist for AZ-900.
- Soft-delete + purge protection are on by default — deleted vault objects are recoverable (and purge-protected vaults can't be force-emptied; this blocks "delete & recreate" automation, a real ops gotcha).
- Pricing tiers: Standard (software keys) vs Premium (HSM-backed).

### Kubernetes Secrets (CKA core)
- **Base64 is encoding, not encryption.** Anyone with `get secrets` RBAC or etcd access reads everything. Mitigations: RBAC scoping, etcd encryption-at-rest (`EncryptionConfiguration`, often KMS-backed via the cloud), and short-lived projected tokens.
- Consumption modes: env vars (`secretKeyRef`), volume mounts (auto-refresh on update; env vars do **not** refresh), `imagePullSecrets`.
- Types: `Opaque`, `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson`, `bootstrap.kubernetes.io/token`.
- `kubectl create secret generic db --from-literal=pw=x` and mounting it are bread-and-butter CKA tasks.

## 4. Differences & Identical Links

**Identical:** version-and-alias model, IAM-gated access, audit logging, "reference the store at runtime, never bake secrets into images/templates."

**Different:**
- Bundling: Azure unifies secrets+keys+certs; AWS/GCP split secret store from KMS.
- Rotation: AWS turnkey ➔ Azure semi-managed ➔ GCP build-it-yourself ➔ k8s none.
- k8s Secrets are a *cache near the workload*, not a system of record — production posture is syncing from a cloud vault.

## 5. Cross-Component Operating Notes

- **External Secrets Operator / CSI Secrets Store driver** are the bridge patterns: pods authenticate via IRSA / Workload Identity / AKS Workload Identity (see `IdentitySecurity`) and pull from Secrets Manager / Secret Manager / Key Vault, materializing native k8s Secrets or mounted files. This chains *two* repos: workload identity + secret store.
- **MWAA/Composer/ADF credentials**:
  - MWAA reads Airflow *connections* from Secrets Manager via a configurable secrets backend (`AIRFLOW__SECRETS__BACKEND`).
  - Cloud Composer reads connections/variables from GCP Secret Manager the same way.
  - ADF uses **Key Vault-linked services**: the pipeline references a Key Vault secret; ADF's Managed Identity needs `get` on secrets.
- **Beanstalk/App Service config**: environment properties are stored in the platform (not a vault) — for real secrets, reference Secrets Manager / Key Vault from app code or use App Service Key Vault references (`@Microsoft.KeyVault(...)`).
- **etcd encryption on managed k8s**: EKS envelope-encrypts secrets with a KMS CMK you supply; GKE uses Cloud KMS application-layer encryption; AKS supports KMS etcd encryption — the clouds literally plug their KMS into the k8s `EncryptionConfiguration` mechanism.
- Network hardening: Key Vault firewall + Private Endpoint; Secrets Manager interface VPC endpoint; Secret Manager via Private Google Access — same pattern, three names (see `Networking`).

## 6. Certification Checkpoints

- **AZ-900**: what Key Vault stores, why (central rotation, audit), Managed Identity + Key Vault as the no-credentials pattern.
- **CKA**: create/mount Secrets & ConfigMaps, env-vs-volume refresh behavior, etcd encryption awareness.
- **DEA-C01**: Secrets Manager vs Parameter Store choice, rotation for Redshift/RDS, KMS grants for encrypted pipelines.
- **GCP PDE**: CMEK on BigQuery/GCS, Secret Manager for pipeline credentials, service-account-scoped accessor roles.
