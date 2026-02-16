---
title: "PostgreSQL"
created: "2026-02-15"
updated: "2026-02-15"
version: 1.0.0
status: published
type: reference
owner: "platform-team"
lifecycle: production
tags: [service, postgresql, database, persistence]
---
# PostgreSQL — Shared Database Instance

Single PostgreSQL instance providing per-service logical databases for all platform services. Initialized via a startup script that creates databases dynamically from an environment variable. Each Django service and Keycloak get their own database within the same instance — logical isolation without operational overhead.

> 📍 **Type:** Service Reference
> 👤 **Owner:** Ktwenty Threel
> 🎯 **Outcome:** Understand the PostgreSQL Setup

---

## Table of Contents

- [[#Overview]]
- [[#Architecture]]
- [[#Design Decisions]]
- [[#Dependencies]]
- [[#Configuration]]
- [[#Dev ↔ Prod Differences]]
- [[#Kubernetes Migration Path]]
- [[#Monitoring]]
- [[#Runbooks]]
- [[#ADRs]]
- [[#References]]

---

## Overview

PostgreSQL runs as a single container (`db`) on the `internal` Docker network. It is not exposed to the internet — only other containers on the same network can connect. In development, port 5432 is forwarded to the host for use with local database tools (pgAdmin, DBeaver, DataGrip). In production, no port is exposed.

### Connection Model

All services connect to PostgreSQL using the Docker internal hostname `db` on port 5432. Each service uses its own database within the shared instance:

| Database   | Consumer               | Connection String                    |
| ---------- | ---------------------- | ------------------------------------ |
| `keycloak` | Keycloak IdP           | `jdbc:postgresql://db:5432/keycloak` |
| `backend`  | Django Backend service | `postgresql://db:5432/backend`       |

Additional databases are created automatically by adding to the `POSTGRES_MULTIPLE_DATABASES` environment variable. The init script runs on first startup only — when the data volume is empty.

### Database Initialization

A single shell script (`init-multiple-dbs.sh`) mounted into `/docker-entrypoint-initdb.d/` reads the `POSTGRES_MULTIPLE_DATABASES` environment variable, splits it by comma, and creates each database if it doesn't already exist. This is the standard PostgreSQL Docker entrypoint hook mechanism — scripts in that directory run once during initial database cluster creation.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        internal network                          │
│                                                                  │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│   │  Keycloak    │    │  Django      │    │  Django      │       │
│   │              │    │  Service A   │    │  Service N   │       │
│   │  keycloak db │    │  backend db  │    │  ... db      │       │
│   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘       │
│          │                   │                   │               │
│          │  jdbc:postgresql  │  psycopg          │               │
│          │  ://db:5432       │  ://db:5432       │               │
│          │                   │                   │               │
│          └─────────┐         │         ┌─────────┘               │
│                    │         │         │                         │
│                    ▼         ▼         ▼                         │
│              ┌────────────────────────────┐                      │
│              │       PostgreSQL           │                      │
│              │  ───────────────────────   │                      │
│              │  postgres:17-alpine        │                      │
│              │  ───────────────────────   │                      │
│              │  keycloak │ backend │ ...  │                      │
│              │  (per-service databases)   │                      │
│              └─────────────┬──────────────┘                      │
│                            │                                     │
│                  ┌─────────┴─────────┐                           │
│                  │   db_data volume  │                           │
│                  │   (named volume)  │                           │
│                  └───────────────────┘                           │
│                                                                  │
│   Dev only: port 5432 exposed to host for pgAdmin/DBeaver        │
└──────────────────────────────────────────────────────────────────┘
```

PostgreSQL sits exclusively on the `internal` Docker network. It has no gateway exposure and no public route — this is intentional. Database connections are container-to-container over the Docker bridge.

For the full platform architecture, see [[infrastructure-engineering-hub#Architecture|Infrastructure Engineering Hub — Architecture]].

---

## Design Decisions

### Shared Instance with Per-Service Databases

| | Shared Instance | Dedicated Instance per Service |
|---|---|---|
| **Choice** | Single PostgreSQL, multiple databases | — |

A single PostgreSQL instance serves all platform services. Each service gets its own database (created via the init script), providing logical isolation without the operational overhead of running multiple database containers. This is the standard pattern for monolithic-to-microservice transition — it works well when services don't need independent scaling or version pinning of their database engine.

The tradeoff is that all services share connection limits, memory, and I/O. The production tuning parameters (`max_connections=200`, `shared_buffers=512MB`) are sized for a single VPS hosting a handful of services.

> [!important] Security: shared instance isolation
> In this setup, the `postgres` superuser has access to all databases. A SQL injection in one Django service could theoretically access another service's database. For the Dev and VPS tiers this is accepted — all services are operated by the same team. For the Kubernetes tier, the recommendation in [[infrastructure-engineering-hub#Scaling Path|Scaling Path]] is to move to a managed service (RDS, CloudSQL) with per-service credentials scoped to individual databases. Alternatively, create dedicated PostgreSQL roles per service with `GRANT` restricted to their own database.

### PostgreSQL Version

| | Current | Target |
|---|---|---|
| **Image** | `postgres:17-alpine` | — |

Previously running 14.15-alpine. Upgraded to 17-alpine for extended support runway. PostgreSQL 14 reaches end-of-life November 2026 (5 years from its September 2021 release). PostgreSQL 17, released September 2024, is supported through September 2029. Keycloak 26.5 requires PostgreSQL 13 as a minimum — 17 provides comfortable headroom.

The Alpine variant is used for smaller image size (~80MB vs ~400MB for Debian). This has no functional impact — the PostgreSQL binary is identical.

> [!important] Major version upgrades require data migration
> PostgreSQL major versions use incompatible on-disk data formats. You cannot simply change the image tag on a running instance with existing data. For dev environments, `docker compose down -v && docker compose up -d` recreates everything from the init script. For production with persistent data, the upgrade requires `pg_dumpall` → stop old → remove/rename volume → start new → restore. See [[#Runbooks]] for the procedure. The `pgautoupgrade/docker-pgautoupgrade` image can automate this with `pg_upgrade --link` but is a community PoC — use with caution.

### Init Script over ORM-Managed Creation

| | Init Script | Django `migrate` creates DBs |
|---|---|---|
| **Choice** | Shell script in `docker-entrypoint-initdb.d/` | — |

Databases are created by a shell script at container first-start, before any service connects. This decouples database provisioning from application code — Keycloak (which is not a Django app) needs its database to exist before it can start. The init script approach is standard Docker PostgreSQL practice and avoids requiring `CREATE DATABASE` privileges in application connection strings.

---

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| Docker named volume (`db_data`) | Infrastructure | Persistent storage for database files |
| `POSTGRES_USER`, `POSTGRES_PASSWORD` | Environment | Superuser credentials (from `.env`) |
| `POSTGRES_MULTIPLE_DATABASES` | Environment | Comma-separated list of databases to create |

PostgreSQL has no upstream service dependencies. It is the first service to start and the dependency target for Keycloak and all Django services (`depends_on: db: condition: service_healthy`).

---

## Configuration

### Init Script

The init script is mounted read-only from the host into the standard PostgreSQL Docker entrypoint directory:

```
services_configs/postgres/init-multiple-dbs.sh
  → /docker-entrypoint-initdb.d/init-multiple-dbs.sh:ro
```

```bash
#!/bin/bash
set -e

function create_database() {
    local database=$1
    echo "Creating database '$database'"
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
        SELECT 'CREATE DATABASE $database'
        WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = '$database')\gexec
EOSQL
}

if [ -n "$POSTGRES_MULTIPLE_DATABASES" ]; then
    echo "Multiple database creation requested: $POSTGRES_MULTIPLE_DATABASES"
    for db in $(echo $POSTGRES_MULTIPLE_DATABASES | tr ',' ' '); do
        create_database $db
    done
    echo "Multiple databases created"
fi
```

The script uses `\gexec` to conditionally execute `CREATE DATABASE` — this makes it idempotent. If the database already exists, the `WHERE NOT EXISTS` clause returns nothing and no statement is executed. The `ON_ERROR_STOP=1` flag ensures any unexpected error halts the script.

To add a new service database: add its name to `POSTGRES_MULTIPLE_DATABASES` in `.env` (e.g., `POSTGRES_MULTIPLE_DATABASES=keycloak,backend,newservice`) and recreate the container with an empty volume. On an existing instance, you can also `docker compose exec db psql -U postgres -c "CREATE DATABASE newservice"`.

### Environment Variables

| Variable                      | Default          | Purpose                                                                                   |
| ----------------------------- | ---------------- | ----------------------------------------------------------------------------------------- |
| `POSTGRES_USER`               | `postgres`       | Superuser name. Shared across all services via `.env`.                                    |
| `POSTGRES_PASSWORD`           | `postgres` (dev) | Superuser password. Dev uses plaintext in `.env`. Prod uses `.env` file outside the repo. |
| `POSTGRES_MULTIPLE_DATABASES` | —                | Comma-separated database names: `keycloak,backend`                                        |

### Production Tuning

In production, PostgreSQL is started with explicit tuning flags via the `command` override:

```yaml
command: >
    postgres
    -c max_connections=200
    -c shared_buffers=512MB
    -c effective_cache_size=1536MB
    -c maintenance_work_mem=128MB
    -c work_mem=4MB
    -c max_locks_per_transaction=256
```

| Parameter | Value | Rationale |
|---|---|---|
| `max_connections` | 200 | Keycloak (up to ~50 connections at load) + Django services + connection overhead. Default of 100 is too low for multi-service. |
| `shared_buffers` | 512MB | Standard recommendation: ~25% of available RAM. Sized for a 2GB container limit. |
| `effective_cache_size` | 1536MB | Tells the query planner how much memory is available for caching (shared_buffers + OS cache). ~75% of container memory. |
| `maintenance_work_mem` | 128MB | Memory for `VACUUM`, `CREATE INDEX`, `ALTER TABLE ADD FOREIGN KEY`. Higher than default (64MB) for faster maintenance on larger tables. |
| `work_mem` | 4MB | Per-operation sort/hash memory. Kept low because it multiplies by active connections (200 × 4MB = 800MB worst case). |
| `max_locks_per_transaction` | 256 | Keycloak runs migrations with many concurrent lock objects. Default of 64 can cause "out of shared memory" errors during Keycloak upgrades. |

These are passed as command-line flags rather than a `postgresql.conf` file for visibility — all tuning is visible directly in the compose file without needing to inspect a mounted config.

### Health Check

```yaml
healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
    interval: 10s
    timeout: 5s
    retries: 5
```

`pg_isready` verifies that the PostgreSQL server is accepting connections. It's a lightweight binary check — it does not run a query. This is sufficient for Docker Compose `depends_on: condition: service_healthy` to gate service startup.

### Docker Compose Service Block (Prod)

```yaml
db:
    image: postgres:17-alpine
    env_file:
        - .env
    restart: unless-stopped
    command: >
        postgres
        -c max_connections=200
        -c shared_buffers=512MB
        -c effective_cache_size=1536MB
        -c maintenance_work_mem=128MB
        -c work_mem=4MB
        -c max_locks_per_transaction=256
    volumes:
        - ./services_configs/postgres/init-multiple-dbs.sh:/docker-entrypoint-initdb.d/init-multiple-dbs.sh:ro
        - db_data:/var/lib/postgresql/data
    healthcheck:
        test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
        interval: 10s
        timeout: 5s
        retries: 5
    deploy:
        resources:
            limits:
                memory: 1536M
            reservations:
                memory: 512M
    logging:
        driver: json-file
        options:
            max-size: "100m"
            max-file: "3"
    networks:
        - internal
```

### Docker Compose Service Block (Dev)

```yaml
db:
    image: postgres:17-alpine
    hostname: db.local
    restart: unless-stopped
    env_file:
        - .env
    environment:
        - POSTGRES_USER=${POSTGRES_USER:-postgres}
        - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:-postgres}
    volumes:
        - ./services_configs/postgres/init-multiple-dbs.sh:/docker-entrypoint-initdb.d/init-multiple-dbs.sh:ro
        - db_data:/var/lib/postgresql/data
    ports:
        - "5432:5432"
    healthcheck:
        test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
        interval: 10s
        timeout: 5s
        retries: 5
    networks:
        - internal
```

### Volume Names

| Environment | Volume Name | Purpose |
|---|---|---|
| Dev | `eih-dev-db-data` | Local development data. Safe to destroy with `docker compose down -v`. |
| Prod | `eih-shared-db-data` | Production data. Persistent across restarts. Back up before any destructive operation. |

---

## Dev ↔ Prod Differences

| Aspect | Dev | Prod |
|---|---|---|
| Image | `postgres:17-alpine` | `postgres:17-alpine` |
| Port exposure | `5432:5432` (host access for DB tools) | No port exposed |
| Hostname | `db.local` | `db` (default) |
| Tuning | Default PostgreSQL settings | Custom `command` flags (see [[#Production Tuning]]) |
| Resource limits | None | Memory: 1536M limit, 512M reserved |
| Logging | Docker default | `json-file`, 100MB max, 3 files rotated |
| Secrets | `.env` with plaintext defaults | `.env` file outside repo |
| Volume name | `eih-dev-db-data` | `eih-shared-db-data` |
| Data lifecycle | Expendable — recreated from init script | Persistent — requires backup before destructive ops |

> [!tip] K8s migration
> At the Kubernetes tier, PostgreSQL moves to a managed service (RDS, CloudSQL, Azure Database). Django services and Keycloak change their connection strings from `db:5432` to the managed instance endpoint. The init script is replaced by Terraform/IaC database provisioning. See [[infrastructure-engineering-hub#Scaling Path|Scaling Path]] for the full transition map.

---

## Kubernetes Migration Path

| Aspect | Docker Compose (current) | Kubernetes |
|---|---|---|
| Instance | Single container, shared | Managed service (RDS, CloudSQL) |
| Database creation | `init-multiple-dbs.sh` | Terraform / IaC provisioning |
| Credentials | `.env` file | K8s Secrets / Vault |
| Connection string | `db:5432` | Managed endpoint (e.g., `mydb.abc123.us-east-1.rds.amazonaws.com:5432`) |
| Per-service isolation | Shared superuser | Dedicated IAM roles / PostgreSQL roles per service |
| Backups | Manual `pg_dump` | Automated managed snapshots |
| Scaling | Single instance, vertical only | Read replicas, connection pooling (PgBouncer) |
| Monitoring | Docker logs, manual queries | CloudWatch / Cloud Monitoring + pg_stat_statements |

---

## Monitoring

### Dev

Logs are the primary monitoring tool: `docker compose logs -f db`. Connect directly with `psql -h localhost -U postgres` or any GUI tool on port 5432 to inspect data.

### Prod

Logs are available via `docker compose logs -f db` with `json-file` driver, capped at 300MB total (3 × 100MB). For active monitoring, connect via `docker compose exec db psql -U postgres` and inspect `pg_stat_activity` for active connections, `pg_stat_user_tables` for table health, and `pg_stat_database` for per-database I/O.

Key queries for operational health:

```sql
-- Active connections per database
SELECT datname, count(*) FROM pg_stat_activity GROUP BY datname;

-- Long-running queries (>5s)
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle' AND (now() - pg_stat_activity.query_start) > interval '5 seconds';

-- Database sizes
SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database WHERE datistemplate = false;
```

---

## Runbooks

### Container fails to start — "database files are incompatible"

This happens after changing the major version tag (e.g., 14→17) without migrating data. The existing volume contains data in the old format. If this is dev: `docker compose down -v && docker compose up -d` — the init script recreates everything. If this is prod: restore from backup to a fresh volume (see [[#Major Version Upgrade (Prod)]]).

### Init script didn't create databases

The init script only runs when the data volume is empty (first initialization). If you added a new database to `POSTGRES_MULTIPLE_DATABASES` after the first run, the script won't re-execute. Create it manually: `docker compose exec db psql -U postgres -c "CREATE DATABASE newservice"`.

### Keycloak fails with "Connection refused" to db:5432

PostgreSQL isn't healthy yet. Check `docker compose ps db` — if health is "starting", Keycloak's `depends_on: condition: service_healthy` should prevent this. If the healthcheck is passing but Keycloak still fails, verify `KC_DB_URL` uses `db` (not `localhost`) and that both containers are on the `internal` network.

### Out of shared memory during Keycloak migration

Keycloak upgrades run complex migrations that acquire many locks. If `max_locks_per_transaction` is too low (default: 64), PostgreSQL throws "out of shared memory" errors. The production config sets this to 256. For dev, either add the same flag or restart Keycloak to retry the migration (it's idempotent).

### Disk space running low on volume

Check database sizes with `docker compose exec db psql -U postgres -c "SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database WHERE datistemplate = false;"`. Run `VACUUM FULL` on bloated tables if needed (causes table lock — schedule during low traffic). For persistent growth, resize the VPS disk.

### Major Version Upgrade (Prod)

This procedure upgrades PostgreSQL across major versions on a running production instance. Expect 5–15 minutes of downtime depending on database size.

**1. Back up:** `docker compose exec db pg_dumpall -U postgres > backup_$(date +%Y%m%d).sql`

**2. Verify backup:** Check that the dump file is non-empty and contains `CREATE DATABASE` statements.

**3. Stop all services:** `docker compose down`

**4. Rename old volume:** `docker volume create eih-shared-db-data-old && docker run --rm -v eih-shared-db-data:/from -v eih-shared-db-data-old:/to alpine sh -c "cp -a /from/. /to/"` (safety copy).

**5. Remove old volume:** `docker volume rm eih-shared-db-data`

**6. Update image tag** in `compose.prod.yaml` to the new version.

**7. Start PostgreSQL only:** `docker compose up -d db` — wait for healthy.

**8. Restore:** `cat backup_*.sql | docker compose exec -T db psql -U postgres`

**9. Verify:** Connect and spot-check tables, row counts, Keycloak login.

**10. Start remaining services:** `docker compose up -d`

**11. Clean up:** Once verified, remove the safety copy volume.

---

## ADRs

- **ADR: Shared instance with per-service databases** — Single PostgreSQL container with multiple logical databases created via init script. Simpler to operate than multiple instances for Dev and VPS tiers. Per-service roles with restricted grants recommended when moving to Kubernetes managed service.
- **ADR: Init script for database provisioning** — Shell script in `docker-entrypoint-initdb.d/` creates databases from `POSTGRES_MULTIPLE_DATABASES` env var. Decouples database creation from application code. Keycloak needs its database before it can run migrations — application-level creation would create a circular dependency.
- **ADR: Alpine image variant** — Smaller image size (~80MB vs ~400MB Debian), identical PostgreSQL binary. No functional difference for this workload. No extensions requiring Debian-specific libraries are used.
- **ADR: Command-line tuning over postgresql.conf** — Production tuning parameters passed as `-c` flags in the compose `command` directive. All tuning visible in a single file without inspecting mounted configs. Easier to diff between environments.
- **ADR: PostgreSQL 17 over 14** — Upgraded from 14.15-alpine to 17-alpine. PostgreSQL 14 EOL is November 2026; 17 extends support to September 2029. Keycloak 26.5 raised the floor to PostgreSQL 13. Upgrade was performed via dump/restore (dev: destroy volume and recreate; prod: `pg_dumpall` → fresh volume → restore). No application code changes required — Django and Keycloak connection strings are version-agnostic.

---

## References

- [PostgreSQL Docker Official Image](https://hub.docker.com/_/postgres)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/17/)
- [PostgreSQL Versioning Policy](https://www.postgresql.org/support/versioning/)
- [docker-entrypoint-initdb.d Mechanism](https://github.com/docker-library/docs/blob/master/postgres/README.md#initialization-scripts)
- [pgautoupgrade — Automated Docker Upgrades](https://github.com/pgautoupgrade/docker-pgautoupgrade)
- [[infrastructure-engineering-hub|Infrastructure Engineering Hub]]
- [[service-keycloak|Keycloak Configuration]]
- [[service-envoy-gateway|Envoy Gateway Configuration]]
- [[database-schemas|Schema Reference]]
- [[networking-and-ports|Networking & Ports Reference]]