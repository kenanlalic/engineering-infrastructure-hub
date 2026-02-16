---
## title: "Keycloak Identity Provider"
created: "2026-02-11"
updated: "2026-02-11"
version: 1.0.0
status: published
type: reference
owner: "platform-team"
lifecycle: production
tags: [service, identity, authentication, oidc]
---
# Keycloak — Centralized Identity & Access Management

Single source of truth for authentication, authorization, and user lifecycle management across all platform services. OIDC-based SSO with RBAC following NIST Core and Hierarchical principles (Levels 1–2), synced to Django services via django-allauth.

> 📍 **Type:** Service Reference
> 👤 **Owner:** Ktwenty Threel
> 🎯 **Outcome:** Understand the Keycloak OIDC Setup

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Design Decisions](#design-decisions)
- [Dependencies](#dependencies)
- [Configuration](#configuration)
- [Realm Management](#realm-management)
- [Realm Template Maintenance](#realm-template-maintenance)
- [Dev ↔ Prod Differences](#dev--prod-differences)
- [Monitoring](#monitoring)
- [Runbooks](#runbooks)
- [ADRs](#adrs)
- [References](#references)

---

## Overview

Keycloak serves as the platform's centralized identity provider. All Django services authenticate through OIDC using **django-allauth** with the Keycloak provider backend.

### Authentication Flow

The browser initiates the OIDC authorization code flow through Envoy Gateway to Keycloak. After the user authenticates, Keycloak redirects back with an authorization code. Django's `CustomSocialAccountAdapter` (via django-allauth) exchanges this code for tokens over the internal backchannel (`http://keycloak.local:8080`), never routing token exchange through the public gateway. On every login, `pre_social_login()` syncs the user profile and Keycloak group memberships from the OIDC claims into Django's local user model. For new users, django-allauth auto-creates the Django user and links the Keycloak social account. Existing Django users with a matching email are auto-linked on first Keycloak login.

Local Django login is restricted to superusers only via `AdminOnlyLocalAuthMiddleware` — regular users must authenticate through Keycloak SSO. This preserves emergency admin access when Keycloak is unavailable.

### Authorization Flow

Authorization follows a **Role-Based Access Control (RBAC)** model based on [NIST RBAC](https://csrc.nist.gov/projects/role-based-access-control) Core and Hierarchical principles (Levels 1–2). This provides users, roles, permissions, user-role assignment, and role inheritance — but does not currently implement Constrained RBAC (separation of duty enforcement) or Symmetric RBAC (role-permission review/audit). Keycloak is the source of truth for user and group management. Group memberships are embedded into OIDC tokens via a custom group membership protocol mapper on the `myclient` client. On the Django side, `keycloak_sync_groups` syncs these Keycloak groups into Django groups and emits a `user_groups_synced` signal. The authorization app catches this signal and updates Django's `is_staff` and `is_superuser` flags based on the role definitions mapped via `KEYCLOAK_GROUP_TO_ROLE`. The `RBACPermissionBackend` resolves permissions (including role inheritance) from these group-to-role mappings, and `PermissionContextMiddleware` attaches `user_roles`, `user_permissions`, and `primary_role` to every authenticated request for use in views and templates.

This decoupled approach keeps each service independently deployable — Keycloak manages *who belongs to which groups*, Django maps groups to roles and decides on *what those roles can do*.

A single realm (`myrealm`) holds all users, groups, and client configurations. The master realm remains at defaults and is used exclusively for Keycloak admin console access.

The service runs behind Envoy Gateway in both dev and prod. All auth flows (login, logout, token exchange) route through the gateway at the `/auth` path prefix.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                           INTERNET                               │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               │ HTTPS (443)
                               ▼
                    ┌─────────────────────┐
                    │    Ngrok Tunnel     │  (dev only)
                    │  Public URL → Local │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Envoy Gateway     │
                    │  ─────────────────  │
                    │  TLS Termination    │
                    │  Path-Based Routing │
                    │  X-Forwarded-*      │
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼───────────────────┐
          │                    │                   │
          ▼                    ▼                   ▼
   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │  Keycloak    │    │  Django      │    │  Django      │
   │  ─────────── │    │  Service A   │    │  Service N   │
   │  OIDC / SSO  │◄───│  allauth     │    │  allauth     │
   │  RBAC Roles  │    │  RBAC sync   │    │  RBAC sync   │
   │  User Mgmt   │    │  DRF + htmx  │    │  DRF + htmx  │
   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
          │                   │                   │
          │         ┌─────────┴───────────────────┘
          │         │
          ▼         ▼
   ┌──────────────────────┐    ┌───────────────┐
   │  PostgreSQL          │    │  MailDev      │
   │  ──────────────────  │    │  ──────────   │
   │  Shared Instance     │    │  SMTP Capture │
   │  Per-Service Schemas │    │  Web UI       │
   └──────────────────────┘    └───────────────┘
```

The `◄───` arrow between Django and Keycloak represents the internal backchannel: Django talks to Keycloak directly over the Docker bridge network (`http://keycloak.local:8080`) for token validation, user info, and OIDC discovery — never routing through the public gateway.

---

## Design Decisions

### Single Realm

| | Dev | Prod |
|---|---|---|
| **Choice** | Single realm (`myrealm`) | Single realm (`myrealm`) |

One realm keeps the setup simple to learn and operate during development. It provides a shared user pool across all services, straightforward token management, and a single place to configure clients and roles. This pattern scales well — additional services register as new clients within the same realm, and role assignments extend naturally without restructuring. Moving to multi-realm is a future option if hard tenant isolation becomes a requirement.

### RBAC Authorization (NIST Core + Hierarchical)

| | Dev | Prod |
|---|---|---|
| **Choice** | Keycloak roles → OIDC claims → Django sync | Same, with stricter role definitions |

Authorization follows [NIST RBAC](https://csrc.nist.gov/projects/role-based-access-control) at the **Core and Hierarchical levels (Levels 1–2)**: users are assigned to roles, roles carry permissions, and senior roles inherit permissions from junior roles. Constrained RBAC (Level 3 — static/dynamic separation of duty) and Symmetric RBAC (Level 4 — role-permission review and audit) are not currently implemented. Keycloak is the single source of truth for user and group management. Group memberships are embedded into OIDC tokens via a custom protocol mapper and synced into Django groups on every login via django-allauth. The authorization app maps these groups to internal roles via `KEYCLOAK_GROUP_TO_ROLE`, resolves permissions (including inheritance), and updates Django's `is_staff`/`is_superuser` flags accordingly. This decouples identity management from application logic — services consume group claims without needing to know how roles are defined or managed, making the architecture microservice-ready from day one.

### Confidential Client with Backchannel Separation

| | Dev | Prod (single-host VPS) |
|---|---|---|
| **Choice** | Confidential client + `KC_HOSTNAME_BACKCHANNEL_DYNAMIC=true` | Confidential client + explicit `KC_HOSTNAME` |

Django is a server-side application, not a SPA — it can securely store a client secret, so there's no reason to use a public client. The confidential client adds defense-in-depth: even if an authorization code is intercepted, it's useless without the secret.

**Dev environment** follows industry best practice for local Docker development: Envoy Gateway terminates TLS (via ngrok's public URL), while Django communicates with Keycloak directly over the Docker bridge network at `http://keycloak.local:8080/auth`. `KC_HOSTNAME_BACKCHANNEL_DYNAMIC=true` allows Keycloak to accept requests on the internal hostname while the browser uses the ngrok hostname — cleanly separating the public-facing path from the service-to-service backchannel. No token, secret, or credential ever leaves the host machine.

**Production on a single-host VPS** maintains the same proven architecture: Envoy Gateway terminates public TLS using Let's Encrypt certificates, and all internal communication (both Envoy → Keycloak and Django → Keycloak) runs plain HTTP over the Docker bridge network. This is the standard and correct pattern for single-host deployments — the Docker bridge is a private, host-local network with no external exposure. Production hardens this with `KC_HOSTNAME=https://${DOMAIN}/auth` for strict issuer validation, replacing dev's permissive `--hostname-strict=false`.

**Scaling consideration (multi-node / k8s):** The direct backchannel carries the client secret and returns access/refresh tokens — the most sensitive exchange in the OIDC flow. On a single host this is inherently protected. In a multi-node Kubernetes deployment where pods may reside on different nodes, this traffic would cross the cluster network unencrypted. At that point, backchannel encryption becomes a requirement — via service mesh with automatic mTLS (Istio, Linkerd) or Keycloak-native TLS on the backchannel listener.

### SMTP

| | Dev | Prod |
|---|---|---|
| **Choice** | MailDev (catches all emails locally) | Real SMTP provider |

MailDev provides a local web UI to inspect verification and password reset emails without sending anything externally. Production uses a properly configured SMTP relay.

### Realm Import Strategy

| | Dev | Prod |
|---|---|---|
| **Choice** | `--import-realm` at startup | `keycloak-config-cli` or Terraform |

Three approaches exist for provisioning Keycloak realms, ranked by complexity:

1. **`--import-realm` at startup** — Keycloak reads `.json` files from `/opt/keycloak/data/import/` on boot. Creates realms that don't exist, silently skips those that do. Zero update capability — changes require `docker compose down -v`. This is our dev choice for simplicity. [Official docs](https://www.keycloak.org/server/importExport)

2. **Admin REST API sidecar** — A separate container waits for Keycloak health, then hits the Admin API to create/update realms, clients, users independently. Full CRUD control but more compose complexity. [Admin REST API docs](https://www.keycloak.org/docs-api/latest/rest-api/index.html)

3. **`keycloak-config-cli` (adorsys)** — Declarative, idempotent configuration-as-code via the Admin API. Tracks diffs, supports variable substitution, runs as a sidecar. The production-grade choice when incremental updates matter. [GitHub](https://github.com/adorsys/keycloak-config-cli)

We use option 1 for dev because it requires zero additional services and our realm template is version-controlled. For production, option 3 is optional path.

---

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| PostgreSQL | Database | Keycloak persistence (schema: `keycloak`) |
| Envoy Gateway | Proxy | TLS termination, path routing, X-Forwarded-* |
| MailDev | Dev tool | Catches email verification/reset emails |

---

## Configuration

### Compose Environment Variables

| Variable | Default | Description |
|---|---|---|
| `KC_BOOTSTRAP_ADMIN_USERNAME` | `admin` | Admin console username. Replaces the deprecated `KEYCLOAK_ADMIN` — the old name still works in 26.0.x but will be removed. |
| `KC_BOOTSTRAP_ADMIN_PASSWORD` | `admin` | Admin console password. Same deprecation applies. Set once on first boot; changing it later has no effect unless the database is wiped. |
| `KC_DB_URL` | — | JDBC PostgreSQL connection string (`jdbc:postgresql://db:5432/keycloak`). Uses the internal Docker network hostname, not `localhost`. |
| `KC_DB_USERNAME` | `postgres` | Database user. Shared with other services via the main `.env` to keep credentials in one place. |
| `KC_DB_PASSWORD` | — | Database password. Pulled from the main `.env` (`POSTGRES_PASSWORD`). Never hardcoded in compose. |

### Realm JSON Environment Variables

These are substituted by Keycloak at import time into `myrealm.json` using `${VAR}` syntax. This allows the same realm template to work across different developer machines or CI environments without modifying the committed file.

| Variable | Default | Description |
|---|---|---|
| `DEV_ADMIN_PASSWORD` | `admin` | Password for the `superuser` dev-admin user. Separated from other users so admins can use a different password if needed. |
| `DEV_USER_PASSWORD` | `password` | Default password for all non-admin dev users. Keeps the template DRY. |
| `KEYCLOAK_CLIENT_SECRET` | `my-client-secret` | OIDC client secret for `myclient`. Must match what Django's `.env` expects — a mismatch is the most common cause of login failures after reimport. |

### Startup Flags

| Flag | Purpose |
|---|---|
| `--http-enabled=true` | Allows HTTP connections. Safe because TLS is terminated at Envoy Gateway, not at Keycloak. In production, Keycloak sits behind the gateway and never receives external traffic directly. |
| `--proxy-headers=xforwarded` | Trusts `X-Forwarded-*` headers from Envoy for correct redirect URIs and HTTPS detection. Without this, Keycloak would generate HTTP callback URLs even though the client sees HTTPS. |
| `--http-relative-path=/auth` | Serves Keycloak at the `/auth` prefix. Matches Envoy's path-based routing rules so all auth endpoints live under a single path. |
| `--hostname-strict=false` | Accepts any hostname (dev only). In production, replace with `KC_HOSTNAME=https://${DOMAIN}/auth` to lock down the issuer URL. |
| `--import-realm` | Auto-imports realm JSON on first start from `/opt/keycloak/data/import/`. Silently skips if the realm already exists. See [Design Decisions → Realm Import Strategy](#realm-import-strategy) for alternatives. |
| `--health-enabled=true` | Enables `/health/ready` on the management port (9000, not 8080). Required for compose health checks and monitoring. |

---

## Realm Management

### How Import Works

On first `docker compose up`, Keycloak reads `myrealm.json` from the mounted import directory, creates the realm with all roles, clients, and users. On every subsequent start, the realm already exists and the import is silently skipped — no updates are applied, no data is overwritten.

To apply changes to the realm configuration:

```bash
docker compose down -v && docker compose up -d
```

This destroys the database volume and reimports from scratch. Any runtime changes (users created via the admin console, password resets, session data) are lost.

### Realm Template Best Practices

**Keep the template minimal.** Only include what you've customized. Keycloak auto-creates default clients (`account`, `admin-cli`, `broker`, `realm-management`, `security-admin-console`), default client scopes, default auth flows, and key providers. Including these adds thousands of lines of noise and creates merge conflicts for no benefit.

**Use plaintext passwords.** Write `"credentials": [{"type": "password", "value": "mypass"}]` — Keycloak hashes automatically on import. Exported hashed credentials are unreadable, version-fragile, and defeat the purpose of a human-maintainable template.

**Use `${ENV_VAR}` substitution** for any value that differs across environments or developer machines. Keycloak resolves these from environment variables during import. Fixed in 26.0.1 (was broken in 26.0.0, issue #33578).

**Name the file `<realm>-realm.json`.** A known Keycloak 26.x bug with hyphenated filenames that don't follow this convention can cause "Session not bound" errors (issue #36284).

**One file per realm, users inline.** The multi-file split (`<realm>-users-0.json`) exists for exports with thousands of users. For a hand-crafted dev template, keep everything in one file.

**No master realm file.** `--import-realm` cannot import the master realm — it already exists before import runs. Configure master via `KC_BOOTSTRAP_ADMIN_*` env vars.

---

## Realm Template Maintenance

The realm JSON is a hand-crafted, version-controlled template — not an export artifact. Treat it like infrastructure-as-code: changes go through pull requests, and the template stays minimal.

**When to edit the template directly:** Adding or removing dev users, changing client configuration, updating role definitions, adjusting redirect URIs. These are all changes to the *desired state* of the dev environment.

**When to export from the admin console:** If you've prototyped a complex change (auth flow, protocol mapper, identity provider) in the UI and need to capture the exact JSON structure. Use `kc.sh export` (never the admin console's partial export — it masks secrets and omits users), then cherry-pick only the relevant section into your template. Don't paste the full export.

```bash
# Export for reference (not for direct use as template)
docker compose exec keycloak /opt/keycloak/bin/kc.sh export \
  --dir /opt/keycloak/data/export \
  --realm myrealm \
  --users realm_file
```

> **Note:** `kc.sh export` may fail with "Address already in use" on a live server. Run it in a separate container sharing the same database, or stop the server first.

---

## Dev ↔ Prod Differences

| Aspect             | Dev                                           | Prod                                                                       |
| ------------------ | --------------------------------------------- | -------------------------------------------------------------------------- |
| Realm provisioning | `--import-realm` at startup                   | `--import-realm` at startup (optional: `keycloak-config-cli` or Terraform) |
| Client secrets     | Hardcoded / env var in `.env`    | External secret manager (Vault, k8s secrets)                               |
| Hostname           | `--hostname-strict=false`                     | `KC_HOSTNAME=https://${DOMAIN}/auth`                                       |
| Backchannel        | `KC_HOSTNAME_BACKCHANNEL_DYNAMIC=true`        | `KC_HOSTNAME_BACKCHANNEL_DYNAMIC=true` (revisit for multi-node k8s)        |
| User passwords     | Plaintext in realm JSON                       | No users in JSON — self-registration or federated                          |
| Image              | Stock `quay.io/keycloak/keycloak:26.0.4`      | Custom image with `--optimized` build                                      |
| Admin credentials  | `KC_BOOTSTRAP_ADMIN_*` in `.env` | Injected from secret manager, rotated                                      |
| Health endpoint    | Port 9000, bash TCP redirect                  | Port 9000, wired to k8s liveness/readiness probes                          |
| SMTP               | MailDev                                       | Real SMTP provider                                                         |
| TLS                | Let's Encrypt via ngrok                       | Let's Encrypt certs at Envoy Gateway                                       |
| Internal transport | Plain HTTP over Docker bridge (single host)   | Plain HTTP over Docker bridge (single host, revisit for multi-node k8s)    |

> **Optional:** prod optimization possible with `--optimized` flag for builds. The above assumes the standard production recommendation. Adjust if your prod Dockerfile differs.

---

## Monitoring

### Dev

- **Logs:** `docker compose logs -f keycloak` — set `KC_LOG_LEVEL=INFO` (or `DEBUG` for troubleshooting auth flows)
- **Admin Console:** Visual inspection of sessions, events, and login errors at `https://${DOMAIN}/auth/admin/`
- **MailDev:** Inspect verification and reset emails at the MailDev web UI

### Prod

- **Healthcheck:** `GET :9000/health/ready` wired to k8s liveness and readiness probes
- **Metrics:** Enable with `KC_METRICS_ENABLED=true`, scrape Prometheus metrics from `:9000/metrics`
- **Key metrics:** Login success/failure rate, token issuance rate, active sessions, cache hit ratios
- **Dashboards:** Grafana dashboard (link TBD)
- **Alerts:** Alert rules for failed logins spike, health endpoint down, DB connection pool exhaustion (configuration TBD)

> **Optinonal:** prod monitoring stack improvement. The above assumes Prometheus + Grafana which is standard for Keycloak. Adjust dashboard and alert links once provisioned.

---

## Runbooks

### Realm won't import on restart

**Expected behavior.** `--import-realm` skips existing realms. To force reimport: `docker compose down -v && docker compose up -d`. This destroys the database volume.

### Keycloak fails to start — "Address already in use"

Another container or previous instance is holding port 8080. Check with `docker ps -a | grep keycloak` and kill orphans with `docker compose down --remove-orphans`.

### Client secret mismatch after reimport

Verify `KEYCLOAK_CLIENT_SECRET` in `.env` matches what Django's `.env` expects. After reimport, the value from the realm JSON (with env var substitution) is what Keycloak uses.

### Dev user can't log in

Check the user exists in Admin Console → Users. If the realm was reimported, all runtime changes (password resets, users created via UI) are lost. Only users defined in the realm JSON survive reimport.

### Django allauth OIDC sync fails

Verify Keycloak is reachable internally at `http://keycloak.local:8080/auth`. Check that `KC_HOSTNAME_BACKCHANNEL_DYNAMIC=true` is set. Confirm the client secret matches on both sides. Check Django logs for `SocialLogin` errors — a 401 from Keycloak usually means a secret mismatch, a connection refused means the internal hostname isn't resolving.

---

## ADRs

- **ADR: Single realm for all services** — Shared user pool, simplified token management, scales by adding clients within the realm
- **ADR: NIST RBAC (Core + Hierarchical) over flat group-based auth** — Keycloak as source of truth for user-role assignments, synced to services via OIDC claims, proper decoupling for microservice architecture. Constrained and Symmetric RBAC levels remain future options.
- **ADR: Confidential client with backchannel separation** — Server-side Django app stores secret securely; dynamic backchannel allows internal/external URL separation without hostname mismatch errors

---

## References

- [Keycloak Server Admin Guide](https://www.keycloak.org/docs/latest/server_admin/)
- [Keycloak Import/Export Documentation](https://www.keycloak.org/server/importExport)
- [Keycloak Admin REST API](https://www.keycloak.org/docs-api/latest/rest-api/index.html)
- [keycloak-config-cli (adorsys)](https://github.com/adorsys/keycloak-config-cli)
- [NIST RBAC Project](https://csrc.nist.gov/projects/role-based-access-control)
- [django-allauth Keycloak Provider](https://docs.allauth.org/en/latest/socialaccount/providers/keycloak.html)
- [[infrastructure-engineering-hub|Infrastructure Engineering Hub]]
- [[service-registry|Service Registry]]