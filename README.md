# NGE Deployment Guide

Agent-oriented blueprint for adding a repeatable Docker deployment to a PHP/Laravel-style application.

This repository deliberately contains **generic examples**, not a turnkey application. An implementation agent must inspect the target application, replace placeholders, select a database, and validate the result before deployment.

## Deliverables

| Path | Purpose |
|---|---|
| `examples/full-source/` | Simple deployment: source code is present on the host and bind-mounted into runtime containers. |
| `examples/registry/` | Complex deployment: CI/build host publishes an immutable image; deployment host pulls it and mounts only runtime state/config. |
| `examples/shared/Dockerfile` | Generic multi-stage production image example. |
| `examples/shared/nginx.conf` | Generic gateway configuration. |
| `requirements/agent-implementation.md` | Mandatory agent workflow and acceptance criteria. |

## Choose one deployment mode

### 1. Full-source deployment (simple)

Use when the deployment host is allowed to contain a Git checkout and build dependencies.

Flow:

1. Clone/pull the application on the deployment host.
2. Copy `.env.example.docker` to `.env` and fill secrets locally.
3. Select one DB profile: `postgres` or `mysql`.
4. Build the local application image.
5. Bind-mount the checkout at `/var/www` into `core`, `gateway`, workers, and one-shot commands.
6. Run dependency installation, asset build, migrations, and service reload through `./nge`.

Advantages: easy to inspect and repair. Trade-offs: mutable host checkout, slower/repeated host-side build, and a broader production filesystem surface.

### 2. Registry deployment (complex)

Use when the runtime host should receive a built artifact rather than application source.

Flow:

1. CI/build host builds `REGISTRY/IMAGE_NAME:IMAGE_TAG` from the application commit.
2. CI authenticates to the registry and pushes the image.
3. Runtime host keeps only Compose, `nge`, `.env`, gateway config, and persistent directories/volumes.
4. Runtime host pulls the exact tag (preferably a commit SHA or digest), recreates services, migrates once, and verifies health.
5. `core`, `horizon`, and `scheduler` all use the exact same image reference.

The example uses manual bind mounts for application-owned runtime state:

- `./runtime/storage:/var/www/storage`
- `./runtime/bootstrap-cache:/var/www/bootstrap/cache`

Source code and built frontend assets remain inside the image. Do **not** mount `./:/var/www` in registry mode, because that would hide the image contents.

## Database selection is intentionally deferred

Both Compose examples provide mutually exclusive profiles:

```bash
./nge db:select postgres
# or
./nge db:select mysql
```

The selection sets `COMPOSE_PROFILES` for subsequent commands. The implementation agent must also set the framework connection values in `.env`:

| Setting | PostgreSQL | MySQL |
|---|---|---|
| `DB_CONNECTION` | `pgsql` | `mysql` |
| `DB_HOST` | `postgres` | `mysql` |
| `DB_PORT` | `5432` | `3306` |

Never enable both database profiles against the same application deployment. Database migration/conversion is outside this blueprint and must be decided before production cutover.

## Agent quick start

1. Read `requirements/agent-implementation.md` completely.
2. Copy **one** example directory into the target project/deployment repository.
3. Copy `examples/shared/Dockerfile` and `examples/shared/nginx.conf` to paths referenced by that example.
4. Rename generic image/user/ports only when required by the target.
5. Copy `.env.example.docker` to `.env`; never commit `.env`.
6. Run:

```bash
chmod +x nge
./nge doctor
./nge db:select postgres   # or mysql, after a decision
./nge config
./nge up
./nge status
./nge health
```

For registry mode, build/push first:

```bash
./nge image:build --tag "$(git rev-parse --short=12 HEAD)"
./nge image:push --tag "$(git rev-parse --short=12 HEAD)"
# On deployment host, set IMAGE_TAG to that immutable tag:
./nge deploy
```

## Required customization points

Before an implementation is considered complete, an agent must resolve:

- framework/runtime and required PHP extensions;
- web process health endpoint;
- queue and scheduler commands;
- writable directories and ownership;
- registry host, namespace, authentication, retention, and immutable tag policy;
- selected database and backup/restore procedure;
- Redis requirement and persistence policy;
- TLS/reverse-proxy boundary;
- secret delivery method;
- migration downtime and rollback strategy;
- deployment host architecture and image platform.

## Secrets and environment files

- `.env.example.docker` contains names and safe placeholders only.
- `.env` is runtime-only and ignored by Git.
- Never bake `.env`, registry credentials, SSH keys, or production certificates into an image.
- Prefer secret files or the deployment platform's secret store where available.
- `nge` supports `VAR_FILE` through its `file_env` helper for selected values an implementation chooses to wire in.

## Verification standard

A deployment is not complete because `docker compose up -d` returned zero. It must prove:

- Compose interpolation/config validation succeeds;
- exactly one database profile is active;
- containers become healthy and remain running;
- the HTTP health endpoint responds successfully;
- a migration runs successfully and only once per release;
- queue and scheduler processes use the same release image;
- writable bind mounts retain data across recreation;
- rollback to the previous immutable image tag works in registry mode;
- no project-specific names or secrets leaked into reusable templates.

See the full checklist in `requirements/agent-implementation.md`.

## Repository status

The intended future remote is a **private** repository named `digital-entropy/nge-guide`. This local repository does not create or push that remote until explicitly approved.
