---
title: "K8s Default Resources"
created: "2026-05-15"
updated: "2026-05-15"
version: 1.0.0
type: reference
owner: Kenan Lalic
lifecycle: production
tags: [kubernetes, argocd, gitops, envoy, cert-manager, sealed-secrets, keycloak, cnpg, postgresql, oci, arm64]
---

# K8s Default Resources — Platform Layer

ArgoCD-managed platform layer for the OCI Free Tier Kubernetes cluster. Sealed Secrets, Envoy Gateway, cert-manager, CloudNativePG, Keycloak, and application workloads — all managed via GitOps using the app-of-apps pattern.

> 📍 **Type:** Service Reference<br>
> 👤 **Owner:** Kenan Lalic<br>
> 🎯 **Outcome:** Understand the K8s platform layer, its bootstrap sequence, and how services are onboarded

> [!NOTE]
> ARM architecture only (`aarch64`). See [ARM64 constraint](../40-kubernetes/40-oci-infrastructure.md#arm64-only) for context.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Design Decisions](#design-decisions)
- [Dependencies](#dependencies)
- [Configuration](#configuration)
- [Bootstrap Sequence](#bootstrap-sequence)
- [Database](#database)
- [Adding a Django Service](#adding-a-django-service)
- [Dev vs Prod Differences](#dev-vs-prod-differences)
- [Runbooks](#runbooks)
- [ADRs](#adrs)
- [References](#references)

---

## Overview

The platform layer is entirely independent from the infrastructure layer. Bring your own GitOps tool, gateway, auth provider, or monitoring stack — nothing in Terraform needs to change.

### What Gets Deployed

| Layer | Service | Status | Purpose |
|---|---|---|---|
| **Cluster** | Sealed Secrets | ✅ Production | GitOps-safe secret management |
| **Cluster** | ArgoCD | ✅ Production | GitOps controller |
| **Cluster** | Envoy Gateway | ✅ Production | Front door — K8s Gateway API |
| **Cluster** | cert-manager | ✅ Production | TLS lifecycle, Let's Encrypt |
| **Cluster** | CloudNativePG (CNPG) | ⚠️ Production (no backup) | PostgreSQL operator — `platform-db` cluster, 10 Gi PVC |
| **Thirdparty** | Keycloak | 🔜 On first service onboard | OIDC/SSO for all services |
| **Thirdparty** | Prometheus + Grafana | 🚧 Not yet deployed | Metrics and alerting |
| **Owned** | Django Services | Add per service | Generated from Copier template |

### Repository Structure

```
k8s-default-resources/
├── argocd-applications/
│   ├── app-of-apps/            ← root ArgoCD Application (bootstrapped once)
│   ├── entrypoints/
│   │   ├── cluster.yaml        ← platform services
│   │   ├── owned.yaml          ← your workloads (Django services)
│   │   └── thirdparty.yaml     ← Keycloak, databases
│   ├── cluster/
│   │   └── cloudnative-pg.yaml ← CNPG ArgoCD Application
│   ├── owned/
│   └── thirdparty/
├── cluster/
│   ├── argocd/
│   ├── cert-manager/
│   ├── sealed-secrets/
│   ├── cloudnative-pg/
│   │   ├── kustomization.yaml           ← operator + cluster resources
│   │   ├── install-v1.28.0-download.yaml
│   │   └── cluster/
│   │       ├── kustomization.yaml
│   │       ├── cluster.yaml             ← platform-db Cluster CR
│   │       └── keycloak-database.yaml   ← keycloak Database CR
│   └── gateways/
│       ├── gatewayclass.yaml
│       ├── gateway.yaml
│       └── envoy-gateway/
├── credentials/
│   └── github.ssh/
└── runbooks/
    ├── argocd-cleanup.md
    └── cert-manager-bootstrap.md
```

### Architecture

```
Internet
    ↓
OCI Flexible Load Balancer  (1 free — auto-provisioned when Gateway type=LoadBalancer)
    ↓  HTTPS :443 (TLS terminated — cert-manager + Let's Encrypt)
Envoy Gateway v1.7.0        (K8s Gateway API native)
    ↓  plain HTTP internally
ArgoCD server v3.3.2        (server.insecure=true — TLS at Gateway boundary)
    ↓
Kubernetes cluster          (2 × ARM64 nodes, OCI Always Free)
    ↓
App-of-Apps
    ├── cluster.apps    → Sealed Secrets, ArgoCD, Envoy Gateway, cert-manager, CNPG
    ├── thirdparty.apps → Keycloak
    └── owned.apps      → Django services
```

### Auth — Current and Planned

| Stage | Auth |
|---|---|
| Now | ArgoCD native admin login |
| When first Django service onboards | Keycloak OIDC — ArgoCD configured as OIDC client |

Keycloak is not part of the bootstrap. It is added as an ArgoCD-managed Application under `argocd-applications/thirdparty/` when the first Django service is onboarded. See [Keycloak OIDC](../20-base-infrastructure/22-keycloak-oidc.md) for the full identity configuration.

---

## Architecture

### App-of-Apps Pattern

The three entrypoints map directly to ownership boundaries:

```
argocd-applications/
├── app-of-apps/
│   └── applications.yaml     ← creates the three entrypoint Applications
├── entrypoints/
│   ├── cluster.yaml          ← owns: cluster/ directory
│   ├── thirdparty.yaml       ← owns: thirdparty/ directory
│   └── owned.yaml            ← owns: owned/ directory
```

Every change to a manifest in the watched directories is picked up by ArgoCD within ~3 minutes of a git push.

### Envoy Gateway — GatewayClass and Gateway

```yaml
# cluster/gateways/gatewayclass.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

```yaml
# cluster/gateways/gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
  namespace: envoy-gateway-system
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
    - name: https
      protocol: HTTPS
      port: 443
      allowedRoutes:
        namespaces:
          from: All
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: platform-tls-cert
            namespace: envoy-gateway-system
```

One Gateway, one OCI Load Balancer (the free allocation). All services route through HTTPRoutes — no additional LoadBalancer services.

### CNPG — Platform Database

The CNPG operator manages PostgreSQL as a Kubernetes-native resource. A single `platform-db` cluster hosts all platform databases. Individual databases are provisioned as `Database` CRs without creating additional PVCs.

```yaml
# cluster/cloudnative-pg/cluster/cluster.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: platform-db
  namespace: cnpg-system
spec:
  instances: 1
  imageName: ghcr.io/cloudnative-pg/postgresql:17

  storage:
    size: 10Gi           # one PVC for the shared cluster

  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: "1"
      memory: 1Gi

  postgresql:
    parameters:
      max_connections: "200"
      shared_buffers: "128MB"
```

```yaml
# cluster/cloudnative-pg/cluster/keycloak-database.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Database
metadata:
  name: keycloak
  namespace: cnpg-system
spec:
  name: keycloak
  owner: keycloak
  cluster:
    name: platform-db
```

Adding a new service database is a new `Database` CR — no new PVC, no new cluster. See [Database — Adding a Service Database](#adding-a-service-database).

### ArgoCD Application for CNPG

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cloudnative-pg
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:kenanlalic/k8s-default-resources.git
    targetRevision: HEAD
    path: cluster/cloudnative-pg
  destination:
    server: https://kubernetes.default.svc
    namespace: cnpg-system
  syncPolicy:
    syncOptions:
      - ServerSideApply=true    # required — CNPG CRDs exceed annotation limit
      - CreateNamespace=true
```

---

## Design Decisions

### GitOps-First Bootstrap

|  | Choice | Alternative |
|---|---|---|
| **Bootstrap order** | ArgoCD first, base infra via GitOps | Gateway first, then ArgoCD |

ArgoCD is deployed before Envoy Gateway. Base infrastructure deploys as ArgoCD-managed Applications — under GitOps from their first apply. Eliminates the untracked window. See [Bootstrap Sequence](#bootstrap-sequence) for both paths.

### Sealed Secrets over External Secrets Operator

|  | Choice | Alternative |
|---|---|---|
| **Secret management** | Sealed Secrets | External Secrets Operator, Vault |

Sealed Secrets encrypts secrets with the cluster's public key and stores ciphertext in git. Survives ArgoCD reinstall. No external dependency. Vault remains the upgrade path for rotation requirements.

### `server.insecure=true` for ArgoCD

|  | Choice | Alternative |
|---|---|---|
| **ArgoCD TLS** | HTTP internally, TLS at Gateway | ArgoCD manages its own TLS |

TLS terminated at Envoy Gateway. Consistent with the platform's backchannel pattern. See [Backchannel Separation](../20-base-infrastructure/22-keycloak-oidc.md#confidential-client-with-backchannel-separation).

### No Gateway Basic Auth for ArgoCD

|  | Choice | Alternative |
|---|---|---|
| **ArgoCD auth** | ArgoCD native login form | Gateway-level Basic Auth |

ArgoCD is a Single Page Application — each unauthenticated API call re-triggers the browser Basic Auth prompt in a loop. The BasicAuth files are retained as reference for future per-service patterns.

### CNPG over Helm-managed PostgreSQL

|  | Choice | Alternative |
|---|---|---|
| **PostgreSQL** | CloudNativePG operator | Bitnami Helm chart, plain StatefulSet |

CNPG manages PostgreSQL lifecycle, connection health, and the `Database` CR pattern — one PVC for the shared cluster, individual databases provisioned as CRs without additional storage. Aligns with the per-service schema pattern from Docker Compose. Future upgrade path to HA is a configuration change (`instances: 2`), not a migration.

### Vendored Manifests

|  | Choice | Alternative |
|---|---|---|
| **Manifests** | Vendored at specific version in git | Pull from upstream at apply time |

No runtime GitHub dependency. Reproducible. Auditable. Version in filename makes the current version visible without reading the file.

### No Kubernetes Network Policies

|  | Choice | Alternative |
|---|---|---|
| **Pod isolation** | No NetworkPolicy — evaluated and not implemented | Calico / Cilium NetworkPolicy per namespace |

NetworkPolicy was evaluated and rejected for this platform. The reasoning:

A compromised Django pod already holds all credentials it needs — `DATABASE_URL`, `KEYCLOAK_CLIENT_SECRET`, `SECRET_KEY` — in its environment variables, and has an active connection pool to PostgreSQL. A NetworkPolicy blocking port 5432 prevents new TCP connections but does not invalidate already-open connections, does not prevent credential theft, and does not prevent replay of stolen credentials from outside the cluster. For Python/Django, arbitrary code execution implies environment variable access — the credential-free attack class that NetworkPolicy addresses is effectively theoretical here.

The effective blast radius control on this platform is PostgreSQL role isolation via CNPG `Database` CRs: each service has its own database owner role and cannot access other services' data even with a direct connection. That isolation is enforced at the PostgreSQL layer, not the network layer.

Additionally, OKE uses Flannel CNI by default, which does not enforce NetworkPolicy. Switching to Calico or Cilium is a cluster-level infrastructure change outside the platform layer's scope.

**Tradeoff accepted:** the marginal security improvement of NetworkPolicy on this platform does not justify the CNI change, the per-service policy maintenance burden, or the operational complexity added to a single-developer reference platform.

---

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| OKE cluster | Infrastructure | [OCI Infrastructure](../40-kubernetes/40-oci-infrastructure.md) — must exist before bootstrap |
| GitHub repository | Git | ArgoCD pulls manifests via SSH deploy key |
| Sealed Secrets controller | In-cluster | Must be running before any SealedSecret is applied |

---

## Configuration

### Version Reference

| Component | Version | Notes |
|---|---|---|
| ArgoCD | v3.3.2 | Latest stable as of March 2026 |
| Envoy Gateway | v1.7.0 | Latest stable as of March 2026 |
| cert-manager | v1.19.4 | Latest stable · fixes CVE-2026-24051, CVE-2025-68121 |
| CloudNativePG | v1.28.0 | Latest stable as of March 2026 |
| PostgreSQL | 17 | Via `ghcr.io/cloudnative-pg/postgresql:17` (ARM64 supported) |

### OCI Always Free Constraints

| Resource | Free Limit | This Setup |
|---|---|---|
| Flexible Load Balancers | 1 | 1 — auto-provisioned by Gateway controller |
| ARM64 compute | 4 oCPU / 24 GB total | 2 nodes × 2 oCPU / 12 GB |
| Block storage | 200 GB total (boot + block combined) | 144 GB boot + ~10 GB CNPG PVC = ~154 GB used |
| LB bandwidth | 10 Mbps | Sufficient for internal tooling |

> [!WARNING]
> Do not create additional `LoadBalancer` services — each provisions a new OCI LB and incurs cost outside the free tier. All services route through the single Gateway via HTTPRoutes.

---

## Bootstrap Sequence

Two paths reach the same end state. Both share the same hard constraints: Sealed Secrets must be first, SSH key must be sealed before ArgoCD can pull from git.

All commands run inside the `aio` container:

```bash
docker compose exec aio /bin/bash
```

### Prerequisites

| Check | Command | Expected |
|---|---|---|
| Cluster reachable | `kubectl get nodes` | 2 nodes · `Ready` · `arm64` |
| No prior ArgoCD | `kubectl get ns argocd` | `Error from server (NotFound)` |
| No prior Envoy Gateway | `kubectl get ns envoy-gateway-system` | `Error from server (NotFound)` |
| Vendored manifests committed | `git log --oneline cluster/gateways/envoy-gateway/` | commit present |

If a prior installation exists, run the [System Cleanup Runbook](https://github.com/kenanlalic/k8s-default-resources/blob/main/docs/system-cleanup.md) first.

### Primary Path — GitOps-First (recommended)

```
Sealed Secrets → SSH Key → ArgoCD → App-of-Apps → [base infra via ArgoCD]
```

**Step 1 — Sealed Secrets**

```bash
cd /workspace/k8s-default-resources
kubectl apply -k ./cluster/sealed-secrets/
kubectl rollout status deployment/sealed-secrets-controller \
  -n kube-system --timeout=120s
```

**Step 2 — GitHub SSH Key**

```bash
# Generate
ssh-keygen -t rsa -b 4096 \
  -C "argocd@k8s-default-resources" \
  -f /workspace/k8s-default-resources/credentials/github.ssh/id_rsa

# Add id_rsa.pub to GitHub → Repository → Settings → Deploy keys (read-only)

# Rename and populate the secret template
cp credentials/github.ssh/github-ssh.yaml.secret_template \
   credentials/github.ssh/github-ssh.yaml.secret
# Edit github-ssh.yaml.secret with your repo URL and private key content

# Seal
kubeseal --format yaml \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --secret-file credentials/github.ssh/github-ssh.yaml.secret \
  > credentials/github.ssh/github.com-k8s-default-resources.yaml

# Commit sealed output
git add credentials/github.ssh/github.com-k8s-default-resources.yaml
git commit -m "chore(secrets): seal github ssh deploy key"
git push

# Apply
kubectl create namespace argocd
kubectl apply -f credentials/github.ssh/github.com-k8s-default-resources.yaml
```

**Step 3 — ArgoCD**

```bash
# Vendor manifest (if not already committed)
curl -sSL \
  https://raw.githubusercontent.com/argoproj/argo-cd/v3.3.2/manifests/install.yaml \
  -o cluster/argocd/install-3.3.2-download.yaml
git add cluster/argocd/install-3.3.2-download.yaml
git commit -m "chore(argocd): vendor install manifest v3.3.2"
git push

# Apply
kubectl apply -k ./cluster/argocd --server-side --force-conflicts

kubectl rollout status deployment/argocd-server -n argocd --timeout=300s
kubectl rollout status deployment/argocd-repo-server -n argocd --timeout=300s
kubectl rollout status deployment/argocd-applicationset-controller -n argocd --timeout=300s
```

Retrieve password and access via port-forward:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d && echo

kubectl port-forward --address 0.0.0.0 svc/argocd-server -n argocd 8080:443
# Access http://localhost:8080 — username: admin
```

Change password immediately (**User Info → Update Password**), then delete the bootstrap secret:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

**Step 4 — App-of-Apps**

```bash
kubectl apply -k ./argocd-applications/app-of-apps
```

Base infrastructure (Envoy Gateway, GatewayClass, Gateway, cert-manager, CNPG) syncs automatically as ArgoCD-managed Applications. Monitor at `http://localhost:8080`.

```bash
# Watch certificate issuance
kubectl get certificate platform-tls-cert -n envoy-gateway-system --watch
# Expected: READY: True

# Watch CNPG cluster initialisation
kubectl get cluster platform-db -n cnpg-system --watch
# Expected: 1/1 instances ready
```

Once the certificate is ready, ArgoCD is accessible at `https://<ip>.sslip.io`.

### Alternative Path — Gateway-First

Use when multiple people need ArgoCD HTTPS access during bootstrap.

```
Sealed Secrets → SSH Key → Envoy Gateway → GatewayClass+Gateway
    → ArgoCD → cert-manager → App-of-Apps
```

Steps 1 and 2 are identical. After the SSH key is applied:

```bash
# Vendor and apply Envoy Gateway
curl -sSL https://github.com/envoyproxy/gateway/releases/download/v1.7.0/install.yaml \
  -o cluster/gateways/envoy-gateway/install-1.7.0-download.yaml
kubectl apply -k ./cluster/gateways/envoy-gateway/ --server-side
kubectl rollout status deployment/envoy-gateway -n envoy-gateway-system --timeout=120s

# GatewayClass + Gateway
kubectl apply -k ./cluster/gateways/
kubectl get gateway eg -n envoy-gateway-system --watch
# Expected: PROGRAMMED: True with external IP
```

Then follow steps 3–4 from the primary path. See [cert-manager Bootstrap](https://github.com/kenanlalic/k8s-default-resources/blob/main/docs/cert-manager-bootstrap.md) for the cert-manager runbook.

### Verify

```bash
kubectl get gateway eg -n envoy-gateway-system
kubectl get certificate platform-tls-cert -n envoy-gateway-system
kubectl get httproute -n argocd
kubectl get applications -n argocd
kubectl get cluster platform-db -n cnpg-system

kubectl run curl-test --image=curlimages/curl --restart=Never --rm -it -- \
  curl -sk -o /dev/null -w "%{http_code}" https://<your-domain>
# Expected: 200
```

---

## Database

### Current Setup — CNPG + OCI Native Block Storage

CloudNativePG manages a single `platform-db` PostgreSQL 17 cluster in the `cnpg-system` namespace. All platform services share this cluster via individual `Database` CRs. One 10 Gi PVC is created by the operator using OCI's native block storage CSI driver.

**Storage budget:**

```
200 GB free allocation
- 144 GB  boot volumes (2 × 72 GB)
-  10 GB  platform-db PVC
= ~46 GB  remaining for additional PVCs
```

**Current databases:**

| Database CR | Database name | Owner | Used by |
|---|---|---|---|
| `keycloak` | `keycloak` | `keycloak` | Keycloak (added at first service onboard) |

### Adding a Service Database

Each Django service gets its own database within the shared `platform-db` cluster. No new PVC is created.

```yaml
# cluster/cloudnative-pg/cluster/<service>-database.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Database
metadata:
  name: <service-name>
  namespace: cnpg-system
spec:
  name: <service-name>
  owner: <service-name>
  cluster:
    name: platform-db
```

Add to `cluster/cloudnative-pg/cluster/kustomization.yaml` and push. ArgoCD syncs within ~3 minutes.

### Database Options

Four approaches to PostgreSQL on this cluster, ordered by operational complexity.

---

#### Option A — CNPG + OCI Native Block Storage (current)

**How it works:** CNPG operator creates a `StatefulSet` with one PVC backed by an OCI Block Volume via the native CSI driver.

| | |
|---|---|
| ✅ | Zero manual storage management — OCI provisions and attaches the volume |
| ✅ | `Database` CR pattern — one PVC for the shared cluster, databases provisioned as CRs |
| ✅ | Aligns with Docker Compose per-service schema pattern |
| ✅ | CNPG handles PostgreSQL lifecycle and monitoring |
| ✅ | Backup via Barman Cloud Plugin to any S3-compatible store (OCI, AWS S3, Azure Blob) — optional, see note below |
| ⚠️ | `instances: 1` — no HA. Database downtime during node drain or OCI idle reclamation |
| ⚠️ | OCI Block Volumes are single-attach — volume follows the pod, not replicated |
| ⚠️ | Each additional CNPG cluster creates a new PVC consuming free block storage headroom |

**HA upgrade path:** change `instances: 2` and add a `PodAntiAffinity` rule to force primary and standby onto different nodes. Creates a second 10 Gi PVC (~20 Gi total). Provides automatic failover on node loss. Still within free headroom.

**Backup path (optional):** CNPG v1.26+ uses the [Barman Cloud Plugin](https://cloudnative-pg.io/plugin-barman-cloud/) for backup. The plugin writes WAL files and base backups to any S3-compatible object store — OCI Object Storage, AWS S3, Azure Blob Storage, or MinIO. On OCI, the Always Free tier includes 20 GB of object storage (separate from the 200 GB block volume allocation). A near-empty reference database fits within this limit. For a database with real data, provision a paid bucket or use a different provider. The `endpointURL` field in the `ObjectStore` CR is the only provider-specific value.

---

#### Option B — CNPG + Longhorn (alternative)

**How it works:** Same CNPG operator and `Cluster` CR — only the `storageClass` field changes to Longhorn's storage class. Longhorn carves space from existing boot volumes rather than creating new OCI Block Volumes.

```yaml
storage:
  size: 10Gi
  storageClass: longhorn   # instead of default OCI CSI
```

| | |
|---|---|
| ✅ | No new OCI Block Volumes — zero impact on the 200 GB free allocation |
| ✅ | Longhorn replicates across both nodes — data survives a node loss even with `instances: 1` |
| ✅ | `instances: 2` on Longhorn = full CNPG HA with no storage billing risk |
| ✅ | Safe to create multiple CNPG clusters without tracking PVC headroom |
| ⚠️ | Longhorn controller consumes ~200–300 MB RAM per node |
| ⚠️ | Node drain must wait for Longhorn replica sync before proceeding to the next node |
| ⚠️ | Longhorn volumes have lower IOPS than OCI native block volumes |
| ⚠️ | Additional operational surface: Longhorn health must be monitored alongside CNPG health |
| ❌ | ~20–30 GB usable across both nodes after OS and container image overhead — less headroom than OCI native at this boot volume size |

**When to choose this:** when you anticipate multiple CNPG clusters (multiple Django services each wanting their own cluster), or when you need CNPG HA without consuming block storage budget.

---

#### Option C — OCI Autonomous Database (free, Oracle dialect)

Two always-free instances with 1 OCPU and 20 GB storage each. Fully managed, zero ops. Not PostgreSQL — Oracle dialect. Django requires `cx_Oracle` driver and schema adjustments. Not compatible with the Docker Compose development pattern. Not recommended for this platform.

---

#### Option D — OCI Database with PostgreSQL (managed, paid)

Fully managed PostgreSQL-compatible service. 100% PostgreSQL compatible, near-instant failover, automated backups. Not in Always Free tier. The correct choice when free tier constraints are no longer relevant.

---

### Decision Summary

| Option | Free tier safe | HA available | Ops complexity | Recommended for |
|---|---|---|---|---|
| A — CNPG + OCI native (current) | ✅ within headroom | ⚠️ instances: 2 + headroom | Low | Reference platform, low PVC count |
| B — CNPG + Longhorn | ✅ zero PVC cost | ✅ Longhorn replication | Medium | Multiple services, HA without billing risk |
| C — OCI Autonomous DB | ✅ | ✅ managed | Zero | Oracle-native projects only |
| D — OCI DB with PostgreSQL | ❌ paid | ✅ managed | Zero | Production, budget available |

---

## Adding a Django Service

Django services generated from the [Copier template](../30-django-service-template/30-django-service-template.md) land in `argocd-applications/owned/`.

```bash
# 1. Generate the service
copier copy --trust gh:yourorg/django-service-template services/my-service

# 2. Add an ArgoCD Application manifest
# argocd-applications/owned/my-service.yaml

# 3. Add a Database CR
# cluster/cloudnative-pg/cluster/my-service-database.yaml

# 4. Push — ArgoCD picks up within ~3 minutes
git add argocd-applications/owned/my-service.yaml
git add cluster/cloudnative-pg/cluster/my-service-database.yaml
git commit -m "feat(argocd): add my-service"
git push
```

Each service gets an HTTPRoute under the platform Gateway, a database in `platform-db`, and connects to Keycloak via its OIDC client. See [Django K8s Migration Path](../30-django-service-template/30-django-service-template.md#kubernetes-migration-path) for the full service configuration.

---

## Dev vs Prod Differences

| Aspect | This Setup | Production Upgrade Path |
|---|---|---|
| Domain | `<ip>.sslip.io` | Real domain — update ClusterIssuer + Certificate |
| ArgoCD auth | Native admin login | Keycloak OIDC (added with first Django service) |
| Secrets | Sealed Secrets | Sealed Secrets or Vault |
| Database | CNPG `instances: 1`, OCI PVC | CNPG `instances: 2` + PodAntiAffinity, or OCI Database with PostgreSQL |
| Backchannel encryption | Plain HTTP (single host) | Service mesh mTLS or Keycloak-native TLS |
| Monitoring | Prometheus + Grafana (cluster layer) | Same + external alerting |

### Switching to a Real Domain

`sslip.io` is the bootstrap default — IP-based DNS, no registration required,
Let's Encrypt compatible. To cut over to a real domain, three files change and
one DNS record is added. No manifests outside these files need updating.

**Step 1 — Point your domain to the Gateway IP**

```bash
# Get the current Gateway external IP
kubectl get gateway eg -n envoy-gateway-system \
  -o jsonpath='{.status.addresses[0].value}'
```

Add an `A` record in Cloudflare (or your DNS provider) pointing your domain
to this IP. If using a wildcard (`*.yourdomain.com`), one record covers all
service subdomains.

**Step 2 — Update the ClusterIssuer**

`cluster/cert-manager/cluster-issuer.yaml` — no change needed. The Let's Encrypt
production issuer is domain-agnostic. Only update this file if switching between
staging and production ACME servers.

**Step 3 — Update the Certificate**

```yaml
# cluster/cert-manager/certificate.yaml
spec:
  dnsNames:
    - yourdomain.com
    - "*.yourdomain.com"   # covers all service subdomains
  secretName: platform-tls-cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

**Step 4 — Update the Keycloak redirect URIs**

Keycloak's realm template contains hardcoded redirect URIs. Update
`myrealm.json` to replace the sslip.io domain with your real domain, then
reimport the realm:

```bash
docker compose down -v && docker compose up -d
```

Or, for a live cluster, update redirect URIs directly in the Keycloak Admin
Console under **Clients → myclient → Valid redirect URIs**.

**Step 5 — Push and verify**

```bash
git add cluster/cert-manager/certificate.yaml
git commit -m "chore(tls): switch to yourdomain.com"
git push

# Watch cert-manager issue the new certificate (~1-3 min)
kubectl get certificate platform-tls-cert -n envoy-gateway-system --watch
# Expected: READY: True
```

> [!NOTE]
> cert-manager rotates the TLS secret in place — the Gateway picks up the new
> certificate automatically. No Gateway restart required.

---

## Runbooks

### Sealed Secrets controller not ready

```bash
kubectl describe pod -n kube-system \
  -l app.kubernetes.io/name=sealed-secrets | grep -A 10 Events
```

### kubeseal fails — controller not found

```bash
kubeseal --format yaml \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --secret-file github-ssh.yaml.secret \
  > github.com-k8s-default-resources.yaml
```

### Envoy Gateway not programming / no address

```bash
kubectl describe gateway eg -n envoy-gateway-system
kubectl logs -n envoy-gateway-system \
  -l app.kubernetes.io/name=envoy-gateway --tail=50
kubectl get svc -n envoy-gateway-system
```

### ArgoCD pods not starting

```bash
kubectl describe pod <pod-name> -n argocd | grep -A 10 Events
# Verify arm64 image
kubectl get pods -n argocd \
  -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}'
```

### Resources landed in default namespace

```bash
kubectl delete all -l app.kubernetes.io/part-of=argocd -n default --ignore-not-found
```

Re-apply with `--server-side --force-conflicts`.

### CNPG cluster not ready

```bash
# Check cluster status
kubectl describe cluster platform-db -n cnpg-system | grep -A 20 Status

# Check operator logs
kubectl logs -n cnpg-system \
  -l app.kubernetes.io/name=cloudnative-pg --tail=50

# Check PVC provisioned
kubectl get pvc -n cnpg-system
```

Common cause: OCI block volume provisioning delay on first deploy. Allow up to 5 minutes. If the PVC is stuck in `Pending`, check that the OCI CSI driver is running:

```bash
kubectl get pods -n kube-system | grep csi
```

### CNPG database unavailable after node drain

With `instances: 1`, draining the node running the PostgreSQL pod causes database downtime until the pod reschedules on the replacement node. This is expected behaviour for the single-instance configuration. Monitor pod rescheduling:

```bash
kubectl get pods -n cnpg-system -w
```

To avoid this in future, upgrade to `instances: 2` before performing maintenance. See [Database — Option A HA upgrade path](#option-a--cnpg--oci-native-block-storage-current).

### Full teardown and reinstall

See [System Cleanup Runbook](https://github.com/kenanlalic/k8s-default-resources/blob/main/docs/system-cleanup.md).

---

## ADRs

- **ADR: GitOps-first bootstrap** — Base infrastructure managed in git from first apply. No untracked window. ArgoCD accessible via port-forward during platform sync. Vendored manifests must be committed before app-of-apps runs.
- **ADR: Sealed Secrets first** — Every secret is a SealedSecret. Controller must exist before any can be decrypted. Immovable regardless of bootstrap path.
- **ADR: App-of-apps pattern** — Three entrypoints (`cluster`, `thirdparty`, `owned`) map to ownership boundaries. Adding a service is a git push.
- **ADR: Envoy Gateway as platform constant** — Same mental model from Docker Compose dev through K8s Gateway API. No context switch for engineers familiar with the base infrastructure layer.
- **ADR: `server.insecure=true`** — TLS at the Gateway boundary. Plain HTTP internally. Consistent with the platform's backchannel pattern.
- **ADR: Keycloak deferred to first service onboard** — No point bootstrapping identity infrastructure before there's a service to authenticate.
- **ADR: CNPG + OCI native block storage, instances: 1** — CloudNativePG chosen over plain Helm chart for operator-managed lifecycle, `Database` CR pattern, and future HA upgrade path (`instances: 2`). OCI native block storage chosen over Longhorn to avoid Longhorn operational overhead at this scale. Single instance accepted as the free-tier HA tradeoff — database downtime during node drain is acceptable for a reference platform. Upgrade path: `instances: 2` + `PodAntiAffinity` when HA is required. Alternative path: switch `storageClass` to Longhorn for zero PVC billing risk with built-in replication.
- **ADR: Shared `platform-db` cluster with `Database` CRs** — One CNPG cluster, multiple databases provisioned as CRs. Matches the Docker Compose per-service schema pattern. One PVC regardless of the number of service databases.
- **ADR: Vendored manifests** — No runtime GitHub dependency. Reproducible. Auditable.
- **ADR: One LoadBalancer constraint** — OCI Always Free allows exactly one Flexible LB. Gateway consumes it. All services route via HTTPRoutes.
- **ADR: `ServerSideApply=true` for CNPG** — CNPG CRDs exceed the client-side apply annotation size limit. Same reason ArgoCD itself requires `--server-side --force-conflicts`.
- **ADR: No Kubernetes Network Policies** — Evaluated and rejected. A compromised Django pod already holds all credentials in its environment — NetworkPolicy on port 5432 does not prevent credential theft or replay. Effective blast radius control is PostgreSQL role isolation via CNPG `Database` CRs. OKE's default Flannel CNI does not enforce NetworkPolicy without a cluster-level CNI change. Tradeoff: marginal security improvement does not justify infrastructure change and per-service maintenance burden on this platform.
- **ADR: CNPG backup not configured by default** — Backup via Barman Cloud Plugin is the documented path but is not deployed. On OCI Always Free, object storage is capped at 20 GB — sufficient for a reference platform's near-empty database, insufficient for production data. Backup configuration is provider-agnostic: only `endpointURL` changes between OCI, AWS S3, and Azure Blob. Accepted risk for a reference platform; implement before using for production data.

---

## References

- [CloudNativePG Documentation](https://cloudnative-pg.io/documentation/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Envoy Gateway Documentation](https://gateway.envoyproxy.io/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [sslip.io](https://sslip.io/)
- [OCI Infrastructure](../40-kubernetes/40-oci-infrastructure.md) — cluster this platform layer runs on
- [Django Service Template](../30-django-service-template/30-django-service-template.md) — services deployed to `owned/`
- [Keycloak OIDC](../20-base-infrastructure/22-keycloak-oidc.md) — identity provider added at first service onboard
