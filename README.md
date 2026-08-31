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

## Intended agent interaction

This repository is an **implementation guide**, not a preselected deployment. When a human says:

> Implement `nge-guide` in my project.

or:

> Implement `nge-guide` in `<public-repository-name>`.

the agent must inspect the project and then ask the human the unresolved architecture questions **before writing deployment files**. At minimum:

1. **Deployment model:** full-source/easy deployment or container-registry deployment?
2. **Database:** PostgreSQL or MySQL?
3. **Environment:** local/development, staging, production, or more than one?
4. **Registry details** (registry mode only): registry hostname, image namespace/name, target platform, authentication mechanism, and tagging policy.
5. **Runtime details:** public port/domain, TLS boundary, persistent upload paths, queue/scheduler requirements, and acceptable migration downtime.

The agent should recommend a default based on evidence from the project, but must not silently decide the deployment model or database. Questions whose answers depend on the selected deployment model should be asked after that selection.

Example first response after inspecting the target:

> I found a PHP application with a queue worker and persistent uploads. Before I implement the deployment, please choose:
>
> 1. Full-source/easy deployment, or container-registry deployment?
> 2. PostgreSQL, or MySQL?
> 3. Which environment and public domain/port is this for?
>
> If you choose registry deployment, I will then ask for the registry and immutable image-tag details.

After receiving the answers, the agent implements only the chosen production database service and adapts `nge`, Compose, Dockerfile, environment example, and deployment documentation accordingly. The PostgreSQL/MySQL sections in this repository are reference stubs for the agent—not a runtime database selector that must be shipped to humans.

## Choose one deployment mode

### 1. Full-source deployment (simple)

Use when the deployment host is allowed to contain a Git checkout and build dependencies.

Flow:

1. Clone/pull the application on the deployment host.
2. Copy `.env.example.docker` to `.env` and fill secrets locally.
3. Have the implementation agent replace the database reference stubs with the chosen database service.
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

## Database implementation is intentionally deferred

The examples show both database services as clearly marked reference stubs. During implementation, the agent asks which database the human will use, removes the unused service/profile/helper, and sets the framework connection values in `.env.example.docker`:

| Setting | PostgreSQL | MySQL |
|---|---|---|
| `DB_CONNECTION` | `pgsql` | `mysql` |
| `DB_HOST` | `postgres` | `mysql` |
| `DB_PORT` | `5432` | `3306` |

The generated project deployment should not make operators choose a database on every run. It should contain one approved database implementation. Database migration/conversion is outside this blueprint and must be decided before production cutover.

## Agent quick start

1. Read `requirements/agent-implementation.md` completely.
2. Copy **one** example directory into the target project/deployment repository.
3. Copy `examples/shared/Dockerfile` and `examples/shared/nginx.conf` to paths referenced by that example.
4. Rename generic image/user/ports only when required by the target.
5. Ask the human to choose PostgreSQL or MySQL, then remove the unused reference stub and its `nge` helper.
6. Copy `.env.example.docker` to `.env`; never commit `.env`.
7. Run:

```bash
chmod +x nge
./nge doctor
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
