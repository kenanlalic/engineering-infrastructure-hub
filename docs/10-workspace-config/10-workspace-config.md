---
title: "VS Code Workspace Configuration"
created: "2026-02-15"
updated: "2026-02-16"
version: 1.0.0
status: published
type: reference
owner: "platform-team"
lifecycle: production
tags: [workspace, vscode, debugger, devcontainer, onboarding]
---

# VS Code Workspace — Multi-Root Development Environment

Single VS Code instance managing all platform repositories, with centralized debugger launch configs, container-attached development, and parallelized test execution. From empty machine to hitting a breakpoint inside a running container.

> 📍 **Type:** Workspace Configuration
> 👤 **Owner:** Ktwenty Threel
> 🎯 **Outcome:** Understand the Workspace Configuration Process

---

## Table of Contents

- [[#Overview]]
- [[#Architecture]]
- [[#Prerequisites]]
- [[#Setup]]
- [[#Adding a Service]]
- [[#Workspace File Reference]]
- [[#Debugging]]
- [[#Testing]]
- [[#Ngrok & Public Access]]
- [[#Runbooks]]
- [[#ADRs]]
- [[#References]]

---

## Overview

The workspace is a single Git repository (`engineering-infrastructure-hub`) that contains the VS Code configuration, the documentation vault, and references to all other repositories. It does not contain service code — each service and the base infrastructure are their own Git repositories, cloned into this directory and git-ignored. VS Code sees them as separate workspace roots.

### Repository Layout

```
engineering-infrastructure-hub/
├── eih.code-workspace             # Multi-root workspace definition
├── .vscode/
│   ├── launch.json                 # Centralized debugger configs (all services)
│   └── settings.json               # Repo exclusions (prevents double entries)
├── docs/                           # Obsidian documentation vault
├── .gitignore
├── README.md
├── LICENSE
├── basic-infrastructure/           # Cloned, git-ignored
├── django-service-template/        # Cloned, git-ignored
└── *-service/                      # Generated services, git-ignored by wildcard
```

### What VS Code Shows

```
📁 🛠️ Setup Workspace                  ← This repo (workspace config, docs)
📁 📦 Basic Infrastructure             ← Envoy Gateway, Keycloak, PostgreSQL, etc.
   ──────────────────────────────
📁 📚 Django Service Template          ← Copier template for generating services
   ──────────────────────────────
📁 💽 Backend Service                  ← Generated service (example)
📁 💽 Billing Service                  ← Generated service (example)
```

Delimiter lines are visual separators created by empty `delimiter-*` directories, hidden from Explorer via `files.exclude`. Cloned repos are excluded from the Setup Workspace root so they don't appear twice.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        VS Code Workspace                            │
│                                                                     │
│   ┌──────────────────────────────────────────────────────────────┐  │
│   │ eih.code-workspace                                          │  │
│   │  Folder roots → emoji-prefixed, per-repo                     │  │
│   │  Settings → file exclusions, git, python analysis            │  │
│   │  Extensions → recommended on first open                      │  │
│   └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   ┌──────────────────┐    ┌────────────────────────────────────┐    │
│   │ .vscode/         │    │ Run & Debug (Ctrl+Shift+D)         │    │
│   │  launch.json     │───▶│  💽 Backend Service  → port 5678   │    │
│   │  (all services)  │    │  💽 Billing Service  → port 5679   │    │
│   │                  │    │  💽 Analytics Service → port 5680  │    │
│   └──────────────────┘    └───────────────┬────────────────────┘    │
│                                           │ debugpy attach          │
│                                           ▼                         │
│   ┌──────────────────────────────────────────────────────────────┐  │
│   │ Docker Containers (via Remote Explorer)                      │  │
│   │                                                              │  │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐         │  │
│   │  │ Django A │ │ Django B │ │ Keycloak │ │ Postgres │  ...    │  │
│   │  │ debugpy  │ │ debugpy  │ │          │ │          │         │  │
│   │  │ :5678    │ │ :5679    │ │          │ │          │         │  │
│   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘         │  │
│   │                                                              │  │
│   │  Test Explorer → discovers tests inside attached container   │  │
│   └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

The key principle: you develop inside the same containers that run your application. VS Code attaches to the running container via Remote Explorer, the debugger connects to `debugpy` over a mapped port, and the Test Explorer discovers tests from inside the container. What works here works in production.

---

## Prerequisites

| Requirement | Version | Verify |
|---|---|---|
| Docker Engine | 24.0+ | `docker --version` |
| Docker Compose | v2.34.0+ | `docker compose version` |
| Python | 3.11 | `python3 --version` |
| VS Code | Latest | `code --version` |
| Git | 2.30+ | `git --version` |
| Ngrok | Latest | `ngrok version` |

---

## Setup

### 1. Clone Workspace

```bash
git clone git@github.com:yourorg/engineering-infrastructure-hub.git
cd engineering-infrastructure-hub
```

### 2. Clone Infrastructure & Template

```bash
git clone git@github.com:yourorg/basic-infrastructure.git
git clone git@github.com:yourorg/django-service-template.git
```

These repos are listed in `.gitignore` — the workspace tracks references to them, not their contents.

### 3. Install VS Code Extensions

Open the workspace and VS Code prompts to install recommended extensions automatically.

| Extension | ID | Required? | Purpose |
|---|---|---|---|
| Python | `ms-python.python` | Required | Language support, IntelliSense (Pylance), debugging (debugpy), linting, testing |
| Dev Containers | `ms-vscode-remote.remote-containers` | Required | Remote Explorer for attaching to running containers, in-container debugging and test execution |
| Docker | `ms-azuretools.vscode-docker` | Required | Container management, log viewing, image inspection |
| Black Formatter | `ms-python.black-formatter` | Recommended | Auto-formatting on save, matches pre-commit hooks in the service template |
| GitLens | `eamodio.gitlens` | Optional | Enhanced git blame, history, and diff — helpful in multi-repo workspaces |

### 4. Open Workspace

```bash
code eih.code-workspace
```

### 5. Start Infrastructure

```bash
cd basic-infrastructure
cp .env.example .env
docker compose up -d
```

This starts the base platform:

| Container | Purpose | Startup |
|---|---|---|
| `eih-envoy-gateway` | TLS termination, path-based routing | Immediate |
| `eih-envoy-certs` | Control plane certificate generation | Runs once, exits |
| `eih-keycloak` | OIDC / SSO identity provider | 60–90s |
| `eih-postgres` | Shared database, per-service schemas | Immediate |
| `eih-maildev` | Email capture with web UI | Immediate |
| `eih-echo` | Request mirror for route debugging | Immediate |

Wait for all services to report healthy: `docker compose ps`.

> [!note] Keycloak startup
> Keycloak takes 60–90 seconds. The health check has `start_period: 90s`. It will show unhealthy briefly — this is normal.

### 6. Start Ngrok

```bash
ngrok http 8080
```

Copy the generated URL — you'll need it as `DOMAIN` in environment files and Keycloak redirect URIs. See [[#Ngrok & Public Access]] for details on free vs. paid accounts.

### 7. Develop Inside Containers

Once infrastructure is running:

1. Start your service (see [[#Adding a Service]])
2. The service container appears in **Remote Explorer** (left sidebar, monitor icon)
3. Open **Run & Debug** (Ctrl+Shift+D) → select your service from the dropdown → F5
4. Set breakpoints, step through code, inspect variables — all inside the running container
5. **Test Explorer** (Ctrl+Shift+T) shows all tests with parallelized per-test execution

---

## Adding a Service

Generating a new service and wiring it into the platform takes a service repo, infrastructure configuration, and a workspace entry.

### Step 1: Generate from Template

```bash
copier copy --trust gh:yourorg/django-service-template backend-service
cd backend-service
make setup
nano .env
```

> [!important] Naming convention
> Always name service directories with a `-service` suffix (e.g. `backend-service`, `billing-service`). The workspace `.gitignore` and `.vscode/settings.json` use `*-service` wildcards — following this convention means zero manual changes to workspace configuration files.

The generated service ships with API, authentication, authorization, and core apps. See the [[template-overview|Django Service Template]] for the full configuration checklist and port allocation.

### Step 2: Register with Infrastructure

Two files in `basic-infrastructure/` need updating:

**Add compose include** — edit `basic-infrastructure/compose.yaml`:

```yaml
include:
  - ../backend-service/compose.yaml
```

**Envoy settings for backend-service route** — three pre-configured YAML files:

```
basic-infrastructure/services_configs/envoy_config/
├── common/backends/backend-service.yaml
├── common/routes/backend-service.yaml
└── prod/routes/backend-service.yaml
```

Works with Django Service Template out of the box should you need to make changes or add services go to gateway integration section and either adjust the service name and port or copy and dajust the files.

### Step 3: Register with Workspace

Add a folder entry to `eih.code-workspace`:

```jsonc
{
  "name": "💾 Backend Service",
  "path": "backend-service/"
}
```

Check the debug configuration in `.vscode/launch.json`:

```jsonc
{
  "name": "Backend: Remote Attach",
  "type": "debugpy",
  "request": "attach",
  "connect": {
      "host": "localhost",
      "port": 5678
  },
  "pathMappings": [
      {
          "localRoot": "${workspaceFolder:💾 Backend Service}/src",
          "remoteRoot": "/code"
      }
  ],
  "justMyCode": false
}
```

Because you named the directory with a `-service` suffix, the `.gitignore` wildcard and `.vscode/settings.json` exclusion already cover it. No other files need editing.

### Step 4: Start & Verify

```bash
cd basic-infrastructure
docker compose up -d
curl https://${DOMAIN}/backend-service/health/
```

### Quick Checklist

- [ ] Generate service from template (name it `*-service`)
- [ ] Configure `.env` (slug, ports, client secret)
- [ ] Add `include` to `basic-infrastructure/compose.yaml`
- [ ] Check Envoy `Backend` and `HTTPRoute` YAML files
- [ ] Add folder entry to `eih.code-workspace`
- [ ] Check debug config in `.vscode/launch.json`
- [ ] `docker compose up -d` → verify health endpoint
- [ ] Select service in Run & Debug → F5 → hit a breakpoint

---

## Workspace File Reference

### `eih.code-workspace`

Defines folder roots, workspace settings, and extension recommendations.

**Folder roots** — each cloned repo appears as a separate root with emoji prefixes:

| Prefix | Meaning |
|---|---|
| 🛠️ | Workspace setup (this repo) |
| 📦 | Base infrastructure |
| 📚 | Template reference |
| 💾 or 💽 | Generated service |
| `────` | Visual delimiter (hidden `delimiter-*` dirs) |

**Workspace settings:**

| Setting | Purpose |
|---|---|
| `files.exclude` | Hides `__pycache__`, `.git`, `.pyc`, `.pytest_cache`, delimiter dirs |
| `scm.repositories.visible: 15` | Shows all Git repos in Source Control (default too low for multi-repo) |
| `git.openRepositoryInParentFolders: "always"` | Auto-detects Git repos in all workspace roots |
| `python.analysis.diagnosticSeverityOverrides` | Unused imports = warning, missing imports = error |

> [!note] Exclusion split
> Repo exclusions (`basic-infrastructure`, `*-service`, etc.) live in `.vscode/settings.json`, not in the workspace file. Workspace `files.exclude` applies to all roots — putting repo exclusions there would hide them everywhere, not just from the Setup Workspace root.

### `.vscode/settings.json`

Prevents cloned repos from appearing as subfolders of Setup Workspace. Without this, every repo shows twice — once as its own workspace root and once as a subfolder under 🛠️ Setup Workspace.

```jsonc
{
  "files.exclude": {
    "**/basic-infrastructure": true,
    "**/django-service-template": true,
    "**/*-service": true
  }
}
```

### `.vscode/launch.json`

All debug configurations in one central file. Every service appears in the Run & Debug dropdown (Ctrl+Shift+D).

### `.gitignore`

Every cloned repo is git-ignored. The `*-service/` wildcard automatically catches any service following the naming convention:

```gitignore
basic-infrastructure/
django-service-template/
*-service/
.DS_Store
.AppleDouble
.LSOverride
```

---

## Debugging

All debug configurations live in `engineering-infrastructure-hub/.vscode/launch.json`. Each service exposes a `debugpy` port starting at 5678, incrementing per service:

| Service | Debug Port |
|---|---|
| Service 01 | 5678 |
| Service 02 | 5679 |
| Service 03 | 5680 |
| ... | +1 per service |

A configuration entry:

```jsonc
{
  "name": "Backend: Remote Attach",
  "type": "debugpy",
  "request": "attach",
  "connect": {
      "host": "localhost",
      "port": 5678
  },
  "pathMappings": [
      {
          "localRoot": "${workspaceFolder:💾 Backend Service}/src",
          "remoteRoot": "/code"
      }
  ],
  "justMyCode": false
}
```

The `workspaceFolder` name must match the folder name in `eih.code-workspace` exactly (including the emoji prefix). The `pathMapping` maps the container's `/code` directory to your local source so breakpoints resolve correctly. The port must match the service's `DEBUG_PORT` in its `.env`.

---

## Testing

When attached to a container via Remote Explorer, the Python extension runs inside the container and discovers tests automatically. Test Explorer (Ctrl+Shift+T) shows all tests with per-test execution and parallelized runs.

The workflow: attach to container via Remote Explorer → open Test Explorer → tests appear → run individually or in bulk. Test output, coverage, and failures all display inline.

---

## Ngrok & Public Access

Ngrok tunnels a public HTTPS URL to your local Envoy Gateway on port 8080. This enables OAuth redirect flows (Keycloak needs a public URL for OIDC callbacks), mobile testing against the local stack, and remote demos.

**Free accounts** generate a new random URL every session. This means updating `DOMAIN` in environment files and Keycloak redirect URIs each restart. **Paid accounts** support a stable custom domain, removing this friction.

For solo development, the free tier works — just update the domain after each Ngrok restart.

---

## Runbooks

### Keycloak shows unhealthy after startup

Expected during the first 60–90 seconds. Wait and re-check: `docker compose ps`. If still unhealthy after 2 minutes: `docker compose logs keycloak`.

### Ngrok URL doesn't reach services

Verify Envoy Gateway is running: `docker compose ps gateway`. Confirm Ngrok tunnels to port 8080. Check that the Ngrok URL matches `DOMAIN` in environment files and Keycloak hostname configuration.

### "Port already in use"

Another process holds a required port. Find it: `sudo lsof -i :8080`. Kill it or adjust port mappings. Most common: a previous Ngrok session or stale container — `docker compose down --remove-orphans`.

### Service folder appears twice in Explorer

The service directory name doesn't end in `-service`, so the wildcard in `.vscode/settings.json` didn't catch it. Rename to follow the convention or add an explicit exclusion.

### Debugger won't attach

Verify the service container is running and `debugpy` is listening on the expected port. Confirm the port in `launch.json` matches the service's `DEBUG_PORT`. Confirm `pathMapping` uses the correct `workspaceFolder` name (must match the emoji-prefixed name exactly). Restart the service container.

### Login redirect fails after Keycloak restart

If the realm was reimported (`docker compose down -v`), verify `KEYCLOAK_CLIENT_SECRET` matches between Keycloak and your service `.env`. See [[22-keycloak-oidc|Keycloak OIDC]].

### Tests not appearing in Test Explorer

Ensure you're attached to the container via Remote Explorer, not running VS Code locally. The Python extension must be running inside the container to discover tests. Check the bottom-left corner of VS Code — it should show the container name.

---

## Clean Slate Files

Reference copies of workspace configuration files before any services have been generated. Use to reset if needed.

### `eih.code-workspace` (clean slate)

```jsonc
{
  "folders": [
    { "name": "🛠️ Setup Workspace", "path": "./" },
    { "name": "📦 Basic Infrastructure", "path": "basic-infrastructure/" },
    { "name": "--------------------", "path": "delimiter-0/" },
    { "name": "📚 Django Service Template", "path": "django-service-template/" },
    { "name": "--------------------", "path": "delimiter-1/" }
  ],
  "settings": {
    "scm.repositories.visible": 15,
    "git.openRepositoryInParentFolders": "always",
    "files.exclude": {
      "**/node_modules": true,
      "**/.git": true,
      "**/.DS_Store": true,
      "**/__pycache__": true,
      "**/*.pyc": true,
      "**/.pytest_cache": true,
      "**/delimiter-*": true
    },
    "python.terminal.activateEnvironment": true,
    "python.analysis.diagnosticSeverityOverrides": {
      "reportUnusedImport": "warning",
      "reportMissingImports": "error"
    }
  },
  "extensions": {
    "recommendations": [
      "ms-python.python",
      "ms-vscode-remote.remote-containers",
      "ms-azuretools.vscode-docker",
      "ms-python.black-formatter",
      "eamodio.gitlens"
    ]
  }
}
```

### `.vscode/launch.json` (clean slate)

```jsonc
{
  "version": "0.2.0",
  "configurations": []
}
```

### `.vscode/settings.json` (clean slate)

```jsonc
{
  "files.exclude": {
    "**/basic-infrastructure": true,
    "**/django-service-template": true,
    "**/*-service": true
  }
}
```

### `.gitignore` (clean slate)

```gitignore
basic-infrastructure/
django-service-template/
*-service/
.DS_Store
.AppleDouble
.LSOverride
```

---

## ADRs

- **ADR: Multi-root workspace over monorepo** — Each service is its own Git repository, cloned into the workspace directory and git-ignored. This preserves independent version history per service while providing a unified VS Code experience. The alternative (monorepo) would simplify cloning but complicate CI/CD, code ownership, and per-service deployment.
- **ADR: Centralized `launch.json` over per-service configs** — All debug configurations live in one file in the workspace root. This puts every service in the Run & Debug dropdown without switching context. Per-service `.vscode/launch.json` files would fragment the debugging experience across workspace roots.
- **ADR: `*-service` naming convention** — Directory suffix convention enables wildcard patterns in `.gitignore` and `settings.json`. New services require zero changes to workspace configuration files — only entries in the workspace folder list and launch config.
- **ADR: Container-attached development over local virtualenvs** — Developers work inside the same containers that run the application. The Python interpreter, dependencies, and environment are identical to production. Local virtualenvs would introduce environment drift.
- **ADR: Emoji-prefixed folder names** — Visual hierarchy in the Explorer sidebar. Infrastructure, template, and services are immediately distinguishable. The delimiter pattern provides grouping without nested folders.

---

## Onboarding Checklist

- [ ] Install prerequisites (Docker, Compose, Python, VS Code, Git, Ngrok)
- [ ] Clone workspace: `git clone git@github.com:yourorg/engineering-infrastructure-hub.git`
- [ ] Clone infrastructure and template into workspace directory
- [ ] Open `eih.code-workspace` in VS Code
- [ ] Install recommended extensions when prompted
- [ ] `docker compose up -d` in `basic-infrastructure/`
- [ ] Start Ngrok: `ngrok http 8080`
- [ ] Verify Keycloak admin console at `https://{ngrok-url}/auth/admin/`
- [ ] Verify MailDev web UI loads
- [ ] Generate a service from the template (name it `*-service`)
- [ ] Add service to compose include and Envoy route
- [ ] Add folder entry to `eih.code-workspace` and debug config to `launch.json`
- [ ] `docker compose up -d` → verify health endpoint through gateway
- [ ] Select service in Run & Debug → F5 → hit a breakpoint
- [ ] Attach to container via Remote Explorer → run tests via Test Explorer
- [ ] Open `docs/` in Obsidian and browse the Engineering Hub

---

## References

- [[01-infrastructure-engineering-hub|Infrastructure Engineering Hub]]
- [[21-envoy-gateway|Envoy Gateway]]
- [[22-keycloak-oidc|Keycloak OIDC]]
- [[20-postgresql|PostgreSQL]]
- [[23-maildev|MailDev]]
- [[30-django-service-template|Django Service Template]]
- [VS Code Multi-Root Workspaces](https://code.visualstudio.com/docs/editor/multi-root-workspaces)
- [Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers)
- [debugpy](https://github.com/microsoft/debugpy)
