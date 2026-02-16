---
title: "MailDev"
created: "2026-02-15"
updated: "2026-02-15"
version: 1.0.0
status: published
type: reference
owner: "platform-team"
lifecycle: development
tags: [service, maildev, email, smtp, dev-only]
---

# MailDev — Local Email Capture & Inspection

Dev-only SMTP server that captures all outbound email from all platform services. Provides a web UI for inspecting email content, formatting, and delivery without sending anything externally. Used for Django mail delivery and email formatting testing, and Keycloak user management workflow testing (verification emails, password resets, account actions).

> 📍 **Type:** Service Reference
> 👤 **Owner:** Ktwenty Threel
> 🎯 **Outcome:** Understand the MailDev Setup

---

## Table of Contents

- [[#Overview]]
- [[#Architecture]]
- [[#Design Decisions]]
- [[#Dependencies]]
- [[#Configuration]]
- [[#Usage]]
- [[#Monitoring]]
- [[#Runbooks]]
- [[#ADRs]]
- [[#References]]

---

## Overview

MailDev runs as a single container (`maildev`) on the `internal` Docker network. It exposes no ports directly to the host — the web UI is accessed through Envoy Gateway, and the SMTP port is only reachable by other containers on the internal network.

All platform services that send email are configured to point their SMTP settings at `maildev.local:1025` in development. Every email sent by any service — Django or Keycloak — is intercepted by MailDev and made available for inspection in the web UI. Nothing leaves the local network.

### What Gets Captured

| Source | Email Types |
|---|---|
| Django services | Email verification, password reset, notification emails, any custom `send_mail()` calls |
| Keycloak | Account verification, password reset, admin-initiated actions, login alerts (if configured in realm) |

### Access

| Interface | Port | Access Method |
|---|---|---|
| Web UI | 1080 (container) | Through Envoy Gateway — routed via dev-only `HTTPRoute` |
| SMTP | 1025 (container) | Internal only — other containers connect to `maildev.local:1025` |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        internal network                          │
│                                                                  │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│   │  Keycloak    │    │  Django      │    │  Django      │       │
│   │              │    │  Service A   │    │  Service N   │       │
│   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘       │
│          │                   │                   │               │
│          │  SMTP :1025       │  SMTP :1025       │               │
│          │                   │                   │               │
│          └─────────┐         │         ┌─────────┘               │
│                    ▼         ▼         ▼                         │
│              ┌────────────────────────────┐                      │
│              │        MailDev             │                      │
│              │  ───────────────────────   │                      │
│              │  SMTP server  → :1025      │                      │
│              │  Web UI       → :1080      │                      │
│              │  ───────────────────────   │                      │
│              │  Captures all email        │                      │
│              │  In-memory storage         │                      │
│              └────────────────────────────┘                      │
│                    ▲                                             │
│                    │ HTTP :1080                                  │
│              ┌─────┴──────────────┐                              │
│              │   Envoy Gateway    │                              │
│              │   (dev route)      │                              │
│              └────────────────────┘                              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

MailDev sits exclusively on the `internal` Docker network. The web UI is served through Envoy Gateway via a dev-only route — it is not accessible when running the production compose file. SMTP traffic stays entirely within the Docker bridge network.

For the full platform architecture, see [[infrastructure-engineering-hub#Architecture|Infrastructure Engineering Hub — Architecture]].

---

## Design Decisions

### MailDev over Alternatives

| | MailDev | Alternatives |
|---|---|---|
| **Choice** | `maildev/maildev:2.1.0` | Mailhog, Mailtrap, Ethereal |

MailDev was chosen for its simplicity — a single container with a built-in web UI, no configuration required. It accepts SMTP on one port and serves a web interface on another. Emails are stored in memory (lost on container restart), which is appropriate for a dev tool.

Mailhog is functionally equivalent but hasn't seen active development since 2020. Mailtrap is a cloud service that requires account signup. MailDev is actively maintained, lightweight, and runs entirely locally.

### Gateway-Routed Web UI

| | Through Gateway | Direct Port Exposure |
|---|---|---|
| **Choice** | Envoy dev-only route | — |

The MailDev web UI is accessed through Envoy Gateway rather than a directly exposed host port. This keeps the access pattern consistent across all services — everything goes through the gateway in development, matching how services are accessed in production. It also means the web UI benefits from the same Ngrok tunnel, making it accessible during remote demos or mobile testing.

### In-Memory Storage

| | In-Memory | Persistent |
|---|---|---|
| **Choice** | Emails lost on restart | — |

MailDev stores captured emails in memory by default. This is intentional — dev email data is ephemeral and doesn't need to survive restarts. If you need to preserve specific emails for reference, screenshot or export them from the web UI before restarting the container.

---

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| Envoy Gateway (dev route) | Internal | Routes web UI traffic |
| None upstream | — | MailDev has no dependencies — it starts independently |

MailDev is a dependency target for services that send email, but it's a soft dependency — services don't fail if MailDev is down, they just can't deliver email. No `depends_on` is configured for MailDev in any service.

---

## Configuration

### Docker Compose Service Block

```yaml
maildev:
    hostname: maildev.local
    image: maildev/maildev:2.1.0
    restart: unless-stopped
    environment:
        - MAILDEV_WEB_PORT=1080
        - MAILDEV_SMTP_PORT=1025
    # No ports — web UI through gateway, SMTP internal only
    networks:
        - internal
```

### Environment Variables

| Variable | Value | Purpose |
|---|---|---|
| `MAILDEV_WEB_PORT` | `1080` | Port for the web inspection UI |
| `MAILDEV_SMTP_PORT` | `1025` | Port for the SMTP server that captures emails |

These are the only configuration parameters. MailDev requires no authentication, database, or external dependencies.

### Service SMTP Configuration

Services connecting to MailDev use these settings:

**Django** (`settings.py` or environment):

```python
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = "maildev.local"
EMAIL_PORT = 1025
EMAIL_USE_TLS = False
EMAIL_USE_SSL = False
```

**Keycloak** (realm settings or environment):

Keycloak's SMTP configuration is set in the realm template or Admin Console under Realm Settings → Email. The host is `maildev.local`, port `1025`, with no authentication and no TLS.

In production, both Django and Keycloak switch to a real SMTP provider (see [[service-keycloak#SMTP|Keycloak SMTP Configuration]]).

---

## Usage

### Django Email Testing

MailDev captures all emails sent via Django's `send_mail()`, `EmailMessage`, or any code path that uses the configured email backend. Common workflows:

**Email verification** — Register a new user via the Django frontend or API. The verification email appears in MailDev immediately. Click the verification link directly from the MailDev web UI to complete the flow.

**Password reset** — Trigger a password reset from the Django login page. Inspect the reset email in MailDev, verify the link format and expiry, then click through to test the reset flow end-to-end.

**Email formatting** — MailDev renders both HTML and plaintext email variants. Use it to verify that HTML templates render correctly, that plaintext fallbacks are readable, and that dynamic content (user names, links, dates) is populated correctly.

### Keycloak User Management Workflow Testing

Keycloak sends emails for several user lifecycle events. MailDev captures all of them:

**Account verification** — When a new user registers (if self-registration is enabled) or when an admin creates a user with "Verify Email" required, Keycloak sends a verification email. Inspect it in MailDev and click the link to verify.

**Password reset** — When a user requests a password reset from the Keycloak login page or an admin triggers "Reset Password" from the Admin Console, the reset email is captured. Test the full flow: request reset → inspect email → click link → set new password → verify login.

**Admin-initiated actions** — The Keycloak Admin Console allows sending "Required Actions" to users (verify email, update password, update profile). Each generates an email captured by MailDev.

### Web UI Features

The MailDev web UI provides real-time email inspection with auto-refresh — new emails appear without page reload. Each email shows the sender, recipient, subject, timestamp, and both HTML and source views. The UI supports searching by recipient, subject, or body text.

---

## Monitoring

### Dev

MailDev is a dev-only tool with minimal monitoring needs. `docker compose logs -f maildev` shows SMTP connection events and any errors. The web UI itself serves as the primary monitoring interface — if you can see emails arriving, MailDev is working.

### Prod

MailDev does not exist in production. The `compose.prod.yaml` does not include the MailDev service. Production email is handled by an external SMTP provider configured in each service's environment.

---

## Runbooks

### Emails not appearing in MailDev

Verify the sending service's SMTP configuration points to `maildev.local:1025`. Check that both the sending service and MailDev are on the `internal` network. Inspect MailDev logs for connection events: `docker compose logs maildev | grep -i smtp`. If the service uses `EMAIL_HOST = "localhost"` instead of `maildev.local`, it's trying to send to itself rather than the MailDev container.

### Web UI not loading through gateway

Check that the dev-only MailDev route exists in the Envoy Gateway dev routes directory. Verify the gateway is running: `docker compose ps gateway`. Check gateway logs for routing errors: `docker compose logs gateway | grep maildev`. If the route was recently added, restart the gateway to pick up the new config.

### MailDev container keeps restarting

Check logs: `docker compose logs maildev`. The most common cause is a port conflict — if another process is bound to 1025 or 1080 inside the container (unlikely with the official image). If the container is OOM-killed, the in-memory email store may have grown too large from extended test runs. Restart clears all stored emails.

### Emails appear in MailDev but links don't work

Email links contain the URL of the originating service. If the link points to `localhost` or an internal hostname, it won't resolve in your browser. For Django, verify `SITE_URL` or `DEFAULT_FROM_EMAIL` domain settings. For Keycloak, verify `KC_HOSTNAME` includes the Ngrok/public domain so that links in Keycloak emails use the correct base URL.

---

## ADRs

- **ADR: MailDev over Mailhog/cloud alternatives** — MailDev is actively maintained, runs entirely locally, requires zero configuration, and provides a web UI for email inspection. Mailhog is unmaintained since 2020. Cloud services (Mailtrap) require account signup and network access. MailDev's in-memory storage is appropriate for ephemeral dev email.
- **ADR: Gateway-routed web UI** — Web UI accessed through Envoy Gateway rather than direct port exposure. Keeps access pattern consistent with all other services. Enables remote access via Ngrok tunnel for demos and mobile testing.
- **ADR: No persistent email storage** — Emails stored in memory, lost on container restart. Dev email is ephemeral by nature. No volume mount needed, no cleanup required.

---

## References

- [MailDev GitHub Repository](https://github.com/maildev/maildev)
- [MailDev Docker Hub](https://hub.docker.com/r/maildev/maildev)
- [[infrastructure-engineering-hub|Infrastructure Engineering Hub]]
- [[service-keycloak|Keycloak Configuration]]
- [[service-keycloak#SMTP|Keycloak SMTP Configuration]]
- [[service-envoy-gateway|Envoy Gateway Configuration]]
- [[networking-and-ports|Networking & Ports Reference]]