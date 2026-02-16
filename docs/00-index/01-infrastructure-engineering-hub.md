---
title: Infrastructure Engineering Hub
created: 2025-02-10
updated: 2026-02-13
version: 3.0.0
status: published
type: moc
owner: "platform-team"

tags:
- moc
- home
- infrastructure
- django
- docker
- keycloak
- envoy
---
# Infrastructure Engineering Hub

**Scalable open-source infrastructure for microservices — from local development through VPS production to Kubernetes with minimal friction.**

> 📍 **Type:** Map of Content (MOC)
> 👤 **Owner:** Ktwenty Threel
> 🎯 **Outcome:** Understand the platform, the architecture, and where everything lives

## Table of Contents

1. [[#Platform at a Glance]]
2. [[#Architecture]]
3. [[#Scaling Path]]
4. [[#Stack Reference]]

---

## Platform at a Glance

A scalable open-source infrastructure for microservices — from local development through VPS production to Kubernetes — with minimal configuration changes between tiers. All components are open-source and interchangeable. The local development experience is first-class, and the same containers that run on your machine deploy to production without modification.

The platform follows a natural progression through three layers:

> [!info] Logical flow
> **Workspace → Infrastructure → Services → Dev → Prod VPS → Prod K8s** Every section in this document follows this progression.

### 🗂️ VS Code Workspace Setup

A multi-root workspace configuration that manages all services, infrastructure, and this documentation in a single VS Code instance. Each service gets its own debugger launch configuration. The VS Code Test Explorer runs inside a dev container with parallelized per-test debugging capabilities.

- [[1.0 - Workspace Config]]
- [[launch-json-debuggers|Debugger Setup]]

### 🏗️ Base Infrastructure

All services run behind **Envoy Gateway**, which handles TLS termination, path-based routing, and header manipulation. A `generate_certs` sidecar provisions internal certificates. **Keycloak** provides OIDC/SSO with a pre-configured realm template for development. **PostgreSQL** runs as a shared instance with per-service schemas, initialized via startup scripts. **MailDev** (dev only) captures all outbound email for inspection. An **echo server** (dev only) mirrors requests for gateway debugging.

Pre-configured configs ship for Envoy Gateway (dev and prod-ready for VPS, minimal changes for K8s), Keycloak realm import (dev), PostgreSQL database initialization scripts, and Envoy secrets generation scripts.

In development, **Ngrok** exposes the entire local stack via a public HTTPS URL — enabling OAuth callbacks, webhook integrations, mobile testing, and client demos against your actual environment. In production on VPS, Ngrok is replaced by direct public exposure through Envoy with Let's Encrypt certificates.

- [[2.0 - PostgreSQL]]
- [[2.1 - Envoy Gateway]]
- [[2.2 - Keycloak OIDC]]
- [[2.3 - MailDev]]

### 🧩 Django Service Template

New services are generated via an updatable [[template-overview|Copier template]] in a few steps. Each service ships with four apps following [HackSoft Styleguide 2+](https://github.com/HackSoftware/Django-Styleguide) best practices: API (Django REST Framework), authentication (django-allauth + Keycloak OIDC), authorization (RBAC with Keycloak as source of truth), and core (admin panel, healthchecks). An optional public app provides a django-htmx powered frontend with branding, SEO, and analytics setup. Pre-commit hooks, linting, and formatting are included.

Multiple Django services can be generated from the same template and deployed on the same infrastructure under different subdomains — each service registers as a new Envoy route and Keycloak client within the existing setup.

- [[3.0 - Django Service Template]]
- [[creating-new-service|Create a New Service]]
- [[template-customization|Template Customization]]

> [!tip] Ready to start? 
> See the [[workspace-setup-readme|Workspace Setup README]] for step-by-step setup.

---

## Architecture

Instead of each service managing its own authentication, routing, and database, these concerns are centralized into shared infrastructure.

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
   │  Per-service debuggers · Parallelized test runner            │
   │  File exclusions · Shared settings                           │
   └──────────────────────────────────────────────────────────────┘
```

The `◄───` arrow between Django Service A and Keycloak represents the internal backchannel: Django exchanges tokens with Keycloak directly over the Docker bridge network, never routing through the public gateway. See [[service-keycloak#Confidential Client with Backchannel Separation|Backchannel Separation]] for details.

### Component Responsibilities

|Component|What It Does|Swap It For|
|---|---|---|
|**Envoy Gateway**|Routes traffic by path/subdomain, terminates TLS (Let's Encrypt on prod, Ngrok on dev), rate limiting, HTTP→HTTPS redirect|Traefik, NGINX, Kong|
|**Keycloak**|SSO, OIDC, user management, RBAC source of truth, synced to services via django-allauth|Any OIDC provider|
|**PostgreSQL**|Shared database instance, per-service schemas, initialized via startup scripts|Any SQL backend|
|**MailDev** _(dev only)_|Captures all outbound email, web UI for inspection|Mailhog, Mailtrap, any SMTP provider|
|**Echo** _(dev only)_|Mirrors requests back for gateway route debugging|Any echo server|
|**Ngrok** _(dev only)_|Exposes local stack via public HTTPS URL for OAuth callbacks, webhooks, mobile testing|Cloudflare Tunnel, localtunnel — replaced by direct DNS in prod|
|**Django Services**|Generated from updatable Copier template, each with API, auth, RBAC, admin, optional htmx frontend|—|
|**VS Code Workspace**|Multi-root config with per-service debuggers and parallelized test execution|JetBrains (manual setup)|

---

## Scaling Path

> [!tip] Same images, different orchestration
> Your application code and Docker images never change between tiers. Only the orchestration layer, secrets management, and infrastructure configuration evolve.

---

### 🛠️ Local Development — Docker Compose

Your laptop runs the full stack. Envoy routes, Keycloak authenticates, PostgreSQL stores, MailDev captures email, VS Code debugs. Ngrok exposes everything via a public HTTPS URL — OAuth callbacks, webhook integrations, mobile testing, client demos all work against your actual dev environment. A separate `compose.yaml` wires up dev-only services (MailDev, echo server) and relaxed configurations (permissive hostnames, plaintext secrets, hot-reload).

---

### 🚀 MVP Production — Docker Compose on VPS

Deploy `compose.prod.yaml` to a single VPS (Hostinger, Kamatera, etc.). Dev-only services are gone. Let's Encrypt replaces Ngrok for public TLS. Keycloak locks down its hostname and issuer. Secrets come from environment files outside the repo. The same Envoy Gateway configs that ran locally now serve production traffic — no rewrite, no translation layer. This tier carries most startups through their first real scaling event.

---

### ⚡ Production at Scale — Kubernetes

Each container becomes a pod. Envoy Gateway configs translate directly to Kubernetes Gateway API resources. PostgreSQL moves to a managed service (RDS, CloudSQL). Keycloak scales horizontally. Backchannel encryption becomes a requirement — via service mesh mTLS or Keycloak-native TLS. The application images stay identical; only the orchestration and persistence layers change.

---

### What Changes Between Tiers

|Component|🛠️ Local Dev|🚀 VPS Prod|⚡ K8s Prod|
|---|---|---|---|
|**Compose file**|`compose.yaml`|`compose.prod.yaml`|Helm charts / manifests|
|**Public access**|Ngrok tunnel|Direct DNS + Let's Encrypt|Ingress / Gateway API|
|**TLS**|Ngrok handles it|Let's Encrypt at Envoy|Cert-manager + Ingress|
|**Envoy Gateway**|File provider, relaxed|File provider, prod-ready|K8s Gateway API (native)|
|**Keycloak hostname**|`--hostname-strict=false`|`KC_HOSTNAME=https://${DOMAIN}/auth`|Same, or dedicated auth subdomain|
|**Keycloak realm**|`--import-realm` from JSON|`--import-realm` (planned: config-cli)|`keycloak-config-cli` or Terraform|
|**Backchannel**|Plain HTTP, Docker bridge|Plain HTTP, Docker bridge|Encrypted (mesh mTLS or native TLS)|
|**Secrets**|`.env` files, plaintext|`.env` files outside repo|K8s Secrets / Vault|
|**PostgreSQL**|Shared container|Shared container + persistent volume|Managed service (RDS, CloudSQL)|
|**Email**|MailDev (captures all)|Real SMTP provider|Real SMTP provider|
|**Echo server**|✅ Running|—|—|
|**Monitoring**|Docker logs + Admin Console|Docker logs + Admin Console|Prometheus + Grafana|
|**Scaling**|Single instance|Single instance|Horizontal pod autoscaling|
|**Deploy method**|`docker compose up`|`docker compose up`|CI/CD pipeline|

---

## Stack Reference

> [!info] Technologies listed by platform layer
> Each layer corresponds to a separate repository. See [[#Platform at a Glance]] for how these layers connect.

---

### 🗂️ Workspace Setup

> Configuration and tooling for the multi-service development experience.

**VS Code Workspace** — Multi-root workspace managing all services, infrastructure, and documentation. Per-service debugger launch configs and parallelized test execution via dev containers. → [[vscode-workspace-config|Configuration]] · [[launch-json-debuggers|Debuggers]] · [[settings-json-exclusions|Settings Reference]]

---

### 🏗️ Base Infrastructure

> Shared services that every Django service depends on. Run with `docker compose up`.

**Envoy Gateway** — Front door for all traffic. TLS termination (Let's Encrypt on prod, Ngrok on dev), path-based and subdomain routing, HTTP→HTTPS redirect, header manipulation. Configs are file-based, prod-ready for VPS, and translate to Kubernetes Gateway API with minimal changes. → [[service-envoy-gateway|Setup & Configuration]] · [[networking-and-ports|Networking Reference]]

**Keycloak** — Centralized identity provider. OIDC/SSO authentication, RBAC group management, user lifecycle. Single realm (`myrealm`), synced to Django services via django-allauth on every login. Admin console for user and session management. → [[service-keycloak|Setup & Configuration]] · [[rbac-permissions-model|RBAC Model]]

**PostgreSQL** — Shared database instance with per-service schemas. Initialized via startup scripts that create databases and schemas automatically. Each Django service gets logical isolation without operational overhead. → [[service-postgres|Setup & Configuration]] · [[database-schemas|Schema Reference]]

**MailDev** _(dev only)_ — Captures all outbound email from all services. Web UI for inspecting verification emails, password resets, and notifications without sending anything externally. Replaced by a real SMTP provider in production. → [[service-maildev|Setup & Configuration]]

**Echo Server** _(dev only)_ — Mirrors incoming requests back in the response. Used for debugging Envoy Gateway routes, headers, and path matching before wiring up real services.

**Ngrok** _(dev only)_ — Exposes the local stack via a public HTTPS URL. Enables OAuth callbacks, webhook integrations, mobile testing, and client demos against the dev environment. Replaced by direct DNS + Let's Encrypt in production.

---

### 🧩 Django Service Template

> Updatable Copier template that generates production-ready Django services.

**Django Services** — Ships with API (DRF + HackSoft Styleguide 2+), authentication (django-allauth + Keycloak OIDC), authorization (RBAC), and core (admin panel, healthchecks) apps. Optional public app with django-htmx frontend with branding, SEO, and analytics setup. Pre-commit, linting, formatting included. Multiple services deploy on the same infrastructure under different subdomains. → [[creating-new-service|Create a Service]] · [[template-overview|Template Overview]] · [[template-customization|Customization]]

**App internals:** → [[app-api-hacksoft-integration|API]] · [[app-authentication-allauth-oidc|Authentication]] · [[app-authorisation-rbac|Authorization]] · [[app-core-admin-panel|Core]] · [[app-public-htmx-frontend|Public Frontend]]

---

### 📋 Operations

> Procedures, workflows, and troubleshooting across all platform components.

→ [[operations-index|Operations Index]]

---

### 📚 Reference

> Specifications, schemas, conventions, and the standards that shaped this platform.

→ [[reference-index|Reference Index]]