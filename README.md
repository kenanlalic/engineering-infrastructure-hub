# Engineering Infrastructure Hub

Production-ready infrastructure layer for multi-service platforms. Envoy Gateway (Kubernetes Gateway API in standalone mode), Keycloak OIDC, and PostgreSQL — running in Docker Compose with a clear scaling path to Kubernetes. Same configuration, same routing, same auth flow at every tier. New services plug in with a single compose include and two YAML files, regardless of framework or language.

```
Internet
  │
  │  HTTPS (443) / HTTP (80 → 301)
  ▼
┌─────────────────────┐
│   Envoy Gateway     │  TLS termination, path-based routing
│   (Gateway API)     │  Kubernetes-native configs in standalone mode
└──────────┬──────────┘
           │  HTTP (internal Docker bridge)
     ┌─────┼─────┐
     ▼     ▼     ▼
 Keycloak  Svc A  Svc N      PostgreSQL (shared instance, per-service DBs)
 (OIDC)    :8000  :8000      MailDev (dev email capture)
```

### Django Service Template

Ships with a Copier-generated Django service template that produces a complete, working application — not a bare scaffold. Five apps, from public-facing landing page to API layer, ready before a developer writes their first line of business logic:

| App | What ships out of the box |
|---|---|
| **public** | Marketing landing page — hero section, offerings, value propositions, contact. Responsive layout with CSS animations, i18n-ready. |
| **authentication** | Login, registration, password reset, profile management, email management — all wired to Keycloak OIDC via django-allauth. |
| **authorization** | RBAC group synchronization from Keycloak, permission model, role-based access control. |
| **core** | Internal dashboard — HTMX live-polling system status, user profile, group display, staff admin access, health checks. |
| **api** | DRF API layer structured around the [HackSoft Django Styleguide 2+](https://github.com/HackSoftware/Django-Styleguide) — services for business logic, selectors for queries, thin views. DDD-leaning architecture that works with Django rather than against it. |

The entire frontend layer — public site, auth pages, internal dashboard — is built with django-htmx, vanilla JS, and CSS. No Node toolchain, no bundler, no `node_modules` security surface, and still capable of reactive UI patterns out of the box.

Every piece is exchangeable. Swap Django for FastAPI, Go, or anything that speaks HTTP and OIDC.

### Developer Experience

The entire development environment runs in containers. Docker and VS Code are the only prerequisites — no local Python, no virtual environments, no system-level dependencies.

The VS Code multi-root workspace gives each service its own Git repository and history while presenting a single window with a centralized debugger launch configuration. Breakpoints, step-through, and variable inspection work inside running containers — the same image, the same OS, the same dependencies that run in production. Tests execute inside containers with parallel runners, eliminating "passes locally, fails in CI" drift. A new developer clones, opens the workspace, and hits `F5`.

See [Workspace Config](docs/10-workspace-config/10-workspace-config.md) for the full setup, debugger configuration, and onboarding guide.

### Scaling Path

The architecture is designed around three tiers that share the same codebase and configuration patterns:

| Tier | Runtime | What Changes | What Stays |
|---|---|---|---|
| **Local** | Docker Compose, Ngrok tunnel | Ephemeral volumes, default tuning, self-signed certs | Envoy Gateway API YAML, compose definitions, auth flow |
| **VPS** | Docker Compose, single server | Persistent volumes, production tuning, Let's Encrypt TLS | Same compose files, same Envoy routes, same Keycloak realm |
| **Kubernetes** | Managed cluster (EKS, GKE, AKS) | RDS/CloudSQL, cert-manager, Envoy Gateway operator, Helm | Same Gateway API YAML — `kubectl apply` directly, no rewrite |

Each tier is a deployment target, not a rewrite. The Envoy Gateway API resources that define routing in Docker Compose are the same files applied to a Kubernetes cluster. Keycloak realm exports, database schemas, and service definitions carry forward. Moving between tiers is an operational change, not an architectural one.

---

## Documentation

The `docs/` directory is an [Obsidian](https://obsidian.md/) vault. Open it in Obsidian for full navigation, graph view, and callout rendering. GitHub renders the markdown partially (wiki-links display as plain text, callouts render as blockquotes).

```
docs/
├── 00-index/
│   └── 01-infrastructure-engineering-hub.md    ← Start here
├── 10-workspace-config/
│   └── 10-workspace-config.md
├── 20-base-infrastructure/
│   ├── 20-postgresql.md
│   ├── 21-envoy-gateway.md
│   ├── 22-keycloak-oidc.md
│   └── 23-maildev.md
└── 30-django-service-template/
    └── 30-django-service-template.md
```

**Highlights:**

- [Infrastructure Engineering Hub](docs/00-index/01-infrastructure-engineering-hub.md) — Platform architecture, design philosophy, three-tier scaling path, service catalog
- [Envoy Gateway](docs/20-base-infrastructure/21-envoy-gateway.md) — Kubernetes Gateway API resources running in Docker Compose standalone mode. Same YAML applies to both environments.
- [Keycloak OIDC](docs/20-base-infrastructure/22-keycloak-oidc.md) — Confidential client with backchannel token exchange, RBAC group sync, realm-as-code
- [Workspace Config](docs/10-workspace-config/10-workspace-config.md) — Full setup guide, debugger configuration, multi-root workspace reference

---

## Getting Started

This repo is the workspace shell. Infrastructure and services are separate repositories cloned into this directory.

```bash
git clone git@github.com:yourorg/setup-workspace.git
cd setup-workspace
git clone git@github.com:yourorg/basic-infrastructure.git
git clone git@github.com:yourorg/django-service-template.git

cd basic-infrastructure
cp .env.example .env
docker compose up -d

code eih.code-workspace
```

See [Workspace Config](docs/10-workspace-config/10-workspace-config.md) for prerequisites, extension setup, container-based debugging, and the full service generation workflow.

---

## License

Licensed under [Apache 2.0](LICENSE). See LICENSE file for details.
