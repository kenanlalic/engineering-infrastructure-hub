---
title: "Envoy Gateway"
created: "2026-02-10"
updated: "2026-02-13"
version: 3.0.0
status: published
type: reference
owner: "platform-team"
lifecycle: production
tags: [service, envoy, gateway, tls, routing, proxy]
---

# Envoy Gateway — TLS Termination & Traffic Routing

Front door for all platform traffic. Handles TLS termination, path-based and subdomain routing, HTTP→HTTPS redirect, and header manipulation. Uses Kubernetes Gateway API resources in standalone file mode — the same configs work in Docker Compose and translate to Kubernetes with minimal changes.

> 📍 **Type:** Service Reference
> 👤 **Owner:** Ktwenty Threel
> 🎯 **Outcome:** Understand the Envoy Gateway Setup

---

## Table of Contents

- [[#Overview]]
- [[#Architecture]]
- [[#Design Decisions]]
- [[#Dependencies]]
- [[#Configuration]]
- [[#Dev ↔ Prod Differences]]
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

**Prod:** Let's Encrypt certificates are obtained via `certbot` on the host and mounted read-only into the Envoy container. The Gateway resource references these via a `Secret` that points to the mounted PEM files. Certificate renewal is automated via systemd timer — certbot renews, a deploy hook restarts Envoy Gateway, and the new certificates are loaded. Zero-downtime renewal.

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

> [!important] Renewal workflow
> Certbot runs twice daily via systemd timer. When certificates are within 30 days of expiry, certbot performs an ACME HTTP-01 challenge, obtains new certs, updates symlinks, and triggers a deploy hook that restarts Envoy Gateway. See [[operations-index|Operations]] for manual renewal procedures and failure recovery.

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

>[!important] Upstream experimental classification 
>Envoy Gateway's standalone mode (file provider + host infrastructure) is classified as **experimental** upstream — their docs say "DO NOT use it in production." This label reflects control plane maturity gaps, not data plane reliability. The Envoy Proxy binary handling traffic is the same battle-tested binary used by Docker Hub, Teleport, and Tencent Cloud in production.
>
What standalone mode lacks compared to the Kubernetes path: no built-in process lifecycle management (crash recovery, graceful drain), no orchestrated hot restart for zero-downtime upgrades, no E2E conformance test suite, single Envoy process (no HPA scaling), and a `v1alpha1` API surface that may change between releases. Feature parity is also catching up incrementally — Secret/ConfigMap parsing, BackendTLSPolicy, and Extension Server support were added across v1.2.7–v1.4.
>
In our Docker Compose setup, most of these gaps are mitigated: Docker handles process restart (`restart: unless-stopped`), health checks, and container lifecycle. The main accepted tradeoffs for the VPS tier are brief downtime during upgrades (`docker compose down && up`) and no horizontal scaling. Both are acceptable for a single-VPS deployment. The Envoy AI Gateway project is also actively dogfooding standalone mode, accelerating its maturation toward production graduation.

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

## Dev ↔ Prod Differences

| Aspect | Dev | Prod |
|---|---|---|
| Public TLS | Ngrok handles it | Let's Encrypt at Envoy |
| Certificate management | None (Ngrok provides certs) | Certbot + systemd timer + deploy hook |
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
- **Metrics:** Envoy exposes Prometheus metrics on the admin port (19001): `envoy_http_downstream_rq_total`, `envoy_http_downstream_rq_xx` (status codes), `envoy_cluster_upstream_cx_connect_fail` (backend failures), `envoy_ssl_ciphers` (TLS usage)
- **Certificate monitoring:** `make cert-info` shows expiry dates. Certbot sends email alerts on renewal failure.
- **Dashboards:** Grafana dashboard (link TBD)
- **Alerts:** Certificate expiry approaching, backend connection failures, 5xx spike (configuration TBD)

> **Optional:** Full observability stack (Prometheus + Grafana) is the standard recommendation. Adjust dashboard and alert links once provisioned.

---

## Runbooks

### Connection refused on HTTPS (port 443)

Gateway container is not running or port mapping is wrong. Check with `docker compose ps gateway`. Verify port mapping (`443:8443`) in compose. Check firewall: `sudo ufw status | grep 443`.

### Certificate verify failed / expired

Certificates have expired or renewal failed. Check expiry with `make cert-info`. Review certbot logs at `/var/log/letsencrypt/letsencrypt.log`. If automated renewal failed: stop the gateway (`docker compose stop gateway`), renew manually (`sudo certbot renew`), restart (`docker compose start gateway`).

### 502 Bad Gateway

Envoy is running but the backend service is down. Check backend health with `docker compose ps`. Inspect gateway logs for upstream connection errors: `docker compose logs gateway | grep upstream_cx`. Restart the affected backend service.

### Keycloak redirect loop

Keycloak doesn't detect it's behind a proxy. Verify `X-Forwarded-Proto: https`, `X-Forwarded-Host: ${DOMAIN}`, and `X-Forwarded-Port: 443` are set in the Keycloak `HTTPRoute` filter. Confirm Keycloak has `--proxy-headers=xforwarded` in its startup flags. See [[service-keycloak#Startup Flags]].

### New service not routing

Verify the `Backend` YAML has the correct hostname and port. Verify the `HTTPRoute` YAML references the correct `Backend` name and has `parentRefs` pointing to the `eg` Gateway. Restart Envoy Gateway to pick up the new files. Check that the service container is on the `internal` Docker network.

### Debugging headers and routing with echo server
Send a request to the echo server's dev-only route (e.g., curl -v https://${DOMAIN}/echo/) — it mirrors back the full request including all headers Envoy injected (X-Forwarded-Proto, X-Forwarded-Host, X-Forwarded-Port), the matched path, and the request method. Use this to verify header manipulation filters and path matching before wiring up a real service.
### Certificate renewal fails

Check that port 80 is accessible from the internet (certbot uses HTTP-01 challenge). Review deploy hook logs at `/var/log/certbot-renewal.log`. Common cause: gateway is holding port 80 during renewal — the deploy hook should handle restart ordering. See [[operations-index|Operations]] for the full renewal recovery procedure.

---

## ADRs

- **ADR: Envoy Gateway over Nginx/Traefik** — Gateway API resource format provides Kubernetes portability without config rewrites. Same YAML runs in Docker standalone and K8s.
- **ADR: Standalone file provider mode** — Enables Gateway API resources without a Kubernetes cluster. File-based config is version-controlled and works identically in dev and prod Docker Compose. Upstream classifies standalone mode as experimental due to control plane maturity gaps (no built-in process lifecycle, no conformance test suite, incremental feature parity). Accepted for VPS tier because Docker Compose mitigates the critical gaps (restart policies, health checks), and the data plane (Envoy Proxy) is production-grade regardless of control plane mode. Revisit classification when standalone graduates upstream or when moving to Kubernetes.
- **ADR: Let's Encrypt with host-level certbot** — Certificates managed on the host, mounted read-only into container. Simpler than in-container cert management, compatible with standard certbot renewal flow.
- **ADR: HTTP 301 redirect at gateway level** — All HTTP→HTTPS redirect handled by Envoy, no application-level redirect logic needed. Single point of enforcement.
- **ADR: `generate_certs` for control plane TLS** — Self-signed certificates secure the xDS gRPC channel between Envoy controller and proxy. Prevents malicious route injection.

---

## References

- [Envoy Gateway Documentation](https://gateway.envoyproxy.io/docs/)
- [Kubernetes Gateway API Specification](https://gateway-api.sigs.k8s.io/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Envoy Proxy Documentation](https://www.envoyproxy.io/docs)
- [[infrastructure-engineering-hub|Infrastructure Engineering Hub]]
- [[service-keycloak|Keycloak Configuration]]
- [[networking-and-ports|Networking & Ports Reference]]