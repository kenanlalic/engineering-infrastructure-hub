---
title: "Envoy Gateway"
created: "2026-02-10"
updated: "2026-05-14"
version: 1.4.0
type: reference
owner: Kenan Lalic
lifecycle: production
tags: [service, envoy, gateway, tls, routing, proxy]
---

# Envoy Gateway — TLS Termination & Traffic Routing

Front door for all platform traffic. Handles TLS termination, path-based and subdomain routing, HTTP→HTTPS redirect, and header manipulation. Uses Kubernetes Gateway API resources in standalone file mode — the same configs work in Docker Compose and translate to Kubernetes with minimal changes.

> 📍 **Type:** Service Reference<br>
> 👤 **Owner:** Kenan Lalic<br>
> 🎯 **Outcome:** Understand the Envoy Gateway Setup<br>

---

## Table of Contents

- [[#Overview]]
- [[#Deployment Tiers]]
- [[#Architecture]]
- [[#Design Decisions]]
- [[#Dependencies]]
- [[#Configuration]]
- [[#Certificate Management]]
- [[#Dev vs Prod Differences]]
- [[#Monitoring]]
- [[#Runbooks]]
- [[#ADRs]]
- [[#References]]

---

## Overview

Envoy Gateway is the single entry point for all external traffic. No backend service is directly exposed to the internet — every request passes through Envoy first.

### Request Flow

All external traffic arrives at Envoy Gateway on ports 80 (HTTP) and 443 (HTTPS). HTTP requests are immediately redirected to HTTPS via a 301. Envoy terminates TLS, inspects the path, and routes to the appropriate backend service over the internal Docker bridge network using plain HTTP. `X-Forwarded-Proto`, `X-Forwarded-Host`, and `X-Forwarded-Port` headers are set on every proxied request so backend services can correctly generate URLs and detect HTTPS.

In development, Ngrok provides the public HTTPS URL that reaches Envoy. In production on VPS, DNS points directly to the server and Envoy terminates TLS using Let's Encrypt certificates.

### Routing Model

Routes are defined as Kubernetes Gateway API resources (`HTTPRoute`, `Gateway`, `Backend`) stored as YAML files. Envoy Gateway reads these from the filesystem in standalone mode — no Kubernetes cluster required. Each Django service registers as a new `HTTPRoute` and `Backend` within the existing configuration.

| Path Prefix | Backend | Purpose |
|---|---|---|
| `/auth/realms`, `/auth/admin`, `/auth/resources`, `/auth/js` | Keycloak (`keycloak.local:8080`) | Identity provider — login, admin console, static assets |
| `/{service}/` | Django Service (`{service}.local:8000`) | Application routes — API, admin, frontend |

Additional services are added by creating a new `HTTPRoute` YAML file in the routes directory and a corresponding `Backend` definition. Envoy Gateway picks up changes on restart.

---

## Deployment Tiers

The same Envoy Gateway container and Gateway API YAML configs run across all three tiers. Only TLS handling, cert management, and orchestration change.

| | 🛠️ Local Dev | 🚀 VPS Prod | ⚡ Kubernetes |
|---|---|---|---|
| **Envoy runs as** | Docker container | Docker container | Pod (via Helm operator) |
| **Config format** | Gateway API YAML files | Same files | Same files — `kubectl apply` |
| **TLS** | Ngrok terminates it before Envoy | Let's Encrypt, terminated at Envoy | cert-manager issues certs, Envoy terminates |
| **Cert management** | None — Ngrok handles it | certbot on host + deploy hook | cert-manager `Certificate` resource |
| **Control plane certs** | `generate_certs` sidecar (self-signed) | Same | Envoy Gateway operator (Helm) |
| **Port 80/443** | Mapped from container | Bound directly on host | Ingress / LoadBalancer Service |
| **Scaling** | Single instance | Single instance | HPA across pods |

**Why the same configs work everywhere:** Envoy Gateway uses the Kubernetes Gateway API spec as its config format — even in Docker standalone mode it reads `HTTPRoute`, `Gateway`, and `Backend` YAML from the filesystem. In Kubernetes, those same files are applied via `kubectl apply`. The only things that change are `Backend` hostnames (Docker FQDN → K8s Service name) and how TLS secrets are provisioned.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                           INTERNET                               │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               │ HTTPS (443) / HTTP (80 → 301)
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
                               │ HTTP (internal)
          ┌────────────────────┼───────────────────┐
          │                    │                   │
          ▼                    ▼                   ▼
   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │  Keycloak    │    │  Django      │    │  Django      │
   │  :8080       │    │  Service A   │    │  Service N   │
   │              │    │  :8000       │    │  :8000       │
   └──────────────┘    └──────────────┘    └──────────────┘
```

Envoy is the only container on both the `internal` and `external` Docker networks. All backend services sit on the `internal` network only — they are unreachable from the internet. Traffic between Envoy and backends is plain HTTP over the Docker bridge, which is safe on a single host (see [[service-keycloak#Confidential Client with Backchannel Separation|Backchannel Separation]] for the security rationale).

For the full platform architecture, see [[infrastructure-engineering-hub#Architecture|Infrastructure Engineering Hub — Architecture]].

---

## Design Decisions

### Envoy Gateway over Traditional Reverse Proxies

| | Envoy Gateway | Traditional (Nginx, Traefik) |
|---|---|---|
| **Choice** | Gateway API resources in standalone file mode | Configuration files with custom syntax |

Envoy Gateway was chosen for Kubernetes-native portability. It uses the Kubernetes [Gateway API](https://gateway-api.sigs.k8s.io/) specification — `HTTPRoute`, `Gateway`, `GatewayClass`, `Backend` — as its configuration format, even when running outside Kubernetes. In Docker Compose standalone mode, these same YAML files are read from the filesystem. In Kubernetes, they're applied directly to the API server with `kubectl apply`. No rewrite, no translation.

| Capability | Nginx | Envoy Gateway |
|---|---|---|
| Config format | Custom `nginx.conf` syntax | Kubernetes Gateway API YAML |
| Routing changes | Requires reload | Hot reload via control plane |
| K8s migration | Full config rewrite | Copy configs as-is |
| Observability | Basic access logs | Prometheus metrics, structured access logs |
| Protocol support | HTTP/1.1, HTTP/2 | HTTP/1.1, HTTP/2, gRPC, WebSocket |
| TLS management | Manual cert handling | Gateway API `Secret` references |

### TLS Strategy

| | Dev | Prod (VPS) |
|---|---|---|
| **Choice** | Ngrok handles TLS | Let's Encrypt certs at Envoy |

**Dev:** Ngrok provides the public HTTPS URL and terminates TLS before traffic reaches Envoy. Envoy still runs with TLS configured on the HTTPS listener, but in practice Ngrok handles the public certificate. No local cert management required.

**Prod:** Let's Encrypt certificates are obtained via `certbot` on the host and mounted read-only into the Envoy container. The Gateway resource references these via a `Secret` that points to the mounted PEM files. Certificate renewal is fully automated via systemd timer and a certbot deploy hook — certbot renews, the hook regenerates secrets and restarts Envoy Gateway. See [[#Certificate Management]] for the full setup.

```
Host filesystem                    Docker container
─────────────────                  ─────────────────
/etc/letsencrypt/                  
├── live/${DOMAIN}/                (read-only mount)
│   ├── fullchain.pem ──symlink──→ Envoy reads here
│   └── privkey.pem ──symlink───→ Envoy reads here
└── archive/${DOMAIN}/             
    ├── fullchain1.pem             (actual files)
    └── privkey1.pem               
```

Certbot manages symlink rotation on renewal. Envoy always reads from `/live/` — no path changes needed.

### Control Plane Certificates (`generate_certs`)

| | Dev | Prod |
|---|---|---|
| **Choice** | Self-signed certs via `generate_certs` sidecar | Same |

Envoy Gateway's architecture consists of two processes: a **controller** (reads Gateway API YAML, computes routing configuration) and an **Envoy proxy** (handles actual traffic). These communicate via gRPC using the xDS protocol. The `generate_certs` sidecar creates self-signed TLS certificates that secure this internal control plane connection — ensuring nothing can inject malicious routing configuration. These certificates have nothing to do with public-facing TLS or backend communication.

### HTTP → HTTPS Redirect

| | Dev | Prod |
|---|---|---|
| **Choice** | 301 redirect on HTTP listener | Same |

A dedicated `HTTPRoute` on the HTTP listener (port 8080, mapped to host port 80) returns a 301 redirect to HTTPS for all requests. This is handled at the Gateway API level — no application-level redirect logic needed.

### OIDC at Gateway Level (Future)

Envoy Gateway supports `SecurityPolicy` with native OIDC integration. This would allow protecting routes at the gateway level without touching application code:

```yaml
# Not currently enabled — noted for future reference
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
spec:
  targetRefs:
    - kind: HTTPRoute
      name: admin-routes
  oidc:
    provider:
      issuer: https://${DOMAIN}/auth/realms/myrealm
    forwardAccessToken: true
```

Currently, OIDC authentication is handled at the Django level via django-allauth. Gateway-level OIDC is a future option for protecting non-Django endpoints or adding defense-in-depth.

---

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `generate_certs` sidecar | Internal | Provisions control plane TLS certificates |
| Let's Encrypt / certbot | External (prod) | Public TLS certificates |
| Ngrok | External (dev) | Public HTTPS URL for local development |

---

## Configuration

### Gateway Resource

The Gateway listener defines how Envoy accepts connections:

```yaml
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
      port: 8080
    - name: https
      protocol: HTTPS
      port: 8443
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: eih-tls-cert
```

The HTTPS listener terminates TLS using the referenced Secret. The HTTP listener exists solely for the 301 redirect route.

### Standalone Configuration

Envoy Gateway runs in standalone (non-Kubernetes) mode via a config file:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyGateway
provider:
  type: Custom
  custom:
    resource:
      type: File
      file:
        paths:
          - "/etc/envoy-gateway/common"
          - "/etc/envoy-gateway/common/backends"
          - "/etc/envoy-gateway/common/routes"
          - "/etc/envoy-gateway/{env}"
          - "/etc/envoy-gateway/{env}/routes"
    infrastructure:
      type: Host
      host:
        configHome: /tmp/envoy-gateway
        dataHome: /tmp/envoy-gateway
extensionApis:
  enableBackend: true
```

`{env}` is `dev` or `prod` — each has its own routes directory for environment-specific configuration. Common backends and routes shared across environments live in `common/`.

> [!important] Upstream experimental classification
> Envoy Gateway's standalone mode (file provider + host infrastructure) is classified as **experimental** upstream — their docs say "DO NOT use it in production." This label reflects control plane maturity gaps, not data plane reliability. The Envoy Proxy binary handling traffic is the same battle-tested binary used by Docker Hub, Teleport, and Tencent Cloud in production.
>
> What standalone mode lacks compared to the Kubernetes path: no built-in process lifecycle management (crash recovery, graceful drain), no orchestrated hot restart for zero-downtime upgrades, no E2E conformance test suite, single Envoy process (no HPA scaling), and a `v1alpha1` API surface that may change between releases. Feature parity is also catching up incrementally — Secret/ConfigMap parsing, BackendTLSPolicy, and Extension Server support were added across v1.2.7–v1.4.
>
> In our Docker Compose setup, most of these gaps are mitigated: Docker handles process restart (`restart: unless-stopped`), health checks, and container lifecycle. The main accepted tradeoffs for the VPS tier are brief downtime during upgrades (`docker compose down && up`) and no horizontal scaling. Both are acceptable for a single-VPS deployment. The Envoy AI Gateway project is also actively dogfooding standalone mode, accelerating its maturation toward production graduation.

### File Structure

```
services_configs/envoy_config/
├── standalone-dev.yaml              # Dev control plane config
├── standalone-prod.yaml             # Prod control plane config
├── common/
│   ├── backends/
│   │   ├── keycloak.yaml            # Backend: keycloak.local:8080
│   │   └── {service}.yaml           # Backend: {service}.local:8000
│   └── routes/
│       ├── keycloak.yaml            # HTTPRoute: /auth/* → keycloak
│       ├── {service}.yaml           # HTTPRoute: /{service}/* → service
│       └── https-redirect.yaml      # HTTPRoute: HTTP 301 → HTTPS
├── dev/
│   ├── gateway.yaml                 # Gateway with dev TLS config
│   └── routes/                      # Dev-only routes (echo server, etc.)
└── prod/
    ├── gateway.yaml                 # Gateway with Let's Encrypt cert ref
    └── routes/                      # Prod-only routes
```

### Backend Definition

Each backend service is defined as a Gateway API `Backend` resource:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: Backend
metadata:
  name: keycloak
  namespace: envoy-gateway-system
spec:
  endpoints:
    - fqdn:
        hostname: keycloak.local
        port: 8080
```

To add a new service: create a `Backend` YAML and a corresponding `HTTPRoute` YAML. No changes to existing configuration required.

### Route Definition

Routes use `HTTPRoute` with path matching and header modification:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: keycloak
  namespace: envoy-gateway-system
spec:
  parentRefs:
    - name: eg
      sectionName: https
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /auth/realms
      filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            set:
              - name: X-Forwarded-Proto
                value: https
              - name: X-Forwarded-Host
                value: ${DOMAIN}
              - name: X-Forwarded-Port
                value: "443"
      backendRefs:
        - group: gateway.envoyproxy.io
          kind: Backend
          name: keycloak
```

> [!important] X-Forwarded-Host
> The `X-Forwarded-Host` value must match your domain. Keycloak uses this to construct redirect URIs and validate the issuer. A mismatch causes redirect loops or token validation failures.

### Compose Configuration

| Variable / Setting | Dev | Prod |
|---|---|---|
| Host port 80 → container 8080 | ✅ | ✅ |
| Host port 443 → container 8443 | ✅ | ✅ |
| Standalone config | `standalone-dev.yaml` | `standalone-prod.yaml` |
| Let's Encrypt mount | — | `/etc/letsencrypt/live/${DOMAIN}` (read-only) |
| Networks | `internal` + `external` | `internal` + `external` |

---

## Certificate Management

### Overview

Let's Encrypt certificates live on the host and are mounted read-only into the Envoy container. Renewal is fully automated — certbot runs twice daily via systemd timer, and a deploy hook handles the gateway reload when a cert is actually renewed.

```
certbot timer (twice daily)
    → certbot renew
        → cert near expiry? yes:
            → ACME HTTP-01 challenge on port 80
            → new cert written to /etc/letsencrypt/archive/
            → symlinks in /live/ updated
            → deploy hook fires
                → gateway stop
                → generate-secrets.sh
                → gateway start
                → renewal logged
```

The deploy hook only fires when certbot actually renews — not on every timer tick.

---

### Obtaining a Certificate (First Time)

```bash
# Stop the gateway to free port 80
docker compose -f compose.prod.yaml stop gateway

# Source env vars
source /opt/deploy/basic-infrastructure/.env

# Obtain certificate
sudo certbot certonly \
  --standalone \
  --non-interactive \
  --agree-tos \
  --email ${CERT_EMAIL} \
  -d ${DOMAIN} \
  -d www.${DOMAIN}

# Verify
sudo openssl x509 -in /etc/letsencrypt/live/${DOMAIN}/fullchain.pem -noout -enddate

# Regenerate secrets and start gateway
sudo bash scripts/generate-secrets.sh
docker compose -f compose.prod.yaml start gateway
```

---

### Automated Renewal Setup

**Step 1 — Create the deploy hook**

Certbot automatically runs scripts in `/etc/letsencrypt/renewal-hooks/deploy/` after every successful renewal.

```bash
sudo nano /etc/letsencrypt/renewal-hooks/deploy/restart-gateway.sh
```

```bash
#!/bin/bash
set -euo pipefail

# ==============================================================================
# CERTBOT DEPLOY HOOK - Certificate Renewal & Gateway Reload
# ==============================================================================

ENV_FILE="/opt/deploy/basic-infrastructure/.env"
COMPOSE_FILE="/opt/deploy/basic-infrastructure/compose.prod.yaml"
SECRETS_SCRIPT="/opt/deploy/basic-infrastructure/scripts/generate-secrets.sh"
LOG="/var/log/certbot-renewal.log"

# Load environment variables
if [[ ! -f "$ENV_FILE" ]]; then
    echo "❌ Error: $ENV_FILE not found"
    exit 1
fi

source "$ENV_FILE"

# Validate required variables
: "${CERT_PATH:?❌ Error: CERT_PATH not set in .env}"
: "${SECRETS_OUTPUT_FILE:?❌ Error: SECRETS_OUTPUT_FILE not set in .env}"

# Validate secrets script exists
if [[ ! -f "$SECRETS_SCRIPT" ]]; then
    echo "❌ Error: Secrets script not found: $SECRETS_SCRIPT"
    exit 1
fi

# Validate Let's Encrypt certs
if [[ ! -d "$CERT_PATH" ]]; then
    echo "❌ Error: Certificate path does not exist: $CERT_PATH"
    exit 1
fi

if [[ ! -f "${CERT_PATH}/fullchain.pem" ]] || [[ ! -f "${CERT_PATH}/privkey.pem" ]]; then
    echo "❌ Error: Certificate files not found in $CERT_PATH"
    exit 1
fi

# ==============================================================================
# Reload gateway with renewed certificates
# ==============================================================================

echo "[$(date)] 🔄 Certificate renewed — reloading gateway..." >> "$LOG"

cd /opt/deploy/basic-infrastructure

docker compose -f "$COMPOSE_FILE" stop gateway      >> "$LOG" 2>&1
bash "$SECRETS_SCRIPT"                              >> "$LOG" 2>&1
docker compose -f "$COMPOSE_FILE" start gateway     >> "$LOG" 2>&1

echo "[$(date)] ✅ Gateway reloaded successfully."                                                                >> "$LOG"
echo "[$(date)] 📋 Certificate expires: $(sudo openssl x509 -in "${CERT_PATH}/fullchain.pem" -noout -enddate | cut -d= -f2)" >> "$LOG"
echo "[$(date)] ⚠️  Secrets regenerated: $SECRETS_OUTPUT_FILE"                                                   >> "$LOG"
```

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/restart-gateway.sh
```

**Step 2 — Enable the systemd timer**

Certbot (snap or apt) ships with a systemd timer. Verify it's active:

```bash
sudo systemctl list-timers | grep certbot
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer
```

If the timer doesn't exist (bare certbot install), create it:

```bash
# /etc/systemd/system/certbot-renew.timer
[Unit]
Description=Twice daily certbot renewal

[Timer]
OnCalendar=*-*-* 00,12:00:00
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
# /etc/systemd/system/certbot-renew.service
[Unit]
Description=Certbot renewal

[Service]
Type=oneshot
ExecStart=/usr/bin/certbot renew --quiet
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now certbot-renew.timer
```

**Step 3 — Test**

```bash
# Dry run — simulates renewal without touching real certs
sudo certbot renew --dry-run

# Force a real renewal to exercise the full hook
sudo certbot renew --force-renewal

# Watch the log
tail -f /var/log/certbot-renewal.log
```

---

### Manual Renewal

If automated renewal fails:

```bash
docker compose -f compose.prod.yaml stop gateway
sudo certbot renew --force-renewal
sudo bash scripts/generate-secrets.sh
docker compose -f compose.prod.yaml start gateway
```

---

## Dev vs Prod Differences

| Aspect | Dev | Prod |
|---|---|---|
| Public TLS | Ngrok handles it | Let's Encrypt at Envoy |
| Certificate management | None (Ngrok provides certs) | certbot + systemd timer + deploy hook |
| Standalone config | `standalone-dev.yaml` | `standalone-prod.yaml` |
| Gateway resource | Dev TLS Secret | Let's Encrypt cert Secret ref |
| Environment routes | Echo server, dev-only routes | Prod-only routes |
| `X-Forwarded-Host` | Ngrok domain | Production domain |
| Rate limiting | Not configured | Planned (local rate limiting via `BackendTrafficPolicy`) |
| Hostname in backends | `keycloak.local`, `{service}.local` | Same (Docker bridge, single host) |
| Resource limits | None | Memory: 1G limit, 512M reserved |

> [!tip] K8s migration
> When moving to Kubernetes, the Gateway API YAML files apply directly via `kubectl apply`. The main changes are: `Backend` endpoints switch from Docker FQDN (`keycloak.local`) to Kubernetes Service names (`keycloak.default.svc.cluster.local`), and TLS moves from mounted Let's Encrypt certs to cert-manager. See [[#Kubernetes Migration Path]] for details.

---

## Kubernetes Migration Path

The standalone file-based configs translate to Kubernetes with minimal changes:

| Resource | Docker Compose (current) | Kubernetes |
|---|---|---|
| `GatewayClass` | File in config directory | `kubectl apply -f gatewayclass.yaml` |
| `Gateway` | File with cert file path | `kubectl apply` with cert-manager `Secret` |
| `HTTPRoute` | File — **identical** | `kubectl apply` — **identical** |
| `Backend` | FQDN: `keycloak.local:8080` | Service: `keycloak.default.svc.cluster.local:8080` |
| TLS Secret | Mounted PEM files from host | cert-manager `Certificate` resource |
| Control plane | `standalone.yaml` + `generate_certs` | Envoy Gateway operator (installed via Helm) |

Backend endpoint change:

```yaml
# Docker Compose (current)
spec:
  endpoints:
    - fqdn:
        hostname: keycloak.local
        port: 8080

# Kubernetes
spec:
  endpoints:
    - fqdn:
        hostname: keycloak.default.svc.cluster.local
        port: 8080
```

Everything else — routes, header modifications, redirect rules, path matching — copies as-is.

---

## Monitoring

### Dev

- **Logs:** `docker compose logs -f gateway` — shows request routing, TLS handshakes, upstream errors
- **Echo server:** Send test requests to echo routes to verify headers, path matching, and forwarded values

### Prod

- **Logs:** `docker compose logs -f gateway` — structured access logs
- **Renewal log:** `tail -f /var/log/certbot-renewal.log` — deploy hook output on every renewal
- **Certbot log:** `/var/log/letsencrypt/letsencrypt.log` — ACME challenge details and errors
- **Metrics:** Envoy exposes Prometheus metrics on the admin port (19001): `envoy_http_downstream_rq_total`, `envoy_http_downstream_rq_xx` (status codes), `envoy_cluster_upstream_cx_connect_fail` (backend failures), `envoy_ssl_ciphers` (TLS usage)
- **Certificate expiry:** `sudo openssl x509 -in /etc/letsencrypt/live/${DOMAIN}/fullchain.pem -noout -enddate`
- **Dashboards:** Grafana dashboard (link TBD)
- **Alerts:** Certificate expiry approaching, backend connection failures, 5xx spike (configuration TBD)

> **Optional:** Full observability stack (Prometheus + Grafana) is the standard recommendation. Adjust dashboard and alert links once provisioned.

---

## Runbooks

### Connection refused on HTTPS (port 443)

Gateway container is not running or port mapping is wrong. Check with `docker compose ps gateway`. Verify port mapping (`443:8443`) in compose. Check firewall: `sudo ufw status | grep 443`.

### Certificate verify failed / expired

Certificates have expired or renewal failed. Check expiry: `sudo openssl x509 -in /etc/letsencrypt/live/${DOMAIN}/fullchain.pem -noout -enddate`. Review certbot logs at `/var/log/letsencrypt/letsencrypt.log`. If automated renewal failed, follow [[#Manual Renewal]].

### 502 Bad Gateway

Envoy is running but the backend service is down. Check backend health with `docker compose ps`. Inspect gateway logs for upstream connection errors: `docker compose logs gateway | grep upstream_cx`. Restart the affected backend service.

### Keycloak redirect loop

Keycloak doesn't detect it's behind a proxy. Verify `X-Forwarded-Proto: https`, `X-Forwarded-Host: ${DOMAIN}`, and `X-Forwarded-Port: 443` are set in the Keycloak `HTTPRoute` filter. Confirm Keycloak has `--proxy-headers=xforwarded` in its startup flags. See [[service-keycloak#Startup Flags]].

### New service not routing

Verify the `Backend` YAML has the correct hostname and port. Verify the `HTTPRoute` YAML references the correct `Backend` name and has `parentRefs` pointing to the `eg` Gateway. Restart Envoy Gateway to pick up the new files. Check that the service container is on the `internal` Docker network.

### Debugging headers and routing with echo server

Send a request to the echo server's dev-only route (e.g., `curl -v https://${DOMAIN}/echo/`) — it mirrors back the full request including all headers Envoy injected (`X-Forwarded-Proto`, `X-Forwarded-Host`, `X-Forwarded-Port`), the matched path, and the request method. Use this to verify header manipulation filters and path matching before wiring up a real service.

### Certificate renewal fails

Check that port 80 is accessible from the internet (certbot uses HTTP-01 challenge). Check `sudo systemctl status certbot.timer`. Review deploy hook logs at `/var/log/certbot-renewal.log` and certbot logs at `/var/log/letsencrypt/letsencrypt.log`. Common cause: gateway is holding port 80 — follow [[#Manual Renewal]] to recover.

---

## ADRs

- **ADR: Envoy Gateway over Nginx/Traefik** — Gateway API resource format provides Kubernetes portability without config rewrites. Same YAML runs in Docker standalone and K8s.
- **ADR: Standalone file provider mode** — Enables Gateway API resources without a Kubernetes cluster. File-based config is version-controlled and works identically in dev and prod Docker Compose. Upstream classifies standalone mode as experimental due to control plane maturity gaps (no built-in process lifecycle, no conformance test suite, incremental feature parity). Accepted for VPS tier because Docker Compose mitigates the critical gaps (restart policies, health checks), and the data plane (Envoy Proxy) is production-grade regardless of control plane mode. Revisit classification when standalone graduates upstream or when moving to Kubernetes.
- **ADR: Let's Encrypt with host-level certbot** — Certificates managed on the host, mounted read-only into container. Simpler than in-container cert management, compatible with standard certbot renewal flow. Deploy hook handles gateway reload after every renewal.
- **ADR: Certbot deploy hook over cron** — Deploy hook fires only on actual renewal, not on every timer tick. Keeps the gateway restart tied to the cert lifecycle without polling or external coordination.
- **ADR: HTTP 301 redirect at gateway level** — All HTTP→HTTPS redirect handled by Envoy, no application-level redirect logic needed. Single point of enforcement.
- **ADR: `generate_certs` for control plane TLS** — Self-signed certificates secure the xDS gRPC channel between Envoy controller and proxy. Prevents malicious route injection.

---

## References

- [Envoy Gateway Documentation](https://gateway.envoyproxy.io/docs/)
- [Kubernetes Gateway API Specification](https://gateway-api.sigs.k8s.io/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Envoy Proxy Documentation](https://www.envoyproxy.io/docs)
- [Certbot Documentation](https://eff-certbot.readthedocs.io/en/stable/)
