---
title: "Django Service Template"
created: "2026-02-15"
updated: "2026-05-16"
version: 1.1.0
type: reference
owner: Kenan Lalic
lifecycle: production
tags: [service, django, template, copier, hacksoft, allauth, rbac, htmx, gunicorn, production-server]
---

# Django Service Template — Copier-Generated Services

Updatable Copier template that generates production-ready Django services. Each service ships with four apps following HackSoft Styleguide 2+ conventions: API (Django REST Framework), authentication (django-allauth + Keycloak OIDC), authorisation (RBAC with Keycloak as source of truth), and core (admin panel, healthchecks, shared infrastructure). An optional public app adds a django-htmx frontend with branding, legal pages, SEO, and privacy-first analytics.

> 📍 **Type:** Service Reference<br>
> 👤 **Owner:** Kenan Lalic<br>
> 🎯 **Outcome:** Understand the Django Service Template<br>

---

## Table of Contents

- [[#Overview]]
- [[#Architecture]]
- [[#Design Decisions]]
- [[#Dependencies]]
- [[#Configuration]]
- [[#Apps at a Glance]]
- [[#RBAC Model]]
- [[#Authorization Usage]]
- [[#HackSoft Styleguide Conventions]]
- [[#Testing]]
- [[#Dev vs Prod Differences]]
- [[#Production Server Configuration]]
- [[#Kubernetes Migration Path]]
- [[#Runbooks]]
- [[#ADRs]]
- [[#References]]

---

## Overview

The template is a [Copier](https://copier.readthedocs.io/) project that scaffolds a complete Django service from a questionnaire. Running `copier copy` generates a working service with authentication, authorisation, API, admin panel, Docker Compose integration, and pre-commit hooks — ready to start with `docker compose up`.

### What Gets Generated

A single Copier run produces a self-contained Django project with Docker packaging, infrastructure wiring, and five application modules:

| App | Always Included | Purpose |
|---|---|---|
| **core** | ✅ | Admin panel, healthcheck endpoint, `TimeStampedModel`, `HtmxMixin`, `ErrorHandlingMixin`, context processors, shared exceptions |
| **authentication** | ✅ | django-allauth + Keycloak OIDC, backchannel token exchange, `CustomSocialAccountAdapter`, local login restriction |
| **authorization** | ✅ | RBAC engine — Keycloak group sync, role-permission mapping, `RBACPermissionBackend`, permission context middleware |
| **api** | ✅ | DRF integration, error handling mixins, API URL namespace |
| **public** | Feature toggle | Landing page, about page, legal pages, privacy-first analytics, sitemaps — gated by `include_frontend_ui` |

The generated service registers with the platform by adding a compose include and Envoy Gateway route files. See [[10-workspace-config#Adding a Service|Adding a Service]] for the full registration checklist.

### Copier Workflow

```
copier copy --trust gh:yourorg/django-service-template backend-service
cd backend-service
make setup       # Install pre-commit hooks, copy .env template
nano .env        # Set service slug, ports, client secret
```

Copier prompts for service identity (name, slug, description), branding (display name, colours), infrastructure (debug port, database port, Keycloak client ID), internationalisation (default language, supported languages), and feature toggles (SEO, analytics, frontend UI). Answers are stored in `.copier-answers.yml` for reproducible updates via `copier update --trust`.

> [!note] Updatable templates
> When the template evolves (new features, bug fixes, convention changes), run `copier update --trust` inside an existing service. Copier merges template changes with your customisations, respecting `_skip_if_exists` rules that protect your app code, migrations, `.env`, locale files, and compose overrides from being overwritten.

### Feature Toggle Dependencies

Three feature toggles control optional functionality. Analytics and SEO depend on the frontend UI being enabled — Copier only prompts for them when `include_frontend_ui` is `true`:

```
include_frontend_ui ──┬── include_seo (sitemaps, robots.txt)
                      └── include_analytics (privacy-first page view tracking)
```

When `include_frontend_ui` is `false`, the `apps/public` directory is still generated but not routed — the root URL configuration gates it with a Copier conditional. You can safely delete `apps/public` and remove `apps.public` from `INSTALLED_APPS` if you don't plan to use it.

---

## Architecture

### Project Directory Structure

```
backend-service/
├── .copier-answers.yml          # Copier answers for reproducible updates
├── .docker/
│   ├── Dockerfile.dev           # Dev image (debugpy, dev dependencies)
│   └── .zshrc_devcontainer      # Shell config for VS Code dev containers
├── .env                         # Environment variables (git-ignored)
├── .pre-commit-config.yaml      # Linting + formatting hooks
├── Dockerfile                   # Production image
├── Makefile                     # Dev shortcuts (setup, lint, test)
├── compose.yaml                 # Docker Compose service definition
├── compose.db.yaml              # Database compose (if separate)
├── compose.prod.yaml            # Production overrides
├── pyproject.toml               # Python project metadata
├── requirements/
│   ├── development.txt          # Dev dependencies (debugpy, pytest, etc.)
│   └── production.txt           # Production dependencies
└── src/
    ├── manage.py
    ├── pytest.ini
    ├── conftest.py              # Shared test fixtures
    ├── branding.yml             # UI branding config (name, colours, contact)
    ├── config/
    │   ├── __init__.py
    │   ├── settings/
    │   │   ├── __init__.py
    │   │   ├── base.py          # Shared settings
    │   │   ├── development.py   # Dev overrides (DEBUG, MailDev SMTP)
    │   │   ├── production.py    # Prod overrides (security, logging)
    │   │   └── test.py          # Test overrides
    │   ├── urls.py              # Root URL configuration
    │   ├── wsgi.py
    │   └── asgi.py
    ├── apps/
    │   ├── __init__.py
    │   ├── core/                # Shared infrastructure
    │   │   ├── apps.py
    │   │   ├── models.py        # TimeStampedModel
    │   │   ├── mixins.py        # HtmxMixin, ErrorHandlingMixin
    │   │   ├── middleware.py    # HealthCheckMiddleware
    │   │   ├── exceptions.py    # ApplicationError hierarchy
    │   │   ├── context_processors.py
    │   │   ├── selectors.py
    │   │   ├── services.py
    │   │   ├── views.py
    │   │   ├── urls.py
    │   │   └── templatetags/
    │   ├── authentication/      # OIDC + django-allauth
    │   │   ├── adapters.py      # CustomSocialAccountAdapter
    │   │   ├── middleware.py    # AdminOnlyLocal, HtmxAuthRedirect
    │   │   ├── signals.py       # user_groups_synced (EMIT)
    │   │   ├── selectors.py
    │   │   ├── services/
    │   │   │   ├── keycloak.py     # Group sync
    │   │   │   ├── local_login.py  # Local login enforcement (superuser-only rule)
    │   │   │   ├── profile.py      # Profile sync
    │   │   │   ├── registration.py # Local user registration + authentication
    │   │   │   │                   # (superuser/emergency access only — SSO is primary)
    │   │   │   └── htmx.py         # HTMX utilities
    │   │   ├── urls.py
    │   │   └── views.py
    │   ├── authorization/       # RBAC engine
    │   │   ├── constants.py     # Roles, permissions, mappings
    │   │   ├── backends.py      # RBACPermissionBackend
    │   │   ├── decorators.py    # @require_permission, @require_role
    │   │   ├── mixins.py        # PermissionRequiredMixin, RoleRequiredMixin
    │   │   ├── middleware.py    # PermissionContextMiddleware
    │   │   ├── signals.py       # handle_user_groups_synced (HANDLE)
    │   │   ├── selectors.py
    │   │   ├── services/
    │   │   │   ├── permissions.py
    │   │   │   └── roles.py
    │   │   └── templatetags/
    │   │       └── authorization_tags.py
    │   ├── api/                 # DRF integration
    │   │   ├── exceptions.py
    │   │   ├── mixins.py
    │   │   ├── urls.py
    │   │   └── views.py
    │   └── public/              # Optional frontend (include_frontend_ui)
    │       ├── admin.py.jinja
    │       ├── middleware.py.jinja
    │       ├── models.py.jinja
    │       ├── services.py.jinja
    │       ├── sitemaps.py.jinja
    │       ├── urls.py.jinja
    │       ├── selectors.py
    │       ├── apps.py
    │       └── views.py
    ├── locale/                  # Translation files (.po/.mo)
    ├── static/
    │   ├── css/
    │   │   ├── custom.css
    │   │   ├── cookie-consent.css
    │   │   └── public/
    │   ├── js/
    │   │   └── cookie-consent.js
    │   ├── images/
    │   │   └── logo.svg
    │   ├── favicons/
    │   └── vendor/
    │       ├── bootstrap/
    │       ├── bootstrap-icons/
    │       └── htmx/
    ├── media/                   # User uploads (git-ignored)
    └── templates/               # Django templates
```

### Internal App Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                        Django Service                             │
│                                                                   │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                     config/urls.py                          │ │
│   │   Non-localized: /admin/, /accounts/, /favicon.ico          │ │
│   │   Localized (i18n_patterns):                                │ │
│   │     /authentication/ → authentication.urls                  │ │
│   │     /api/            → api.urls                             │ │
│   │     /                → core.urls (dashboard, control panel) │ │
│   │     /                → public.urls (landing, about, legal)  │ │
│   └────────────────────┬────────────────────────────────────────┘ │
│                        │                                          │
│   ┌────────────────────┼────────────────────────────────────────┐ │
│   │                    ▼         App Layer                      │ │
│   │                                                             │ │
│   │   ┌───────────┐  ┌──────────────┐  ┌────────────────────┐   │ │
│   │   │   core    │  │   auth'n     │  │   authorization    │   │ │
│   │   │───────────│  │──────────────│  │────────────────────│   │ │
│   │   │ Admin     │  │ allauth      │  │ RBAC engine        │   │ │
│   │   │ Health    │  │ Keycloak     │  │ Group → Role sync  │   │ │
│   │   │ Mixins    │  │ OIDC adapter │  │ Permission backend │   │ │
│   │   │ Context   │◄─│ Group sync   │─▶│ is_staff/super     │   │ │
│   │   │ Base      │  │              │  │ flag management    │   │ │
│   │   │ models    │  │   signals ───┼──┼─▶ user_groups_     │   │ │
│   │   │           │  │              │  │   synced handler   │   │ │
│   │   └───────────┘  └──────────────┘  └────────────────────┘   │ │
│   │        ▲                                                    │ │
│   │        │ extends TimeStampedModel                           │ │
│   │   ┌────┴──────┐         ┌──────────┐                        │ │
│   │   │  public   │         │   api    │                        │ │
│   │   │───────────│         │──────────│                        │ │
│   │   │ Landing   │         │ DRF      │                        │ │
│   │   │ About     │         │ Error    │                        │ │
│   │   │ Legal     │         │ handling │                        │ │
│   │   │ Sitemap   │         │ Mixins   │                        │ │
│   │   │ Analytics │         │          │                        │ │
│   │   └───────────┘         └──────────┘                        │ │
│   │   (optional)                                                │ │
│   └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

### Middleware Stack

Order is critical. Each middleware has a specific position for a reason:

```python
MIDDLEWARE = [
    # ── Before ALLOWED_HOSTS ────────────────────────────────────
    "apps.core.middleware.HealthCheckMiddleware",
    # ── Django Security ─────────────────────────────────────────
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.locale.LocaleMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    # ── django-allauth ──────────────────────────────────────────
    "allauth.account.middleware.AccountMiddleware",
    # ── django-htmx (adds request.htmx attribute) ──────────────
    "django_htmx.middleware.HtmxMiddleware",
    # ── Authorization context (after HtmxMiddleware) ────────────
    "apps.authorization.middleware.PermissionContextMiddleware",
    # ── Local login restriction ─────────────────────────────────
    "apps.authentication.middleware.AdminOnlyLocalAuthMiddleware",
    # ── HTMX redirect handler (MUST BE LAST) ────────────────────
    "apps.authentication.middleware.HtmxAuthRedirectMiddleware",
    # ── Privacy analytics (if enabled, after HTMX) ──────────────
    # "apps.public.middleware.PrivacyAnalyticsMiddleware",
]
```

| Middleware | Position Rationale |
|---|---|
| `HealthCheckMiddleware` | Before `SecurityMiddleware` — intercepts `/health/` before `ALLOWED_HOSTS` validation so probes work without hostname config |
| `AccountMiddleware` | After `AuthenticationMiddleware` — allauth requires `request.user` to exist |
| `HtmxMiddleware` | Before authorisation and auth middleware — sets `request.htmx` used by downstream middleware |
| `PermissionContextMiddleware` | After `HtmxMiddleware` — attaches `user_roles`, `user_permissions`, `primary_role` to authenticated requests only |
| `AdminOnlyLocalAuthMiddleware` | After `PermissionContextMiddleware` — needs auth context to determine redirect behaviour |
| `HtmxAuthRedirectMiddleware` | Last — converts 302 redirects to `HX-Redirect` headers for HTMX requests (must see final response) |
| `PrivacyAnalyticsMiddleware` | After HTMX middleware — excludes HTMX requests from tracking |

> [!info] PermissionContextMiddleware and anonymous users
> `PermissionContextMiddleware` only queries the database when `request.user.is_authenticated` is `True`. Anonymous requests receive empty `user_roles = []` and `user_permissions = frozenset()` at zero DB cost.

### Authentication Backends

```python
AUTHENTICATION_BACKENDS = [
    "apps.authorization.backends.RBACPermissionBackend",
    "django.contrib.auth.backends.ModelBackend",
    "allauth.account.auth_backends.AuthenticationBackend",
]
```

`RBACPermissionBackend` is listed first so permission checks resolve through the RBAC role hierarchy before falling through to Django's default `ModelBackend`. The allauth backend handles social account authentication.

### Signal Flow: Authentication → Authorization

The authentication and authorisation apps communicate via Django signals, keeping them independently deployable:

```
1. User logs in via Keycloak (django-allauth OIDC flow)
2. CustomSocialAccountAdapter.pre_social_login() fires
3. keycloak.keycloak_sync_groups() syncs Keycloak groups → Django groups
4. user_groups_synced.send() emitted by authentication app
5. authorization.signals.handle_user_groups_synced() receives signal
6. roles.update_user_access_flags() updates is_staff / is_superuser
7. Permission cache cleared
8. PermissionContextMiddleware enriches all subsequent requests
```

The authentication app owns the _what_ (which groups does this user belong to), the authorisation app owns the _so what_ (what can those groups do). This separation means either app can be replaced or updated independently.

> [!info] Signal isolation in tests
> Signal tests use explicit `connect_signals()` / `disconnect_signals()` helpers from `apps.authorization.signals` to control handler registration. This prevents cross-test bleed when signals are registered globally via `AppConfig.ready()`. Always call `disconnect_signals()` in `tearDown`. See [[#Testing]] for the full pattern.

### URL Routing

The root URL configuration (`config/urls.py`) separates non-localised and localised routes:

**Non-localised** (no language prefix): admin panel (with login redirect to allauth), allauth account URLs, favicon redirect, and optionally sitemap.xml.

**Localised** (`i18n_patterns`, language prefix when multilingual): authentication URLs, core URLs (dashboard, control panel), API namespace, and optionally public URLs (landing, about, legal pages). The public app inclusion is gated by the `include_frontend_ui` Copier toggle.

> [!important] FORCE_SCRIPT_NAME and proxy headers
> When running behind Envoy Gateway with path-prefix routing (e.g., `/backend/`), three settings must be configured together in `config/settings/base.py`. Missing any one causes incorrect URL generation, broken redirects, or HTTPS detection failures:
>
> ```python
> # Uncomment and set after generation when using path-prefix routing:
> FORCE_SCRIPT_NAME = f"/{os.environ.get('PROJECT_SLUG', '')}/"
> USE_X_FORWARDED_HOST = True        # Trust X-Forwarded-Host from Envoy
> SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
> ```
>
> `FORCE_SCRIPT_NAME` alone → correct paths, wrong HTTPS detection. `USE_X_FORWARDED_HOST` alone → wrong domain in generated URLs. `SECURE_PROXY_SSL_HEADER` → required for `SECURE_SSL_REDIRECT` to work correctly.

### Debug Port Numbering

All services run on port 8000 internally (Docker service name resolves via the `internal` network — e.g., `backend.local:8000`). No external backend port mapping is needed. The only port that requires unique numbering across services is the debugpy port for VS Code remote attach:

| Service | Debug Port |
|---|---|
| Service 01 | 5678 |
| Service 02 | 5679 |
| Service 03 | 5680 |
| ... | +1 per service |

Set `DEBUG_PORT` in each service's `.env` to the next available number. This port is mapped to the host so VS Code can attach from the centralised `launch.json`. See [[10-workspace-config#Debugging|Workspace Debugging]] for the full setup.

### Gateway Path Transformation

Envoy Gateway strips the service path prefix before forwarding to Django. Django receives clean paths:

| Incoming Request | After Rewrite | Django Receives |
|---|---|---|
| `/{service}/` | `/` | `/` |
| `/{service}/admin/` | `/admin/` | `/admin/` |
| `/{service}/api/users/` | `/api/users/` | `/api/users/` |
| `/{service}/static/css/app.css` | `/static/css/app.css` | `/static/css/app.css` |

This is handled by the `URLRewrite` filter with `ReplacePrefixMatch` in the HTTPRoute YAML. See [[21-envoy-gateway#Route Definition|Envoy Route Definition]] for the full configuration.

---

## Design Decisions

### HackSoft Styleguide 2+

| | HackSoft Styleguide | Django "Fat Models" |
|---|---|---|
| **Choice** | Selectors + Services pattern | — |

All apps follow the [HackSoft Django Styleguide](https://github.com/HackSoftware/Django-Styleguide). Models are minimal (fields, Meta, `__str__`). Read operations go through selectors. Write operations and business logic go through services. Views are thin wrappers that delegate to selectors and services. This enforces a clear separation that scales from a single developer to a team — business logic lives in testable, reusable functions rather than being scattered across models, views, and serialisers. See [[#HackSoft Styleguide Conventions]] for the naming and structural rules.

### Copier over Cookiecutter

| | Copier | Cookiecutter |
|---|---|---|
| **Choice** | Updatable template with `copier update` | One-time generation |

Copier was chosen for its update mechanism. When the template evolves — new conventions, bug fixes, security patches — existing services can pull changes with `copier update --trust`. Cookiecutter generates once and the generated project drifts from the template permanently. Copier's `_skip_if_exists` rules protect customised files (app code, migrations, `.env`, locale files) while updating infrastructure files (Dockerfile, compose, settings).

### django-allauth over Manual OIDC

| | django-allauth | Manual OIDC (python-jose, oauthlib) |
|---|---|---|
| **Choice** | Managed provider with adapter pattern | — |

django-allauth handles the full OIDC authorisation code flow, token management, user creation, and social account linking. The `CustomSocialAccountAdapter` hooks into `pre_social_login()` to sync Keycloak groups on every login. Manual OIDC implementation would require reimplementing token validation, session management, CSRF protection, and error handling — all of which allauth provides and maintains. See [[22-keycloak-oidc#Authentication Flow|Keycloak Authentication Flow]] for the full sequence.

### RBAC Sync Strategy

| | Sync on Login | Webhook / Real-time |
|---|---|---|
| **Choice** | Sync groups from OIDC claims on every login | — |

Group memberships are synced from Keycloak OIDC claims into Django groups on every login via `pre_social_login()`. This means role changes in Keycloak take effect on the user's next login — not immediately. The tradeoff is simplicity: no webhook infrastructure, no Keycloak event listeners, no background workers. For most applications, next-login propagation is acceptable. If immediate revocation is required, Keycloak's token expiry can be shortened or a webhook listener can be added later without changing the core sync architecture.

### Local Login Restriction

| | Admin-only local login | Full local login |
|---|---|---|
| **Choice** | `AdminOnlyLocalAuthMiddleware` restricts local auth to superusers | — |

Regular users must authenticate through Keycloak SSO. Local Django username/password login is restricted to superusers only via middleware. This preserves emergency admin access when Keycloak is unavailable while ensuring all regular authentication flows through the centralised identity provider.

### Class-Based Views (Django) / Function-Based Views (DRF API)

| | Django views | DRF API endpoints |
|---|---|---|
| **Choice** | CBVs with mixins | `@api_view` FBVs |

All Django template and HTMX views use class-based views exclusively. Authorisation is applied via mixins (`PermissionRequiredMixin`, `RoleRequiredMixin`) — the authorisation surface is visible in class definitions rather than scattered across function signatures.

DRF API endpoints use HackSoft's preferred function-based `@api_view` pattern. This aligns with the Locality of Behaviour principle: the permission check, business logic call, and response serialisation are co-located in a single readable function rather than distributed across a class hierarchy.

```python
# Django/HTMX view — authorisation visible at class definition:
class ArticleListView(PermissionRequiredMixin, ListView):
    required_permission = "content.view"

# DRF API view — explicit, no mixin indirection:
@api_view(["GET"])
def article_list_api(request):
    permissions.require_permission_or_raise(
        user=request.user, permission="content.view"
    )
    articles = article_list()
    return Response(ArticleSerializer(articles, many=True).data)
```

Decorator-based equivalents (`@require_permission`, `@require_role`) exist for rare edge cases but are not the standard pattern.

### Branding via YAML

| | `branding.yml` | Environment variables |
|---|---|---|
| **Choice** | YAML file loaded at startup, exposed via context processor | — |

UI branding (display name, colours, contact info, assets) lives in `branding.yml` rather than environment variables. Branding is structural configuration — it rarely changes between deploys and benefits from version control, comments, and nested structure. The context processor loads it once at startup and exposes it as `BRANDING` in all templates. Environment variables are reserved for infrastructure concerns (database URLs, secrets, ports).

---

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| Envoy Gateway | Infrastructure | TLS termination, path-based routing, `X-Forwarded-*` headers |
| Keycloak | Infrastructure | OIDC authentication, group management, SSO |
| PostgreSQL | Infrastructure | Per-service database (created via init script) |
| MailDev | Dev tool | Email capture for verification and password reset flows |
| django-allauth | Python package | Social authentication with Keycloak OIDC |
| django-htmx | Python package | HTMX request handling via `request.htmx` |
| Django REST Framework | Python package | API layer |

The service connects to infrastructure via the Docker internal network. Keycloak is reached directly at `http://keycloak.local:8080` for backchannel token exchange (never through the public gateway). PostgreSQL is reached at `db:5432`. MailDev receives SMTP at `maildev:1025`.

For infrastructure setup details, see [[20-postgresql|PostgreSQL]], [[21-envoy-gateway|Envoy Gateway]], [[22-keycloak-oidc|Keycloak]], and [[23-maildev|MailDev]].

---

## Configuration

### Copier Variables

These are prompted during `copier copy` and stored in `.copier-answers.yml`:

**Service Identity**

| Variable | Default | Purpose |
|---|---|---|
| `service_name` | — | Human-readable name (e.g., "Inventory Service") |
| `service_slug` | Auto from name | Python package name, compose service name, `.env` `PROJECT_SLUG` |
| `service_description` | `"Enterprise Django + HTMX + Envoy + Keycloak"` | Used in `.env` and `branding.yml` |

**Branding**

| Variable | Default | Purpose |
|---|---|---|
| `project_display_name` | Same as `service_name` | UI display name → `branding.yml` `name` |
| `brand_color_primary` | `#0d6efd` | → `branding.yml` → CSS `var(--brand-primary)` |
| `brand_color_secondary` | `#6c757d` | → `branding.yml` → CSS `var(--brand-secondary)` |
| `brand_color_accent` | `#198754` | → `branding.yml` → CSS `var(--brand-accent)` |

**Infrastructure**

| Variable | Default | Purpose |
|---|---|---|
| `debug_port` | `5678` | debugpy port for VS Code remote attach (increment per service) |
| `db_port` | — | Host-mapped port for pgAdmin/DBeaver access (dev only) |
| `keycloak_client_id` | `myclient` | OIDC client ID registered in Keycloak realm |

**Internationalisation**

| Variable | Default | Purpose |
|---|---|---|
| `default_language` | `en` | `LANGUAGE_CODE` in Django settings |
| `supported_languages` | Same as default | Comma-separated codes (e.g., `en,de,bs`) |

**Feature Toggles**

| Variable | Default | Depends On | Purpose |
|---|---|---|---|
| `include_debug_toolbar` | `true` | — | Django Debug Toolbar in dev |
| `include_frontend_ui` | `false` | — | Public app (landing, about, legal pages) |
| `include_seo` | `false` | `include_frontend_ui` | Sitemaps, `robots.txt` |
| `include_analytics` | `false` | `include_frontend_ui` | Privacy-first page view tracking (GDPR compliant) |

### Environment Variables

After generation, the service `.env` file contains runtime configuration. Some variables are defined in the parent project's `.env` and propagated via compose include (`DOMAIN`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `KEYCLOAK_CLIENT_SECRET`). Key service-level variables:

| Variable | Example | Purpose |
|---|---|---|
| `PROJECT_SLUG` | `backend` | URL prefix, database name, compose project name |
| `DJANGO_SETTINGS_MODULE` | `config.settings` | Settings module |
| `DJANGO_ENV` | `development` | Environment identifier |
| `SECRET_KEY` | (generated) | Django secret key |
| `DEBUG` | `True` | Debug mode |
| `ALLOWED_HOSTS` | `localhost,127.0.0.1,.ngrok.app` | Hostname validation |
| `BEHIND_PROXY` | `True` | Enables proxy header trust (`X-Forwarded-*`) |
| `DATABASE_NAME` | `${PROJECT_SLUG}_db` | Per-service database name |
| `DATABASE_HOST` | `db` | PostgreSQL hostname (Docker internal) |
| `DATABASE_PORT` | `5432` | PostgreSQL internal port (always 5432) |
| `KEYCLOAK_SERVER_URL` | `http://keycloak.local:8080/auth` | Internal backchannel URL |
| `KEYCLOAK_HEALTH_URL` | `http://keycloak.local:9000/auth` | Keycloak health endpoint |
| `KEYCLOAK_REALM` | `myrealm` | OIDC realm name |
| `KEYCLOAK_CLIENT_ID` | `myclient` | OIDC client identifier |
| `BACKEND_INTERNAL_PORT` | `8000` | Container-internal Django port |
| `DEBUG_PORT` | `5678` | debugpy listen port (unique per service) |
| `DB_PORT` | (varies) | Host-mapped port for pgAdmin/DBeaver |
| `EMAIL_HOST` | `maildev` | SMTP server (MailDev in dev) |
| `EMAIL_PORT` | `1025` | SMTP port |

> [!info] Parent project variables
> `DOMAIN`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `KEYCLOAK_CLIENT_SECRET` are **not** defined in the service `.env`. They live in the parent project's (basic-infrastructure) `.env` and are propagated to services via Docker Compose `include`. This keeps credentials in one place.

> [!important] Client secret sync
> `KEYCLOAK_CLIENT_SECRET` is propagated from the parent project's `.env` via compose include. It must match the value in the Keycloak realm template. A mismatch is the most common cause of login failures. See [[22-keycloak-oidc#Runbooks|Keycloak Runbooks]] for troubleshooting.

### INSTALLED_APPS

App order matters — authorisation depends on authentication signals:

```python
INSTALLED_APPS = [
    # Django core...
    "django_htmx",
    "allauth",
    "allauth.account",
    "allauth.socialaccount",
    "allauth.socialaccount.providers.openid_connect",
    # Local apps (order matters)
    "apps.core",
    "apps.authentication",   # Emits signals
    "apps.authorization",    # Handles signals
    "apps.api",
    # "apps.public",         # If include_frontend_ui
]
```

### Health Check

Every generated service exposes a health endpoint handled by `HealthCheckMiddleware` in the core app. The middleware intercepts `/health/` before `ALLOWED_HOSTS` validation (placed before `SecurityMiddleware` in the stack), checks actual database connectivity, and returns a JSON response:

```json
{"status": "healthy", "database": "connected"}
```

This endpoint is used by Docker Compose health checks and Envoy Gateway routing verification. Through the gateway, it's accessible at `https://${DOMAIN}/${PROJECT_SLUG}/health/`. In Kubernetes, it wires directly to liveness and readiness probes — see [[#Kubernetes Migration Path]].

---

## Apps at a Glance

Each app follows the HackSoft Styleguide structure. Detailed documentation for each app is linked below — this section provides orientation.

### Core

Shared infrastructure used by all other apps.

| Component | Purpose |
|---|---|
| `TimeStampedModel` | Abstract base with `created_at` / `updated_at` (translated field names) |
| `HtmxMixin` | View mixin — swaps template to `htmx_template_name` for HTMX requests |
| `ErrorHandlingMixin` | View mixin — catches `ApplicationError` subclasses in `dispatch()`, renders structured error response at the configured status code |
| `HealthCheckMiddleware` | Intercepts `/health/` before `ALLOWED_HOSTS`, checks DB connectivity |
| `ApplicationError` | Base exception class — `ValidationError`, `PermissionDenied`, `ServiceUnavailable` |
| Context processor | Exposes `BRANDING`, i18n helpers, feature flags to all templates |
| Admin panel | Django admin at `/admin/` with login redirected to allauth |

### Authentication

OIDC authentication via django-allauth with Keycloak provider.

| Component | Purpose |
|---|---|
| `CustomSocialAccountAdapter` | Hooks `pre_social_login()` to sync profile and groups from OIDC claims |
| `AdminOnlyLocalAuthMiddleware` | Restricts local login to superusers — regular users must use Keycloak SSO |
| `HtmxAuthRedirectMiddleware` | Returns HTMX-compatible redirects (`HX-Redirect` header) for expired sessions |
| `keycloak_sync_groups` service | Syncs Keycloak group memberships into Django groups |
| `registration` service | Local user registration and authentication — superuser and emergency access only. All regular authentication flows through Keycloak SSO; this service is kept for bootstrapping and recovery scenarios. |
| `local_login` service | Business logic for the superuser-only local login rule — extracted from `AdminOnlyLocalAuthMiddleware` so the middleware stays a thin wrapper |
| `user_groups_synced` signal | Emitted after group sync — consumed by the authorisation app |
| `user_profile_synced` signal | Emitted after profile data updated from OIDC claims |
| `user_created_from_keycloak` signal | Emitted when a new Django user is auto-created via SSO |

### Authorization

RBAC engine following NIST Core + Hierarchical principles (Levels 1–2). See [[#RBAC Model]] and [[#Authorization Usage]] for full details.

| Component | Purpose |
|---|---|
| `constants.py` | Role definitions, permission mappings, Keycloak group-to-role configuration |
| `RBACPermissionBackend` | Custom auth backend resolving permissions from role mappings |
| `PermissionContextMiddleware` | Attaches `user_roles`, `user_permissions`, `primary_role` to every authenticated request |
| `handle_user_groups_synced` | Signal handler — updates `is_staff` / `is_superuser` from role definitions |
| `Permission` dataclass | Type-safe permissions as `ResourceDomain.Action` (e.g., `content.view`) |
| Mixins | `PermissionRequiredMixin`, `RoleRequiredMixin`, `AnyPermissionRequiredMixin`, `AllPermissionsRequiredMixin` |
| Decorators | `@require_permission`, `@require_role` — available for edge cases, CBV mixins are the standard pattern |
| Template tags | `{% has_permission %}`, `{% has_any_permission %}`, `can_view` / `in_role` filters |

### API

Django REST Framework integration with standardised error handling.

| Component | Purpose |
|---|---|
| Error handling mixins | Catches `ApplicationError` subclasses and returns structured JSON responses |
| URL namespace | All API endpoints under `/api/` prefix |

### Public (Optional)

Frontend pages gated by `include_frontend_ui`. Sub-features (`include_seo`, `include_analytics`) are conditionally generated via Copier Jinja templates.

| Component | Condition | Purpose |
|---|---|---|
| `LandingView` | `include_frontend_ui` | Public landing page driven by `BRANDING` context |
| `AboutView` | `include_frontend_ui` | About page with team section |
| `LegalPageView` | `include_frontend_ui` | Legal page detail (terms, privacy, imprint, withdrawal) |
| `LegalPage` model | `include_frontend_ui` | Versioned legal content, managed via admin |
| `PageView` model | `include_analytics` | Anonymous page view counter (GDPR compliant — no cookies, no IPs) |
| `PrivacyAnalyticsMiddleware` | `include_analytics` | Records page views, excludes static/admin/HTMX/XHR requests |
| `page_view_record` service | `include_analytics` | Atomic count increment with referrer domain extraction |
| `StaticViewSitemap` | `include_seo` | XML sitemap for static public pages |
| `robots.txt` route | `include_seo` | Robots exclusion at `/robots.txt` |

---

## RBAC Model

### Role Hierarchy

Authorisation follows [NIST RBAC](https://csrc.nist.gov/projects/role-based-access-control) Core and Hierarchical principles (Levels 1–2). Senior roles inherit all permissions from junior roles in their chain:

```
                    ┌─────────────────┐
                    │  Administrator  │  ← Full system access
                    │    (Tier 6)     │    is_staff=True, is_superuser=True
                    └────────┬────────┘
                             │ inherits
                    ┌────────▼────────┐
                    │     Manager     │  ← User viewing, full reports
                    │    (Tier 5)     │    is_staff=True
                    └────────┬────────┘
                             │ inherits
                    ┌────────▼────────┐
                    │     Editor      │  ← Full content lifecycle
                    │    (Tier 4)     │    is_staff=True
                    └────────┬────────┘
                             │ inherits
                    ┌────────▼────────┐
                    │    Operator     │  ← Day-to-day operations
                    │    (Tier 3)     │    is_staff=True
                    └────────┬────────┘
                             │ inherits
          ┌──────────────────┼──────────────────┐
          │                                     │
┌─────────▼──────────┐               ┌──────────▼────────────┐
│   Contributor      │               │      Reviewer         │
│    (Tier 2)        │               │      (Tier 2)         │
└─────────┬──────────┘               └──────────┬────────────┘
          │ inherits                            │
          └────────────────┬────────────────────┘
                           │
                  ┌────────▼────────┐
                  │     Viewer      │  ← Read-only access
                  │    (Tier 1)     │
                  └─────────────────┘

        SPECIAL ROLES (No Inheritance)
        ┌───────────────────┐
        │     Auditor       │  ← Read-only all + audit logs
        │   (Compliance)    │    is_staff=True
        └───────────────────┘
```

### Keycloak Group Mapping

Groups in Keycloak map to roles in Django via `KEYCLOAK_GROUP_TO_ROLE` in `apps/authorization/constants.py`:

| Keycloak Group | Django Role | `is_staff` | `is_superuser` |
|---|---|---|---|
| `django-admins` | Administrator | ✅ | ✅ |
| `django-managers` | Manager | ✅ | — |
| `django-editors` | Editor | ✅ | — |
| `django-operators` | Operator | ✅ | — |
| `django-contributors` | Contributor | — | — |
| `django-reviewers` | Reviewer | — | — |
| `django-viewers` | Viewer | — | — |
| `django-auditors` | Auditor | ✅ | — |

### Permission Format

Permissions follow a `domain.action` string format:

**Domains:** `content`, `users`, `reports`, `system`, `workflow`, `audit_log`

**Actions:** `view`, `create`, `update`, `delete`, `approve`, `publish`, `export`, `audit`, `manage`

**Examples:** `content.view`, `content.publish`, `users.manage`, `reports.export`

Permissions are defined as `Permission` dataclasses using typed enums (`PermissionAction`, `ResourceDomain`) for compile-time safety. The string representation (`domain.action`) is used in CBV mixins, template tags, and service checks.

### Customisation Points

| What | Where | When |
|---|---|---|
| Add/modify roles | `apps/authorization/constants.py` → `ROLES` | New permission levels needed |
| Add Keycloak groups | `apps/authorization/constants.py` → `KEYCLOAK_GROUP_TO_ROLE` | New group mapped in Keycloak |
| Add permissions | `apps/authorization/constants.py` → `ResourceDomain`, `PermissionAction` | New resource domains or actions |
| Change login behaviour | `apps/authentication/adapters.py` | Custom social account adapter logic |
| Add apps | `config/settings/base.py` → `INSTALLED_APPS` | New Django apps |
| Modify routing | `config/urls.py` | New URL patterns |

---

## Authorization Usage

### Class-Based Views

```python
from apps.authorization.mixins import (
    PermissionRequiredMixin,
    AnyPermissionRequiredMixin,
    AllPermissionsRequiredMixin,
    RoleRequiredMixin,
)

class ArticleListView(PermissionRequiredMixin, ListView):
    required_permission = "content.view"

class ContentAccessView(AnyPermissionRequiredMixin, TemplateView):
    required_permissions = ["content.view", "content.manage"]

class PublishView(AllPermissionsRequiredMixin, TemplateView):
    required_permissions = ["content.update", "content.publish"]

class EditorDashboard(RoleRequiredMixin, TemplateView):
    required_role = "Editor"
```

### In Services (HackSoft Pattern)

```python
from apps.authorization.services import permissions

def publish_article(*, user, article):
    """Publish article with permission check."""
    permissions.require_permission_or_raise(
        user=user, permission="content.publish"
    )
    article.status = "published"
    article.save()
    return article
```

### In Templates

```html
{% load authorization_tags %}

{# Check single permission #}
{% has_permission "content.view" as can_view %}
{% if can_view %}
    <a href="{% url 'content:list' %}">View Content</a>
{% endif %}

{# Check multiple permissions (OR) #}
{% has_any_permission "content.edit,content.manage" as can_edit %}

{# Using context processor variables (from PermissionContextMiddleware) #}
{% if "content.publish" in user_permissions %}
    <button hx-post="{% url 'publish' %}">Publish</button>
{% endif %}

{# Role badge #}
{% if primary_role %}
    <span class="badge">{{ primary_role.name }}</span>
{% endif %}

{# Filters #}
{% if user|can_view:"content" %}...{% endif %}
{% if user|in_role:"Administrator" %}...{% endif %}
```

### Authorization API Reference

**Selectors** (read-only queries):

```python
from apps.authorization import selectors

selectors.user_get_role_names(user=user)        # ['Editor', 'Reviewer']
selectors.user_get_roles(user=user)             # [Role(...), ...]
selectors.user_get_permissions(user=user)       # frozenset[Permission]
selectors.user_has_permission(user=user, permission="content.view")
selectors.user_has_any_permission(user=user, permissions=["a", "b"])
selectors.user_has_all_permissions(user=user, permissions=["a", "b"])
selectors.user_has_role(user=user, role_name="Editor")
selectors.user_get_primary_role(user=user)      # Role or None
```

**Services** (business logic):

```python
from apps.authorization.services import permissions, roles

permissions.check_permission(user=user, permission="content.view")
permissions.require_permission_or_raise(user=user, permission="content.view")
permissions.require_any_permission_or_raise(user=user, permissions=["a", "b"])
permissions.require_all_permissions_or_raise(user=user, permissions=["a", "b"])

roles.update_user_access_flags(user=user)       # Returns bool (changed)
roles.get_user_role_summary(user=user)          # Debug summary dict
```

---

## HackSoft Styleguide Conventions

All apps follow these structural rules. The template enforces them — new code should maintain them.

### Models

Models contain fields, `Meta`, and `__str__` only. No business logic beyond `clean()`. All field `verbose_name` arguments use `_()` for translation. Timestamps inherit from `TimeStampedModel` rather than duplicating fields. Use `UniqueConstraint` over the deprecated `unique_together`.

```python
class LegalPage(TimeStampedModel):
    slug = models.CharField(_("slug"), max_length=50, choices=SLUG_CHOICES, unique=True)
    title = models.CharField(_("title"), max_length=200)

    class Meta:
        verbose_name = _("Legal Page")

    def __str__(self):
        return f"{self.get_slug_display()} (v{self.version})"
```

### Selectors

Read-only database operations. Every selector uses keyword-only arguments (the `*` marker). Naming follows `{entity}_{verb}_{qualifier}`. Returns querysets or model instances. No side effects.

```python
def legal_page_get_or_404(*, slug: str) -> LegalPage:
    return get_object_or_404(LegalPage, slug=slug)
```

### Services

Write operations and business logic. Same keyword-only argument and naming rules as selectors. Services may call selectors but never the reverse. Explicit exception handling over silent failures.

```python
def page_view_record(*, path: str, referrer: str = "", host: str = "") -> None:
    referrer_domain = _extract_referrer_domain(referrer=referrer, host=host)
    try:
        obj, created = PageView.objects.get_or_create(...)
        if not created:
            PageView.objects.filter(pk=obj.pk).update(count=F("count") + 1)
    except DatabaseError:
        logger.warning("Failed to record page view for %s", path, exc_info=True)
```

### When to Split `services.py` into `services/`

Start with a single `services.py`. Split into a `services/` package when the file exceeds ~150 lines or when two or more clearly separate responsibility groups emerge (e.g., group sync, profile sync, HTMX utilities).

```
Single file:         Package (when it grows):
services.py          services/
                     ├── __init__.py   # re-export public API
                     ├── keycloak.py
                     └── profile.py
```

The `__init__.py` re-exports for stable import paths — callers import from `apps.authentication.services`, not from the individual submodule.

### Views

Thin wrappers that delegate to selectors and services. No ORM queries in views. Context assembly calls selectors; form handling calls services.

```python
class LegalPageView(HtmxMixin, TemplateView):
    template_name = "public/pages/legal_detail.html"

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context["page"] = legal_page_get_or_404(slug=self.kwargs["slug"])
        return context
```

### Admin

Dedicated `admin.py` per app. Fieldset labels use `_()`. Business logic delegates to services or uses model-level constraints — no raw ORM queries in admin methods.

### Exceptions

Domain-specific exceptions in `exceptions.py`, inheriting from `ApplicationError`. Services raise these; views and API mixins catch and handle them.

```
ApplicationError
├── ValidationError        # Business rule validation failed
├── PermissionDenied       # Missing required permissions
└── ServiceUnavailable     # External dependency unavailable
```

### Naming Summary

| Layer | Pattern | Example |
|---|---|---|
| Selector | `{entity}_{verb}_{qualifier}` | `legal_page_get_or_404` |
| Service | `{entity}_{verb}_{qualifier}` | `page_view_record` |
| Private helper | `_{verb}_{noun}` | `_extract_referrer_domain` |

---

## Testing

Tests are organised by architectural layer, matching the HackSoft pattern. Each test file targets a single layer:

| File | What It Tests | Layer |
|---|---|---|
| `test_models.py` | Field constraints, `__str__`, unique constraints | Models |
| `test_selectors.py` | Read operations, queryset behaviour, edge cases | Selectors |
| `test_services.py` | Business logic, write operations, error handling | Services |
| `test_views.py` | HTTP responses, context, integration with middleware | Views |
| `test_middleware.py` | Request/response transformation, header handling | Middleware |
| `test_mixins.py` | Mixin behaviour in isolation (e.g., `HtmxMixin` template switching) | Mixins |
| `test_signals.py` | Signal emission and handler behaviour across app boundaries | Signals |
| `test_constants.py` | Configuration integrity (role definitions, permission mappings) | Constants |

Tests run inside the Docker container via VS Code Test Explorer (attached via Remote Explorer) or directly with `pytest`. Shared fixtures (`conftest.py`) provide reusable `user`, `staff_user`, and `user_with_groups` fixtures.

### Shared Fixtures (`conftest.py`)

```python
@pytest.fixture
def user(db):
    return User.objects.create_user(
        username="testuser",
        email="testuser@example.com",
        password="testpass123"
    )

@pytest.fixture
def staff_user(db):
    return User.objects.create_user(
        username="staffuser",
        email="staff@example.com",
        password="testpass123",
        is_staff=True
    )

@pytest.fixture
def user_with_groups(db):
    """User with two Keycloak-mapped groups for RBAC testing.

    Has django-editors and django-reviewers — useful for testing
    multi-role permission resolution and primary_role selection.
    """
    user = User.objects.create_user(
        username="groupeduser",
        email="grouped@example.com",
        password="testpass123"
    )
    g1, _ = Group.objects.get_or_create(name="django-editors")
    g2, _ = Group.objects.get_or_create(name="django-reviewers")
    user.groups.add(g1, g2)
    return user
```

For view tests involving OIDC users, use `SocialAuthUserMixin` from `apps.authentication.tests.views`. OIDC-authenticated users always have a verified `EmailAddress` and a linked `SocialAccount` — the mixin ensures tests start from realistic production state rather than bare `User` objects:

```python
class TestProfileView(SocialAuthUserMixin, ViewTestCase):
    def setUp(self):
        super().setUp()
        self.user = self.create_social_auth_user(
            email="test@example.com",
            groups=["django-editors"],
        )
```

### Cross-App Signal Testing

Signal tests verify the authentication → authorisation boundary. The authentication app emits `user_groups_synced`, the authorisation app handles it via `handle_user_groups_synced`. Tests use explicit helpers to control handler registration:

```python
from apps.authorization.signals import connect_signals, disconnect_signals

class TestSignalIntegration(TestCase):
    def setUp(self):
        connect_signals()      # Register handler explicitly

    def tearDown(self):
        disconnect_signals()   # Guaranteed cleanup — prevents cross-test bleed

    def test_full_signal_flow_updates_flags(self):
        """Verify the complete authentication → authorisation path."""
        user = User.objects.create_user(...)
        group, _ = Group.objects.get_or_create(name="django-admins")
        user.groups.add(group)

        user_groups_synced.send(sender=User, user=user)

        user.refresh_from_db()
        self.assertTrue(user.is_staff)
        self.assertTrue(user.is_superuser)
```

> [!important] Test signal connection by effect, not introspection
> Do not assert signal connection by inspecting `signal.receivers` — weak references make this unreliable. Always verify connection by observing the side effect the handler produces (flag updated, cache cleared, etc.).

### View Test Fixtures

For view tests involving authenticated users, use `SocialAuthUserMixin` rather than `User.objects.create_user()` directly. Users authenticated via Keycloak always have three records: a `User`, a verified `EmailAddress`, and a `SocialAccount`. Views and middleware that check `SocialAccount.objects.filter(provider="keycloak")` will behave differently for users created without these records.

```python
class MyViewTest(SocialAuthUserMixin, ViewTestCase):
    def setUp(self):
        self.user = self.create_social_auth_user(
            email="test@example.com",
            groups=["django-editors"],
        )
```

Use `client.force_login(self.user)` after creation. Do not use `create_user()` directly in view tests unless testing the admin-only local login path specifically.

---

## Dev vs Prod Differences

| Aspect | Dev | Prod |
|---|---|---|
| Settings module | `config.settings` (`DJANGO_ENV=development`) | `config.settings` (`DJANGO_ENV=production`) |
| `DEBUG` | `True` | `False` |
| `ALLOWED_HOSTS` | `localhost,127.0.0.1,.ngrok.app` | Explicit domain list |
| Secret key | Hardcoded default | Generated, stored outside repo |
| Database | `db:5432` (shared PostgreSQL container) | Same container or managed service |
| Email | MailDev (`maildev:1025`) | Real SMTP provider |
| Static files | Django `runserver` serves them | `collectstatic` + Envoy / whitenoise |
| Debug toolbar | Enabled (if `include_debug_toolbar`) | Disabled |
| debugpy | Listening on `DEBUG_PORT` | Not installed |
| Application server | Django `runserver` (hot-reload, single process) | Gunicorn (multi-worker, multi-thread) |
| `SECURE_SSL_REDIRECT` | `False` | `True` (handled at Envoy, defence-in-depth) |
| Keycloak backchannel | `http://keycloak.local:8080` | Same (single-host VPS) |
| Compose file | `compose.yaml` | `compose.prod.yaml` |
| Container restart | `unless-stopped` | `unless-stopped` |
| Resource limits | None | `deploy.resources` in `compose.prod.yaml` |
| Logging | Console output | `json-file` driver with rotation |

---

## Production Server Configuration

### Why `runserver` Is Not for Production

Django's built-in `runserver` handles one request at a time in a single process. If two users click simultaneously, one waits. It has no process management, no fault tolerance, and no worker recycling. It is deliberately simple and safe for development only.

In production, a **WSGI server** sits between incoming traffic and Django. Its job is to run multiple copies of your Django application in parallel, distribute requests across them, and restart crashed workers automatically — without you doing anything. Django code never changes; only the layer in front of it does.

```
Development:
  Browser → runserver (1 process, 1 request at a time)

Production:
  Browser → Gunicorn → Worker 1 (Django) ─┐
                      → Worker 2 (Django) ──┤→ PostgreSQL / Keycloak
```

### Gunicorn — Default for This Platform

Gunicorn runs a fixed number of worker processes. Each worker is a full copy of your Django application. Each worker additionally handles multiple requests in parallel using threads — while one thread waits for a database query or Keycloak token exchange, the other threads continue serving requests.

```
Gunicorn master process  (manages workers, never handles requests)
├── Worker 1 (Django process)
│   ├── Thread 1 — handling login request, waiting for Keycloak
│   ├── Thread 2 — handling dashboard request, waiting for DB query
│   ├── Thread 3 — handling API request, actively processing
│   └── Thread 4 — idle
└── Worker 2 (Django process)
    └── Thread 1–4 — handling other requests
```

The current `compose.prod.yaml` command:

```bash
gunicorn config.wsgi:application \
    --bind=0.0.0.0:8000 \
    --workers=2 \
    --threads=4 \
    --worker-class=gthread \
    --timeout=120 \
    --keep-alive=5 \
    --max-requests=1000 \
    --max-requests-jitter=100 \
    --log-level=info \
    --access-logfile=- \
    --error-logfile=-
```

**Configuration reference:**

| Flag | Value | Why |
|---|---|---|
| `--workers` | `2` | Conservative for a shared environment. See worker sizing below. |
| `--threads` | `4` | I/O-bound workload — DB queries and Keycloak OIDC exchanges spend most of their time waiting. Threads fill that wait time with other requests. |
| `--worker-class` | `gthread` | Enables threads inside workers. Required when `--threads > 1`. |
| `--timeout` | `120` | Kills a worker that spends more than 120 seconds on a single request. 30s is too tight — Keycloak token exchanges can be slow under load or on restart. Note: this kills the entire worker process, not just the slow request. See uWSGI's `harakiri` below for per-request timeout. |
| `--keep-alive` | `5` | Holds the HTTP connection open 5 seconds after a response. Reduces connection overhead for HTMX partial updates that make many small requests. |
| `--max-requests` | `1000` | Retires a worker after 1000 requests and spawns a fresh one. Python applications accumulate small amounts of memory with each request — this is normal and not a bug. Worker recycling prevents slow accumulation from reaching the memory limit. |
| `--max-requests-jitter` | `100` | Randomises the recycling threshold per worker (between 1000 and 1100 requests). Without jitter, all workers would retire simultaneously, briefly leaving zero workers available under load. |
| `--access-logfile` | `-` | Logs to stdout. Docker and Kubernetes capture stdout automatically — no log file management needed. |

### Worker Sizing

The standard Gunicorn formula is `(2 × CPU cores) + 1`. On OCI Always Free, each node has 2 oCPU — and on A1 Arm, 1 oCPU = 1 core = 1 vCPU (single hardware execution thread, unlike x86 where 1 oCPU = 2 vCPU). The formula gives 5 workers for a dedicated 2-core server.

The platform runs 2 workers instead of 5 because the OCI Always Free cluster is **shared**: each node runs system pods, CNPG (PostgreSQL), Keycloak, ArgoCD, and a Django service simultaneously. Claiming 5 Django workers on a 2-core node leaves nothing for the other processes. The correct approach is to size Django conservatively and let the OS scheduler balance CPU time across all workloads.

```
OCI Always Free node: 2 oCPU, 12 GB RAM
Approximate memory distribution at rest:
  System pods + kubelet:   ~512 MB
  CNPG (PostgreSQL):       ~512 MB
  Keycloak:                ~512 MB
  ArgoCD components:       ~256 MB
  Django (2w × 4t):        ~512 MB
  ─────────────────────────────────
  Total expected:          ~2.3 GB  (well within 12 GB — headroom for spikes)
```

### Container Resource Limits

The `compose.prod.yaml` `deploy.resources` block sets a hard memory ceiling:

```yaml
deploy:
  resources:
    limits:
      memory: 1G      # Container is OOM-killed by the Linux kernel if exceeded
    reservations:
      memory: 512M    # Docker tries to guarantee this much RAM is available
```

> [!info] Docker Compose vs Docker Swarm
> `deploy.resources` is applied by modern Docker Compose (v2.17+) without Swarm mode. `limits.memory` enforces a real kernel cgroup ceiling. `deploy.replicas` and `deploy.restart_policy` are Swarm-only and ignored by `docker compose up` — but resource limits are not.

The memory arithmetic:

```
2 workers × 4 threads × ~64 MB per thread ≈ 512 MB under normal load
1 GB limit gives headroom for request spikes, large responses, and collectstatic
```

The Gunicorn worker configuration is the primary control. The `deploy.resources` limit is the safety net — if misconfigured workers grow past 1 GB, the container is killed rather than taking down the whole VPS.

### uWSGI — Alternative for Permanent VPS

| | Gunicorn | uWSGI |
|---|---|---|
| **Choice** | Default — VPS as stepping stone to K8s | Recommended alternative for permanent VPS |

uWSGI's key advantage is the **cheaper busyness algorithm** — dynamic worker scaling that watches actual load and adjusts the worker count in real time. Where Gunicorn runs a fixed team, uWSGI hires and fires workers based on how busy the system is:

```
uWSGI with cheaper busyness on a shared VPS:
  Night (low traffic):   4 workers → ~256 MB used by Django
                         freed RAM available to PostgreSQL query cache
  Morning spike:         busyness hits 70% → scales to 8, 12 workers
                         handles burst without request queueing
  After spike settles:   scales back to minimum over 30 seconds
```

On a shared VPS where Django, Keycloak, and PostgreSQL all compete for the same RAM, memory that Django releases at night benefits PostgreSQL's `shared_buffers` — making database queries faster for everyone. Gunicorn's fixed count holds its full memory allocation whether it needs it or not.

uWSGI also has `harakiri` — a per-request timeout that kills only the hanging request, not the entire worker process. Gunicorn's `--timeout` kills the whole worker, which drops all threads that worker was handling. For Keycloak OIDC exchanges that occasionally hang during Keycloak restarts, this is a meaningful operational difference.

**When to use uWSGI instead of Gunicorn:**

- The VPS tier is permanent — K8s is not in the near-term plan
- Traffic is variable (internal tools with sharp 9am spikes, overnight lulls)
- The VPS is shared between Django, Keycloak, and PostgreSQL and memory efficiency matters

**When to stay with Gunicorn:**

- The VPS is a stepping stone toward Kubernetes
- K8s HPA handles dynamic scaling at the pod level, making uWSGI's cheaper algorithm redundant
- Operational simplicity and consistent config across VPS and K8s outweigh efficiency gains

---

## Kubernetes Migration Path

| Aspect | Docker Compose (current) | Kubernetes |
|---|---|---|
| Deployment | `compose.yaml` service block | Deployment + Service manifests |
| Configuration | `.env` file | ConfigMap + Secrets |
| Health check | `HealthCheckMiddleware` at `/health/` | Same, wired to liveness/readiness probes |
| Database | `db:5432` | CNPG-managed PostgreSQL cluster |
| Keycloak backchannel | `http://keycloak.local:8080` | `http://keycloak.default.svc.cluster.local:8080` (encrypt via mesh mTLS) |
| Static files | Volume mount or whitenoise | Whitenoise (simplest), nginx sidecar, or CDN / object storage |
| Email | MailDev (dev) / SMTP relay (prod) | SMTP relay or SES |
| Scaling | Fixed Gunicorn workers | HPA based on CPU/memory metrics |
| Secrets | `.env` outside repo | Sealed Secrets (Vault: upgrade path for rotation requirements) |
| Resource limits | `deploy.resources` in compose | `resources` block in Deployment manifest |
| Image architecture | amd64 | arm64 (OCI Always Free) — verify all base images publish `linux/arm64` |

The application image is identical across tiers. Only environment variables, orchestration, and the resource configuration change.

### K8s-Specific Gunicorn Addition

When running in Kubernetes, add `--worker-tmp-dir /dev/shm` to the Gunicorn command:

```bash
gunicorn config.wsgi:application \
    --bind=0.0.0.0:8000 \
    --workers=2 \
    --threads=4 \
    --worker-class=gthread \
    --timeout=120 \
    --keep-alive=5 \
    --max-requests=1000 \
    --max-requests-jitter=100 \
    --worker-tmp-dir /dev/shm \
    --log-level=info \
    --access-logfile=- \
    --error-logfile=-
```

Gunicorn workers write heartbeat files to `/tmp` by default. In Kubernetes, `/tmp` may be on a read-only root filesystem. The worker writes the heartbeat, Gunicorn sees it as dead, restarts it, which immediately tries to write the heartbeat again — an infinite restart loop that appears as a running but cycling pod. `/dev/shm` is a RAM-backed tmpfs that is always writable in Kubernetes pods and requires no volume mount configuration.

### K8s Resource Configuration

The `deploy.resources` block from `compose.prod.yaml` does not carry over to Kubernetes. Set resource requests and limits explicitly in the Deployment manifest, sized consistently with the Gunicorn worker configuration:

```yaml
resources:
  requests:
    memory: 256Mi   # Kubernetes scheduler reserves this on a node
    cpu: 100m       # 1/10 of a CPU core — reasonable for an idle Django service
  limits:
    memory: 512Mi   # Container is OOM-killed if exceeded
    cpu: 500m       # Throttled at half a CPU core

# Arithmetic check:
# 2 workers × 4 threads × ~64 MB = 512 MB working memory
# limits.memory: 512Mi matches expected working memory
# requests.memory: 256Mi is half of limit — scheduler guarantee, not the ceiling
```

> [!warning] Resource limits are required on the OCI Always Free cluster
> Each node has 2 oCPU and 12 GB RAM shared between system pods, CNPG, Keycloak, ArgoCD, and Django services. A Django service with no limits can OOM-kill CNPG or Keycloak. Always set resource limits before deploying to this cluster.

### Liveness and Readiness Probes

The `/health/` endpoint from `HealthCheckMiddleware` wires directly to Kubernetes pod management:

```yaml
livenessProbe:
  httpGet:
    path: /health/
    port: 8000
  initialDelaySeconds: 30   # Give Gunicorn time to start all workers
  periodSeconds: 10
  failureThreshold: 3       # Kill and restart pod after 3 consecutive failures

readinessProbe:
  httpGet:
    path: /health/
    port: 8000
  initialDelaySeconds: 10   # Start checking sooner than liveness
  periodSeconds: 5
  failureThreshold: 3       # Remove from Gateway load balancing after 3 failures
                            # Does not restart the pod — liveness handles that
```

The difference: **liveness** asks "is this pod alive?" — failure triggers a restart. **Readiness** asks "is this pod ready to receive traffic?" — failure removes it from the Envoy Gateway backend rotation without restarting it. Django starts accepting connections before all Gunicorn workers are fully warm — the readiness probe keeps the pod out of rotation until it is actually ready.

---

## Runbooks

### Service won't start — ModuleNotFoundError

Missing Python dependency. Verify `requirements/development.txt` (or `production.txt`) includes all needed packages. Rebuild the container: `docker compose build --no-cache <service>`. If the error references an app module, check `INSTALLED_APPS` in settings matches the generated app names.

### Login redirects to Keycloak but fails with 401

Client secret mismatch between the parent project's `.env` (`KEYCLOAK_CLIENT_SECRET`, propagated via compose include) and the Keycloak realm. Verify both sides match. If the realm was reimported (`docker compose down -v`), the secret reverts to the realm template value. See [[22-keycloak-oidc#Client secret mismatch after reimport|Keycloak: Client Secret Mismatch]].

### Login works but user has no permissions

Keycloak groups are synced on login, but the authorisation app's `KEYCLOAK_GROUP_TO_ROLE` mapping doesn't include the user's group. Check the mapping in `apps/authorization/constants.py`. Verify the user's Keycloak groups in the Admin Console match the expected group names. Log out and log back in to trigger a fresh sync.

### Health endpoint returns 503 — database unreachable

PostgreSQL is not running or the service can't reach it. Check `docker compose ps db` — if unhealthy, see [[20-postgresql#Runbooks|PostgreSQL Runbooks]]. Verify `DATABASE_HOST` in `.env` is set to `db` (not `localhost`). Confirm both containers are on the `internal` network.

### Service returns 404 with path prefix in URL

Envoy Gateway is not stripping the service path prefix. Verify the HTTPRoute has the `URLRewrite` filter with `ReplacePrefixMatch: /`. Also check that `FORCE_SCRIPT_NAME` is configured in `base.py` if Django needs to generate URLs with the prefix. See [[21-envoy-gateway#Route Definition|Envoy Route Definition]].

### Gateway returns 503 — service unreachable

Backend hostname or port mismatch in the Envoy `Backend` YAML. Verify the hostname matches the Docker service hostname (e.g., `backend.local`) and the port is `8000` (the internal Django port). Check that the service container is on the `internal` network and is running: `docker compose ps`.

### HTMX requests return full page instead of partial

The view is missing `HtmxMixin`, or `htmx_template_name` is not set. Verify the view inherits `HtmxMixin` and that the HTMX partial template exists at the specified path. Check that `django-htmx` middleware is in the middleware stack (it sets `request.htmx`).

### copier update overwrites my customisations

Check `_skip_if_exists` in `copier.yml`. App code under `src/apps/*` is protected by default (except core, api, authentication, authorization — which are template-managed). If a file should be protected, add it to the skip rules. Migrations, `.env`, locale files, and compose overrides are already protected.

### Admin login page appears instead of Keycloak redirect

The admin login is intercepted by `AdminOnlyLocalAuthMiddleware` and redirected to Keycloak for non-superusers. If you're seeing the Django admin login form, either you're a superuser (expected behaviour) or the middleware isn't in the `MIDDLEWARE` stack. The root URL config also redirects `/admin/login/` to `account_login` as a fallback.

### Emails not arriving in MailDev

Verify `EMAIL_HOST=maildev` and `EMAIL_PORT=1025` in `.env`. Check that `EMAIL_BACKEND` is set to `django.core.mail.backends.smtp.EmailBackend` (not the console backend). See [[23-maildev#Runbooks|MailDev Runbooks]] for gateway routing issues.

### Analytics middleware not recording page views

The `PrivacyAnalyticsMiddleware` is only generated when `include_analytics=true`. If enabled, verify it's in the `MIDDLEWARE` stack. The middleware excludes admin, static, media, API, health, HTMX, and XHR requests by design. Only GET requests returning 200 are tracked.

---

## ADRs

- **ADR: HackSoft Styleguide 2+ over fat models** — Selectors + Services pattern enforces separation of concerns across all apps. Business logic is testable in isolation, views stay thin, models stay minimal. Team onboarding is faster because the pattern is consistent and documented. Tradeoff: more files per feature (model + selector + service + view), but each file has a single responsibility.
- **ADR: Copier over Cookiecutter** — Copier's `copier update` enables template evolution across existing services. `_skip_if_exists` protects customised files while updating infrastructure. Cookiecutter is one-shot — generated projects drift from the template permanently.
- **ADR: django-allauth over manual OIDC** — Managed authentication flow with adapter hooks for customisation. Handles token lifecycle, CSRF, session management, social account linking. Manual OIDC would require reimplementing all of this with ongoing maintenance burden.
- **ADR: RBAC sync on login over webhooks** — Group memberships synced from OIDC claims on every login. Simpler than webhook infrastructure — no event listeners, no background workers. Role changes propagate on next login. Acceptable latency for most applications; webhook option remains available for immediate revocation requirements.
- **ADR: NIST RBAC (Core + Hierarchical) over flat groups** — Proper role hierarchy with permission inheritance. Keycloak manages group assignments, Django maps groups to roles with defined permissions. Constrained RBAC (Level 3 — separation of duty) and Symmetric RBAC (Level 4 — role-permission audit) remain future options.
- **ADR: Local login restricted to superusers** — Emergency admin access preserved when Keycloak is unavailable. Regular users authenticate exclusively through SSO, maintaining centralised identity management.
- **ADR: Branding in YAML over environment variables** — Structural UI configuration benefits from version control, comments, and nesting. Environment variables reserved for infrastructure (secrets, URLs, ports). Context processor loads `branding.yml` once at startup.
- **ADR: Feature toggles via Copier conditionals** — `include_frontend_ui`, `include_seo`, `include_analytics` control code generation at template level. Jinja conditionals in `.py.jinja` files produce clean Python output — no runtime feature flags, no dead code paths in generated services.
- **ADR: Class-based views only (Django) / function-based views (DRF API)** — CBVs for Django template and HTMX views — authorisation surface visible at class definition. FBVs for DRF API endpoints — aligns with HackSoft's Locality of Behaviour guidance, permission check and business logic co-located in a single readable function.
- **ADR: Signal-based auth→authz communication** — Authentication emits `user_groups_synced`, authorisation handles it. Apps have no direct imports of each other's internals. Either can be replaced or tested independently. Django signals provide the decoupling without adding message broker complexity.
- **ADR: entity_verb_qualifier naming over verb-first** — Selectors and services follow `legal_page_get_or_404` not `get_legal_page_or_404`. Entity-first naming groups related functions naturally in alphabetical listings and IDE autocomplete. Consistent across all apps.
- **ADR: Privacy-first analytics over GA4** — `PageView` model stores only path, date, referrer domain, and count. No cookies, no IP addresses, no user identification. GDPR compliant under legitimate interest (Art. 6(1)(f)) without consent requirement. GA4 is available as an optional addition with cookie consent banner.
- **ADR: Gunicorn over uWSGI** — Gunicorn is the default because VPS is documented as a stepping stone toward Kubernetes. The same Gunicorn configuration runs unmodified on both tiers — no migration step between VPS and K8s. Kubernetes HPA handles dynamic scaling at the pod level, making uWSGI's cheaper algorithm redundant at the K8s tier. For a permanent VPS deployment where Kubernetes is not in scope, uWSGI's dynamic worker scaling and per-request `harakiri` timeout are genuinely preferable — the decision depends on which tier you are optimising for, not on capability.

---

## References

- [HackSoft Django Styleguide](https://github.com/HackSoftware/Django-Styleguide)
- [Copier Documentation](https://copier.readthedocs.io/)
- [django-allauth Documentation](https://docs.allauth.org/)
- [Django REST Framework](https://www.django-rest-framework.org/)
- [django-htmx Documentation](https://django-htmx.readthedocs.io/)
- [NIST RBAC Model](https://csrc.nist.gov/projects/role-based-access-control)
- [OWASP Access Control Cheat Sheet](https://owasp.org/www-community/Access_Control)
- [Gunicorn Documentation](https://docs.gunicorn.org/)
- [uWSGI cheaper busyness algorithm](https://uwsgi-docs.readthedocs.io/en/latest/Cheaper.html)
