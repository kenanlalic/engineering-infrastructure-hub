---
title: "Infrastructure Engineering Hub"
created: "2026-02-01"
updated: "2026-05-15"
version: 2.0.0
type: reference
owner: Kenan Lalic
lifecycle: production
tags: [hub, infrastructure, overview, scaling, kubernetes, django, envoy, keycloak]
---

# Infrastructure Engineering Hub

**Scalable open-source infrastructure for microservices — from local development through VPS production to Kubernetes with minimal friction.**

> 📍 **Type:** Map of Content (MOC)<br>
> 👤 **Owner:** Kenan Lalic<br>
> 🎯 **Outcome:** Understand the platform, the architecture, and where everything lives

---

## Table of Contents

1. [Platform at a Glance](#platform-at-a-glance)
2. [Architecture](#architecture)
3. [Scaling Path](#scaling-path)
4. [Stack Reference](#stack-reference)

---

## Platform at a Glance

A scalable open-source infrastructure for microservices — from local development through VPS production to Kubernetes — with minimal configuration changes between tiers. All components are open-source and interchangeable. The local development experience is first-class, and the same containers that run on your machine deploy to production without modification.

The platform follows a natural progression through four layers:

**Workspace → Infrastructure → Services → Dev → Prod VPS → Prod K8s**
Every section in this document follows this progression.

### 🗂️ VS Code Workspace Setup

A multi-root workspace configuration that manages all services, infrastructure, and this documentation in a single VS Code instance. Each service gets its own debugger launch configuration. The VS Code Test Explorer runs inside a dev container with parallelised per-test debugging capabilities.

- [1.0 - Workspace Config](../10-workspace-config/10-workspace-config.md)

### 🏗️ Base Infrastructure

All services run behind **Envoy Gateway**, which handles TLS termination, path-based routing, and header manipulation. A `generate_certs` sidecar provisions internal certificates. **Keycloak** provides OIDC/SSO with a pre-configured realm template for development. **PostgreSQL** runs as a shared instance with per-service schemas, initialised via startup scripts. **MailDev** (dev only) captures all outbound email for inspection. An **echo server** (dev only) mirrors requests for gateway debugging.

Pre-configured configs ship for Envoy Gateway (dev and prod-ready for VPS, minimal changes for K8s), Keycloak realm import (dev), PostgreSQL database initialisation scripts, and Envoy secrets generation scripts.

In development, **Ngrok** exposes the entire local stack via a public HTTPS URL — enabling OAuth callbacks, webhook integrations, mobile testing, and client demos against your actual environment. In production on VPS, Ngrok is replaced by direct public exposure through Envoy with Let's Encrypt certificates.

- [2.0 - PostgreSQL](../20-base-infrastructure/20-postgresql.md)
- [2.1 - Envoy Gateway](../20-base-infrastructure/21-envoy-gateway.md)
- [2.2 - Keycloak OIDC](../20-base-infrastructure/22-keycloak-oidc.md)
- [2.3 - MailDev](../20-base-infrastructure/23-maildev.md)

### 🧩 Django Service Template

New services are generated via an updatable [Copier Template](https://copier.readthedocs.io/en/stable/) in a few steps. Each service ships with four apps following [HackSoft Styleguide 2+](https://github.com/HackSoftware/Django-Styleguide) best practices: API (Django REST Framework), authentication (django-allauth + Keycloak OIDC), authorization (RBAC with Keycloak as source of truth), and core (admin panel, healthchecks). An optional public app provides a django-htmx powered frontend with branding, SEO, and analytics setup. Pre-commit hooks, linting, and formatting are included.

Multiple Django services can be generated from the same template and deployed on the same infrastructure under different subdomains — each service registers as a new Envoy route and Keycloak client within the existing setup.

- [3.0 - Django Service Template](../30-django-service-template/30-django-service-template.md)

### ⎈ Kubernetes — OCI Free Tier

The same Django services and infrastructure components that run on Docker Compose scale directly to Kubernetes on Oracle Cloud's Always Free tier. **OKE** provides a managed control plane. **ARM64 worker nodes** run all workloads. **Envoy Gateway** translates directly from the Docker Compose config to K8s Gateway API resources — no rewrite. **ArgoCD** manages the cluster via GitOps using an app-of-apps pattern. **cert-manager** handles TLS with Let's Encrypt. **Sealed Secrets** provides GitOps-safe secret management.

Zero monthly cost. The same application images that run locally deploy to production without modification.

> [!NOTE]
> ARM architecture only (`aarch64`). All images and workloads must support `linux/arm64`.

- [4.0 - OCI Infrastructure](../40-kubernetes/40-oci-infrastructure.md)
- [4.1 - K8s Default Resources](../40-kubernetes/41-k8s-default-resources.md)

---

## Architecture

Instead of each service managing its own authentication, routing, and database, these concerns are centralised into shared infrastructure.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              INTERNET                                    │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
                                │ HTTPS (443) / HTTP (80 → 301 redirect)
                                ▼
                     ┌─────────────────────┐
                     │    Ngrok Tunnel     │    (dev only — replaced by
                     │  (Public URL →      │     direct DNS in prod)
                     │   Local Stack)      │
                     └──────────┬──────────┘
                                │
                                ▼
                     ┌─────────────────────┐
                     │   Envoy Gateway     │
                     │  ─────────────────  │
                     │  TLS Termination    │
                     │  Path-Based Routing │
                     │  Rate Limiting      │
                     │  HTTP → HTTPS       │
                     │  X-Forwarded-*      │
                     └──────────┬──────────┘
                                │
           ┌────────────────────┼────────────────────┐
           │                    │                    │
           ▼                    ▼                    ▼
   ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
   │  Keycloak     │    │  Django       │    │  Django       │
   │  ──────────   │    │  Service A    │    │  Service N    │
   │  OIDC / SSO   │◄───│  ──────────   │    │  ──────────   │
   │  RBAC Roles   │    │  allauth      │    │  allauth      │
   │  User Mgmt    │    │  RBAC sync    │    │  RBAC sync    │
   │  Source of    │    │  DRF + htmx   │    │  DRF + htmx   │
   │  Truth        │    │               │    │               │
   └──────┬────────┘    └───────┬───────┘    └───────┬───────┘
          │                     │                    │
          │         ┌───────────┴────────────────────┘
          │         │
          ▼         ▼
   ┌──────────────────────┐    ┌───────────────┐    ┌──────────────┐
   │  PostgreSQL          │    │  MailDev      │    │  Echo        │
   │  ─────────────────   │    │  ──────────   │    │  ──────────  │
   │  Shared Instance     │    │  SMTP Capture │    │  Request     │
   │  Per-Service Schemas │    │  Web UI       │    │  Debugging   │
   └──────────────────────┘    │  (dev only)   │    │  (dev only)  │
                               └───────────────┘    └──────────────┘

   ┌──────────────────────────────────────────────────────────────┐
   │  VS Code Multi-Root Workspace                                │
   │  ─────────────────────────────                               │
   │  Per-service debuggers · Parallelised test runner            │
   │  File exclusions · Shared settings                           │
   └──────────────────────────────────────────────────────────────┘
```

The `◄───` arrow between Django Service A and Keycloak represents the internal backchannel: Django exchanges tokens with Keycloak directly over the Docker bridge network, never routing through the public gateway. See [Backchannel Separation](../20-base-infrastructure/22-keycloak-oidc.md#confidential-client-with-backchannel-separation) for details.

### Component Responsibilities

| Component | What It Does | Swap It For |
| --- | --- | --- |
| **Envoy Gateway** | Routes traffic by path/subdomain, terminates TLS, rate limiting, HTTP→HTTPS redirect | Traefik, NGINX, Kong |
| **Keycloak** | SSO, OIDC, user management, RBAC source of truth, synced to services via django-allauth | Any OIDC provider |
| **PostgreSQL** | Shared database instance, per-service schemas, initialised via startup scripts | Any SQL backend |
| **MailDev** *(dev only)* | Captures all outbound email, web UI for inspection | Mailhog, Mailtrap, any SMTP provider |
| **Echo** *(dev only)* | Mirrors requests back for gateway route debugging | Any echo server |
| **Ngrok** *(dev only)* | Exposes local stack via public HTTPS URL for OAuth callbacks, webhooks, mobile testing | Cloudflare Tunnel, localtunnel — replaced by direct DNS in prod |
| **Django Services** | Generated from updatable Copier template, each with API, auth, RBAC, admin, optional htmx frontend | — |
| **VS Code Workspace** | Multi-root config with per-service debuggers and parallelised test execution | JetBrains (manual setup) |
| **ArgoCD** *(K8s only)* | GitOps controller — app-of-apps pattern, pull-based deployment from git | Flux |
| **Sealed Secrets** *(K8s only)* | GitOps-safe secret management — secrets encrypted at rest in git | External Secrets Operator, Vault |
| **cert-manager** *(K8s only)* | TLS certificate lifecycle, Let's Encrypt integration, Gateway API support | Manual cert rotation |
| **OKE** *(K8s only)* | Managed Kubernetes control plane on Oracle Cloud Always Free | Any managed K8s (EKS, GKE, AKS) |

---

## Scaling Path

> **Same images, different orchestration.**
> Your application code and Docker images never change between tiers. Only the orchestration layer, secrets management, and infrastructure configuration evolve.

### 🛠️ Local Development — Docker Compose

Your laptop runs the full stack. Envoy routes, Keycloak authenticates, PostgreSQL stores, MailDev captures email, VS Code debugs. Ngrok exposes everything via a public HTTPS URL — OAuth callbacks, webhook integrations, mobile testing, client demos all work against your actual dev environment. A separate `compose.yaml` wires up dev-only services (MailDev, echo server) and relaxed configurations (permissive hostnames, plaintext secrets, hot-reload).

### 🚀 MVP Production — Docker Compose on VPS

Deploy `compose.prod.yaml` to a single VPS (Hostinger, Kamatera, etc.). Dev-only services are gone. Let's Encrypt replaces Ngrok for public TLS. Keycloak locks down its hostname and issuer. Secrets come from environment files outside the repo. The same Envoy Gateway configs that ran locally now serve production traffic — no rewrite, no translation layer. This tier carries most startups through their first real scaling event.

### ⎈ Production at Scale — Kubernetes

Each container becomes a pod. Envoy Gateway configs translate directly to Kubernetes Gateway API resources. PostgreSQL moves to a managed service or Longhorn-backed stateful deployment. Keycloak scales horizontally. Backchannel encryption becomes a requirement — via service mesh mTLS or Keycloak-native TLS. ArgoCD manages all workloads via GitOps. The application images stay identical; only the orchestration and persistence layers change.

See [OCI Infrastructure](../40-kubernetes/40-oci-infrastructure.md) and [K8s Default Resources](../40-kubernetes/41-k8s-default-resources.md) for the full K8s setup.

### What Changes Between Tiers

| Component | 🛠️ Local Dev | 🚀 VPS Prod | ⎈ K8s Prod |
| --- | --- | --- | --- |
| **Compose file** | `compose.yaml` | `compose.prod.yaml` | kustomize manifests + ArgoCD |
| **Public access** | Ngrok tunnel | Direct DNS + Let's Encrypt | Ingress / Gateway API |
| **TLS** | Ngrok handles it | Let's Encrypt at Envoy | cert-manager + Gateway API |
| **Envoy Gateway** | File provider, relaxed | File provider, prod-ready | K8s Gateway API (native) |
| **Keycloak hostname** | `--hostname-strict=false` | `KC_HOSTNAME=https://${DOMAIN}/auth` | Same, or dedicated auth subdomain |
| **Keycloak realm** | `--import-realm` from JSON | `--import-realm` (planned: config-cli) | `keycloak-config-cli` or Terraform |
| **Backchannel** | Plain HTTP, Docker bridge | Plain HTTP, Docker bridge | Encrypted (mesh mTLS or native TLS) |
| **Secrets** | `.env` files, plaintext | `.env` files outside repo | Sealed Secrets (Vault: upgrade path for rotation requirements) |
| **PostgreSQL** | Shared container | Shared container + persistent volume | Longhorn-backed stateful or managed service |
| **Email** | MailDev (captures all) | Real SMTP provider | Real SMTP provider |
| **Echo server** | ✅ Running | — | — |
| **Monitoring** | Docker logs + Admin Console | Docker logs + Admin Console | Prometheus + Grafana |
| **Scaling** | Single instance | Single instance | Horizontal pod autoscaling |
| **Deploy method** | `docker compose up` | `docker compose up` | GitOps via ArgoCD <!-- TODO: decide on CI/CD pipeline strategy — tag-triggered GitHub Actions vs pure pull-based GitOps --> |
| **Image architecture** | amd64 | amd64 | arm64 (OCI Always Free) |

---

## Stack Reference

Technologies listed by platform layer. Each layer corresponds to a separate repository. See [Platform at a Glance](#platform-at-a-glance) for how these layers connect.

### 🗂️ Workspace Setup

> Configuration and tooling for the multi-service development experience.

**VS Code Workspace** — Multi-root workspace managing all services, infrastructure, and documentation. Per-service debugger launch configs and parallelised test execution via dev containers. → [Configuration](../10-workspace-config/10-workspace-config.md#overview) · [Setup](../10-workspace-config/10-workspace-config.md#setup) · [Debugging](../10-workspace-config/10-workspace-config.md#debugging)

### 🏗️ Base Infrastructure

> Shared services that every Django service depends on. Run with `docker compose up`.

**Envoy Gateway** — Front door for all traffic. TLS termination, path-based and subdomain routing, HTTP→HTTPS redirect, header manipulation. Configs are file-based, prod-ready for VPS, and translate to Kubernetes Gateway API with minimal changes. → [Setup & Configuration](../20-base-infrastructure/21-envoy-gateway.md#overview)

**Keycloak** — Centralised identity provider. OIDC/SSO authentication, RBAC group management, user lifecycle. Single realm (`myrealm`), synced to Django services via django-allauth on every login. Admin console for user and session management. → [Setup & Configuration](../20-base-infrastructure/22-keycloak-oidc.md#overview) · [RBAC Model](../30-django-service-template/30-django-service-template.md#rbac-model)

**PostgreSQL** — Shared database instance with per-service schemas. Initialised via startup scripts that create databases and schemas automatically. Each Django service gets logical isolation without operational overhead. → [Setup & Configuration](../20-base-infrastructure/20-postgresql.md#overview)

**MailDev** *(dev only)* — Captures all outbound email from all services. Web UI for inspecting verification emails, password resets, and notifications without sending anything externally. Replaced by a real SMTP provider in production. → [Setup & Configuration](../20-base-infrastructure/23-maildev.md#overview)

**Echo Server** *(dev only)* — Mirrors incoming requests back in the response. Used for debugging Envoy Gateway routes, headers, and path matching before wiring up real services.

**Ngrok** *(dev only)* — Exposes the local stack via a public HTTPS URL. Enables OAuth callbacks, webhook integrations, mobile testing, and client demos against the dev environment. Replaced by direct DNS + Let's Encrypt in production.

### 🧩 Django Service Template

> Updatable Copier template that generates production-ready Django services.

**Django Services** — Ships with API (DRF + HackSoft Styleguide 2+), authentication (django-allauth + Keycloak OIDC), authorization (RBAC), and core (admin panel, healthchecks) apps. Optional public app with django-htmx frontend with branding, SEO, and analytics setup. Pre-commit, linting, formatting included. Multiple services deploy on the same infrastructure under different subdomains. → [Create a Service](../30-django-service-template/30-django-service-template.md#copier-workflow)

**App internals:** → [API](../30-django-service-template/30-django-service-template.md#api) · [Authentication](../30-django-service-template/30-django-service-template.md#authentication) · [Authorization](../30-django-service-template/30-django-service-template.md#authorization) · [Core](../30-django-service-template/30-django-service-template.md#core) · [Public Frontend](../30-django-service-template/30-django-service-template.md#public-optional)

### ⎈ Kubernetes — OCI Free Tier

> Zero-cost production Kubernetes on Oracle Cloud Always Free. ARM64 only.

**OCI Infrastructure** — Terraform-provisioned OKE cluster, VCN, ARM64 node pool, load balancer, object storage. All within Always Free tier. → [Setup & Configuration](../40-kubernetes/40-oci-infrastructure.md#overview) · [Provisioning](../40-kubernetes/40-oci-infrastructure.md#provisioning)

**K8s Default Resources** — ArgoCD app-of-apps platform layer. Sealed Secrets, Envoy Gateway (K8s Gateway API), cert-manager, Keycloak, Longhorn storage, optional AI and database workloads. → [Setup & Configuration](../40-kubernetes/41-k8s-default-resources.md#overview) · [Bootstrap](../40-kubernetes/41-k8s-default-resources.md#bootstrap-sequence)
