---
title: "OCI Infrastructure"
created: "2026-05-15"
updated: "2026-05-15"
version: 1.0.0
type: reference
owner: Kenan Lalic
lifecycle: production
tags: [kubernetes, oci, terraform, oracle, arm64, oke, free-tier, infrastructure]
---

# OCI Infrastructure — Terraform-Provisioned Kubernetes

Terraform-provisioned Kubernetes cluster on Oracle Cloud's Always Free tier. OKE control plane, ARM64 worker nodes, load balancer, object storage, and DNS — all within free allocations. Zero monthly cost.

> 📍 **Type:** Service Reference<br>
> 👤 **Owner:** Kenan Lalic<br>
> 🎯 **Outcome:** Understand the OCI infrastructure layer and how to provision it

> [!NOTE]
> ARM architecture only (`aarch64`). The Always Free tier does not include x86 instances. All images and workloads must support `linux/arm64`.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Design Decisions](#design-decisions)
- [Dependencies](#dependencies)
- [Configuration](#configuration)
- [Provisioning](#provisioning)
- [Cluster Operations](#cluster-operations)
- [Dev vs Prod Differences](#dev-vs-prod-differences)
- [Kubernetes Migration Path](#kubernetes-migration-path)
- [Runbooks](#runbooks)
- [ADRs](#adrs)
- [References](#references)

---

## Overview

The OCI infrastructure layer provisions a working Kubernetes cluster and stops. Everything that runs *on* the cluster — GitOps, ingress, auth, monitoring — is the platform layer's concern and lives in the K8s default resources repository. The two layers are independent: swap the platform layer without touching Terraform.

### What Gets Provisioned

A single `terraform apply` produces a complete cluster ready to receive workloads:

| Resource | Type | Notes |
|---|---|---|
| **VCN + Subnets** | OCI Networking | Private subnet for nodes, public subnet for load balancer and control plane endpoint |
| **OKE Control Plane** | Managed K8s | Free, managed by Oracle. Public endpoint enabled. |
| **Node Pool** | 2 × VM.Standard.A1.Flex | 2 oCPU / 12 GB each — 4 oCPU / 24 GB total. Boot volume 72 GB per node. |
| **Flexible Load Balancer** | Layer 7, 10 Mbps | Free — one per tenancy. Auto-provisioned by the Gateway controller. |
| **Object Storage Bucket** | `terraform-states` | Versioned. S3-compatible backend for Terraform state. |

### Infrastructure Layout

```
Oracle Cloud (Always Free)
├── VCN + Subnets
│   ├── OKE Control Plane (free, managed, public endpoint)
│   ├── Node Pool
│   │   ├── Worker 1 — VM.Standard.A1.Flex (2 oCPU, 12 GB, 72 GB boot)
│   │   └── Worker 2 — VM.Standard.A1.Flex (2 oCPU, 12 GB, 72 GB boot)
│   └── Load Balancer (Flexible, 10 Mbps, Layer 7, free)
└── Object Storage
    └── terraform-states bucket (versioned, S3-compatible)

Cloudflare (Free)
└── DNS management
```

### Cost

**$0/month.** Compute, control plane, load balancers, and object storage stay within the Always Free tier. DNS moved to Cloudflare to eliminate the last remaining OCI charge.

### Tooling

You do not install Terraform, the OCI CLI, or kubectl on your machine. The `aio` (all-in-one) container bundles every tool needed to provision and manage the cluster. `compose.yaml` mounts your local `credentials/` directory into the container — files created in your project folder are immediately available inside.

```
./oci-infrastructure/
├── terraform/infra/       # OKE cluster, VCN, subnets, node pool
├── credentials/           # OCI API keys, SSH keys, kubeconfig (gitignored)
├── docker/                # Dockerfile for the aio tooling container
└── compose.yaml           # Development environment
```

---

## Architecture

```
Oracle Cloud (Always Free)
│
├── VCN
│   ├── Public Subnet ──── OCI Flexible Load Balancer (1 free, auto-provisioned)
│   │                           ↓ HTTPS :443 / HTTP :80
│   │                      Envoy Gateway (K8s Gateway API)
│   │                           ↓ routes to cluster services
│   └── Private Subnet ─── OKE Node Pool
│                           ├── Worker 1 (ARM64, 2 oCPU, 12 GB, 72 GB boot)
│                           └── Worker 2 (ARM64, 2 oCPU, 12 GB, 72 GB boot)
│
└── Object Storage ──────── terraform-states bucket (Terraform backend)

Cloudflare
└── DNS → OCI Load Balancer IP
```

---

## Design Decisions

### All-in-One Tooling Container

|  | Choice | Alternative |
|---|---|---|
| **Tooling** | `aio` container via Docker Compose | Install Terraform + OCI CLI + kubectl on host |

The `aio` container is the only host dependency beyond Docker itself. Eliminates version conflicts, keeps the host clean, and ensures reproducible tooling across team members.

### Cloudflare DNS over OCI DNS

|  | Choice | Alternative |
|---|---|---|
| **DNS** | Cloudflare free tier | OCI DNS (incurs cost outside Always Free) |

OCI DNS is not included in the Always Free tier. Cloudflare's free tier eliminates the last charge.

### ARM64 Only

|  | Choice | Alternative |
|---|---|---|
| **Compute** | VM.Standard.A1.Flex (ARM64) | x86 instances (not available on Always Free) |

The Always Free tier provides 4 oCPU and 24 GB RAM on ARM64 at no cost. x86 Always Free compute is limited to AMD micro instances (1 oCPU / 1 GB) — insufficient for Kubernetes. All platform images (Envoy Gateway, Keycloak, ArgoCD, CNPG, Django) publish ARM64 variants.

### Boot Volume at 72 GB

|  | Choice | Alternative |
|---|---|---|
| **Boot volume** | 72 GB per node | 50 GB (OKE default) · 100 GB (common recommendation) |

See [ADR: Boot volume sized at 72 GB](#adrs) for the full reasoning. Short version: 100 GB per node exhausts the 200 GB free block storage allocation with zero headroom for PVCs. 72 GB is the deliberate balance point between node storage capacity and PVC budget.

### Terraform State in OCI Object Storage

|  | Choice | Alternative |
|---|---|---|
| **State backend** | OCI Object Storage (S3-compatible) | Local state |

Remote state enables team collaboration and prevents state file loss. Versioning provides state history. Local state is supported — comment out the backend block in `_terraform.tf` for single-developer setups.

---

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| OCI Always Free account | Cloud | Infrastructure host |
| Docker + Compose | Host tooling | Only host dependency — all other tools run in the `aio` container |
| Cloudflare account | DNS | Free DNS management — only needed for the platform layer, not Terraform |

---

## Configuration

### Block Storage Budget

> [!WARNING]
> The Always Free tier provides **200 GB of block storage total**, shared between boot volumes and PersistentVolumeClaims. Boot volumes count against this limit.
>
> ```
> 2 nodes × 72 GB boot = 144 GB consumed by boot volumes
> Remaining free headroom:  56 GB for PVCs
> CNPG platform-db PVC:    ~10 GB consumed
> Safe remaining budget:   ~46 GB
> ```
>
> Do not increase boot volume size without recalculating headroom. At 2 × 100 GB boot volumes (a common recommendation), the entire free allocation is consumed and every PVC incurs charges at $0.0255/GB/month with no grace period.

### Terraform Variables

Copy the template and edit with your values:

```bash
cd terraform/infra
cp terraform.tfvars_template terraform.tfvars
```

| Variable | Purpose |
|---|---|
| `compartment_id` | OCI tenancy OCID — the `tenancy` value from `credentials/.oci/config` |
| `kubernetes_version` | Target K8s version — controls both control plane and node pool image selection |
| `kubernetes_worker_nodes` | Number of worker nodes — set to `2` for the Always Free 2-node setup |
| `ssh_public_key_path` | Path to SSH public key for OKE node access |
| See `terraform.tfvars_template` | All required variables documented in the template |

`terraform.tfvars` contains sensitive values and is gitignored. Only the `_template` file is committed.

### Node Pool Configuration

The node pool is provisioned with the following key settings:

```hcl
node_shape = "VM.Standard.A1.Flex"

node_shape_config {
  memory_in_gbs = 12
  ocpus         = 2
}

node_source_details {
  image_id    = <latest ARM64 OKE image for kubernetes_version>
  source_type = "image"
  boot_volume_size_in_gbs = 72   # see ADR: Boot volume sized at 72 GB
}
```

The node pool image is dynamically selected at apply time — always the latest ARM64 OKE-optimised image for the target Kubernetes version. Nodes are distributed across all available Availability Domains via a dynamic `placement_configs` block. With `size = 2`, OCI places nodes in two separate ADs.

### Credentials Structure

All credential files are created in your local project folder under `credentials/.oci/`. The `compose.yaml` mounts this directory into the `aio` container — changes on either side are reflected immediately.

```
credentials/.oci/
├── config                      # OCI CLI + Terraform configuration
├── oci_api_key.pem             # Private key (API authentication)
├── oci_api_key_public.pem      # Public key (API authentication)
└── oci_api_key_public.pub      # SSH public key (OKE node access)
```

> [!WARNING]
> Verify that `credentials/` is listed in `.gitignore` before proceeding. These files must never be committed.

### OCI Config File

```ini
# credentials/.oci/config

# --- OCI API authentication (Terraform resource creation, oci CLI) ---
[DEFAULT]
user=ocid1.user.oc1..xxx
fingerprint=xx:xx:xx:...
tenancy=ocid1.tenancy.oc1..xxx
region=eu-frankfurt-1
key_file=~/.oci/oci_api_key.pem

# --- S3-compatible credentials (Terraform state backend only) ---
[default]
aws_access_key_id=<customer-secret-key-id>
aws_secret_access_key=<customer-secret-key>
```

The `[DEFAULT]` section authenticates all OCI API calls. The `[default]` section with `aws_*` keys is only used by Terraform to read/write state from the `terraform-states` bucket. Only required if using the remote backend.

### OCI Always Free Constraints

| Resource | Free Limit | This Setup |
|---|---|---|
| Flexible Load Balancers | 1 | 1 — auto-provisioned by Gateway controller |
| ARM64 compute | 4 oCPU / 24 GB total | 2 nodes × 2 oCPU / 12 GB |
| Block storage | 200 GB total (boot + block combined) | 144 GB boot volumes + ~10 GB CNPG PVC = ~154 GB used · ~46 GB remaining |
| Object storage | 20 GB | Terraform state only — minimal usage |
| LB bandwidth | 10 Mbps | Sufficient for internal tooling |

### Terraform Provider

The OCI provider is pinned to `~> 7.19.0` in `_terraform.tf`. To update:

```bash
terraform init -upgrade
```

> [!NOTE]
> Terraform ≥ v1.12 is required when using the remote OCI backend. With the backend commented out, any recent version works.

---

## Provisioning

All commands run inside the `aio` container:

```bash
docker compose up -d
docker compose exec aio /bin/bash
```

### Step 1 — Generate OCI API Keys

In the Oracle Cloud Console, navigate to **Identity → My Profile → API Keys**:

1. Click **Add API key → Generate API key pair**
2. Save the private key to `credentials/.oci/oci_api_key.pem`
3. Save the public key to `credentials/.oci/oci_api_key_public.pem`
4. Click **Add** — copy the **Configuration file preview** into `credentials/.oci/config`
5. Set the `key_file` path to the container mount point: `key_file=~/.oci/oci_api_key.pem`

### Step 2 — Set File Permissions

```bash
chmod 600 ~/.oci/config ~/.oci/oci_api_key.pem
```

> [!WARNING]
> The OCI CLI refuses to run if config permissions are too open. Fix the permissions — do not suppress this warning with environment variables.

### Step 3 — Verify OCI CLI Access

```bash
oci iam user list --all
```

Expected: your user information in JSON.

### Step 4 — Generate SSH Key for OKE Nodes

```bash
ssh-keygen -y -f ~/.oci/oci_api_key.pem > ~/.oci/oci_api_key_public.pub
```

### Step 5 — (Optional) Create Terraform State Bucket

Only required if using the remote backend:

```bash
oci os bucket create \
  --name terraform-states \
  --versioning Enabled \
  --compartment-id <your-tenancy-ocid>
```

Create S3-compatible credentials in the Oracle Cloud Console at **Identity → User → Customer Secret Keys** and add them to the `[default]` section of `credentials/.oci/config`.

### Step 6 — Provision the Cluster

```bash
cd terraform/infra
terraform init
terraform plan
terraform apply
```

Type `yes` when prompted. Creates the VCN, subnets, OKE cluster, and node pool. Takes 10–25 minutes.

### Step 7 — Verify

```bash
kubectl get nodes
```

Expected: two `Ready` ARM64 nodes. The kubeconfig is at `terraform/infra/.kube.config` and is used automatically inside the `aio` container.

---

## Cluster Operations

> [!WARNING]
> Oracle reclaims idle Always Free instances with CPU utilisation below 20% (95th percentile) over any 7-day period. A running cluster with active workloads (Keycloak, Django services, CNPG) stays above threshold. An empty cluster may not.

### Kubernetes Version Upgrade

> [!IMPORTANT]
> Only upgrade to the version shown by `available-kubernetes-upgrades`. Incremental upgrades are safer than skipping minor versions. The Kubernetes version skew policy allows kubelets to trail by three minor versions.

```bash
cd terraform/infra

# Check available upgrade versions
oci ce cluster get \
  --cluster-id $(terraform output --raw k8s_cluster_id) \
  | jq -r '.data."available-kubernetes-upgrades"'

# Update the version variable in _variables.tf
# Then apply
terraform apply
```

### Rolling Node Replacement

Terraform upgrades the control plane and node pool configuration but does not replace running nodes. Roll them one at a time:

```bash
# Drain and cordon
kubectl drain <node-name> --force --ignore-daemonsets --delete-emptydir-data
kubectl cordon <node-name>

# Terminate the instance via OCI Console → OKE → Node Pool → match by private IP
oci compute instance terminate --force --instance-id <instance-ocid>

# Wait for replacement node
kubectl get nodes -w
```

> [!WARNING]
> `--delete-emptydir-data` deletes emptyDir volumes on drain. Ensure Django media uploads use a PVC or object storage, not emptyDir. CNPG data is safe — it uses a persistent block volume that reattaches to the replacement node.

> [!NOTE]
> With CNPG `instances: 1`, draining the node running the PostgreSQL pod causes database downtime until the pod reschedules on the replacement node. This is the accepted tradeoff for the single-instance free tier configuration. See [K8s Default Resources — Database Options](../40-kubernetes/41-k8s-default-resources.md#database) for the HA upgrade path.

### Cluster Access Outside the Container

```bash
export KUBECONFIG=<path-to-repo>/terraform/infra/.kube.config
```

---

## Dev vs Prod Differences

| Aspect | This Setup (Free Tier) | Production Upgrade Path |
|---|---|---|
| Compute | ARM64 only | Add x86 node pool (paid) |
| Availability | 2 nodes across ADs | Additional nodes across all ADs |
| Database | CNPG single instance, OCI PVC | CNPG `instances: 2` + PodAntiAffinity, or OCI Database with PostgreSQL (managed) |
| Secrets | Sealed Secrets | Sealed Secrets or Vault |
| Monitoring | Prometheus + Grafana (platform layer) | Same + external alerting |
| Cost | $0 | Pay-as-you-go beyond free limits |

---

## Kubernetes Migration Path

See the [What Changes Between Tiers](../00-index/01-infrastructure-engineering-hub.md#what-changes-between-tiers) table for the full platform migration map. This infrastructure layer is the K8s tier target — application images, Envoy Gateway config, and Keycloak setup are identical to the VPS tier. Only the orchestration layer changes.

### Cloud Provider Portability

The **Kubernetes application layer is fully provider-agnostic.** Django services, Keycloak, Envoy Gateway (K8s Gateway API), ArgoCD, cert-manager, Sealed Secrets, and CNPG all run identically on EKS, AKS, GKE, or any conformant managed Kubernetes service. The `GatewayClass`, `Gateway`, and `HTTPRoute` manifests are Kubernetes-standard — no vendor extensions.

The **Terraform layer requires provider-specific substitution.** This repository provisions OKE. Deploying to AWS or Azure means replacing `terraform/infra/` with an equivalent module for the target provider. The application layer above it is unchanged.

| OCI Component | AWS Equivalent | Azure Equivalent |
|---|---|---|
| OKE | EKS | AKS |
| OCI Flexible Load Balancer | AWS NLB/ALB | Azure Load Balancer |
| OCI Block Volume CSI (`oci-bv`) | AWS EBS CSI (`gp3`) | Azure Disk CSI (`managed-premium`) |
| OCI Object Storage (Terraform state) | S3 | Azure Blob Storage |
| Terraform OCI provider | Terraform AWS provider | Terraform AzureRM provider |
| `credentials/.oci/` + `oci` CLI | `~/.aws/credentials` + `aws` CLI | Service principal + `az` CLI |

> [!NOTE]
> The CNPG `Cluster` CR references `storageClass: oci-bv` (the OCI native CSI driver).
> On AWS or Azure, update this field to the equivalent storage class (`gp3` or `managed-premium`).
> Option B (Longhorn) avoids this entirely — Longhorn's storage class name is consistent
> across providers.

---

## Runbooks

### "Out of host capacity" on node pool creation

The most common failure. The OKE control plane creates successfully but ARM worker nodes fail because the region has no available A1 capacity.

**Fixes, in order of effectiveness:**

1. **Upgrade to Pay As You Go** — PAYG accounts access a larger capacity pool. Still $0 within Always Free limits.
2. **Retry `terraform apply`** — capacity fluctuates. Early morning UTC (02:00–08:00) and weekends have better availability.
3. **Automate retries** — `terraform apply -auto-approve` on a cron every 5–10 minutes until it succeeds.

### `terraform destroy` fails on VCN

OCI load balancers or resources created by Kubernetes hold references to VCN subnets. Delete load balancers manually in the OCI Console (**Networking → Load Balancers**), then re-run `terraform destroy`.

### OCI CLI refuses to run

```bash
chmod 600 ~/.oci/config ~/.oci/oci_api_key.pem
```

### Unexpected block storage charges

Check the total across all block and boot volumes in your tenancy. The 200 GB free allocation covers both. Any volume that pushes the total past 200 GB triggers billing.

```bash
# List all block volumes in tenancy
oci bv volume list --compartment-id <tenancy-ocid> \
  --query 'data[].{"name":"display-name","size":"size-in-gbs"}' \
  --output table

# List all boot volumes
oci bv boot-volume list --compartment-id <tenancy-ocid> \
  --availability-domain <ad-name> \
  --query 'data[].{"name":"display-name","size":"size-in-gbs"}' \
  --output table
```

### Nodes stopped by Oracle idle reclamation

Oracle stopped the instances due to CPU utilisation below 20% for 7 days. Restart via OCI Console → Compute → Instances. Deploy active workloads (Keycloak, Django services) to keep utilisation above threshold.

---

## ADRs

- **ADR: aio tooling container** — Single host dependency (Docker). Eliminates version conflicts and ensures reproducible tooling across team members.
- **ADR: ARM64 only** — Always Free tier provides 4 oCPU / 24 GB on ARM64 at zero cost. All platform images publish ARM64 variants. x86 Always Free instances are insufficient for K8s.
- **ADR: Cloudflare DNS** — OCI DNS incurs cost outside Always Free. Cloudflare free tier eliminates the last charge.
- **ADR: Boot volume sized at 72 GB** — The Always Free tier provides 200 GB of block storage total, shared between boot volumes and PersistentVolumeClaims. The OKE default is 50 GB per node (leaving 100 GB for PVCs but reducing in-node container image capacity). A common recommendation of 100 GB per node exhausts the free allocation entirely with zero PVC headroom. 72 GB (144 GB total for 2 nodes) leaves ~56 GB for PVCs — enough for CNPG (10 Gi) plus Keycloak and future services, while maintaining adequate node storage for container images and system pods.
- **ADR: Terraform state in OCI Object Storage** — Remote state enables collaboration and prevents loss. S3-compatible backend, versioned, within free object storage limits. Local state supported by commenting out the backend block.
- **ADR: Platform layer separation** — Terraform provisions infrastructure and stops at a working kubeconfig. Everything on the cluster is the platform layer's concern. Either layer can be replaced without touching the other.
- **ADR: Dynamic node pool image selection** — Node pool image is resolved at apply time via a `jq_query` data source filtering for the latest ARM64 OKE image matching the target Kubernetes version. This ensures nodes always use the correct OKE-optimised image without manual updates.

---

## References

- [OCI Always Free tier details](https://blogs.oracle.com/cloud-infrastructure/post/oracle-builds-out-their-portfolio-of-oracle-cloud-infrastructure-always-free-services)
- [OCI Load Balancer annotations](https://github.com/oracle/oci-cloud-controller-manager/blob/master/docs/load-balancer-annotations.md)
- [Kubernetes version skew policy](https://kubernetes.io/releases/version-skew-policy/#kubelet)
- [Terraform S3 backend configuration](https://developer.hashicorp.com/terraform/language/backend/s3)
- [CloudNativePG Documentation](https://cloudnative-pg.io/documentation/)
- [Arnold Galovics — Free Kubernetes on Oracle Cloud](https://arnoldgalovics.com/free-kubernetes-oracle-cloud/)
- [piontec/free-oci-kubernetes](https://github.com/piontec/free-oci-kubernetes)
- [K8s Default Resources](../40-kubernetes/41-k8s-default-resources.md) — platform layer running on this cluster
