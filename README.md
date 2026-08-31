# NGE Deployment Guide for Laravel

> [!IMPORTANT]
> **THIS GUIDE IS SPECIFICALLY FOR LARAVEL APPLICATIONS.** It assumes Laravel Artisan, Composer, a Vite/PNPM frontend, Laravel Horizon-style queue workers, Laravel's scheduler, and Laravel writable paths such as `storage/` and `bootstrap/cache/`.

This is an agent-oriented blueprint for adding a repeatable Docker deployment to a **Laravel project**. It deliberately contains generic Laravel examples, not a turnkey application. An implementation agent must inspect the target Laravel version and architecture, replace placeholders, ask the human for deployment and database choices, and validate the result before deployment.

It is not a generic PHP, Symfony, WordPress, Node.js, or framework-neutral Docker guide. Agents must not apply it unchanged to non-Laravel repositories.

## Expected Laravel project structure after implementation

This guide adds deployment files **to the root of an existing Laravel repository**. It does not create a separate deployment subproject. A resulting Laravel project may deliberately support full-source and registry deployment together, as `loket.pay` does.

```text
laravel-application/
├── app/                         # Existing Laravel application code
├── bootstrap/
│   └── cache/                   # Writable Laravel runtime cache
├── config/                      # Laravel configuration
├── database/                    # Migrations, factories, and seeders
├── public/                      # Public entry point and compiled assets
├── resources/                   # Blade/Vue/JS/CSS source
├── routes/                      # Laravel routes
├── storage/                     # Logs, uploads, sessions, framework state
├── tests/                       # Laravel tests
├── artisan
├── composer.json
├── composer.lock
├── package.json
├── pnpm-lock.yaml
│
├── nge                          # ONE interface for both deployment paths:
│                                # Compose, Artisan, Composer, PNPM, updates,
│                                # image build/push, migrations, health, reload
│
├── docker-compose.yml           # Full-source/server deployment: builds a thin
│                                # runtime image and bind-mounts ./:/var/www
├── docker-compose.local.yml     # Optional local/development Compose variant
│                                # with developer tooling and exposed DB/cache
│
├── Dockerfile                   # Registry build: multi-stage Composer + Vite
│                                # build producing a self-contained Laravel image
├── .dockerignore                # Excludes .env, Git data, dependencies, logs,
│                                # and runtime artifacts from image context
├── .env.example.docker          # Safe Docker/Laravel environment contract
│
├── .docker/
│   ├── image/
│   │   ├── Dockerfile           # Thin runtime image for source-mounted Compose
│   │   └── local/
│   │       ├── Dockerfile       # Optional development runtime image
│   │       └── conf.d/          # Optional PHP/Xdebug configuration
│   └── matter/
│       ├── gateway/
│       │   └── nginx.conf       # Nginx → Laravel core configuration
│       ├── psysh/               # Optional persistent PsySH configuration
│       └── sentry-relay/        # Optional Sentry Relay configuration
│
└── .github/
    └── workflows/
        ├── image.yml            # Build/push commit-tagged registry image
        └── deploy.yml           # Optional remote deployment using ./nge
```

Optional paths such as Xdebug, PsySH, Horizon, Redis, Sentry Relay, or frontend tooling should be generated only when the target Laravel application uses them.

### Hybrid deployment, like `loket.pay`

Full-source and registry deployment can coexist in one Laravel repository:

| File/path | Full-source responsibility | Registry responsibility |
|---|---|---|
| `nge` | Compose, Composer, PNPM, Artisan, source update, migration, and reload commands | Image build/push plus pull/deploy/rollback commands |
| `docker-compose.yml` | Builds a thin runtime and bind-mounts the Laravel checkout | A registry-specific variant references the published image and mounts only runtime state |
| `docker-compose.local.yml` | Local development with source mounts and developer services | Normally unused on a registry deployment host |
| `.docker/image/Dockerfile` | Small PHP runtime because source comes from the host | Not the final release artifact |
| root `Dockerfile` | Not required for ordinary source-mounted startup | Builds the self-contained Laravel release image with Composer dependencies and compiled frontend assets |
| `.docker/matter/` | Nginx and optional runtime/development configuration | External runtime configuration where appropriate |
| `.env.example.docker` | Laravel and Compose variables for source deployment | Registry/image/runtime variables when registry deployment is enabled |
| `.github/workflows/` | May invoke `./nge code:update` on remote source checkouts | Builds, tags, pushes, and deploys immutable images |

The critical difference is where application code comes from:

```yaml
# Full-source: code comes from the deployment host checkout
volumes:
  - ./:/var/www
```

```yaml
# Registry: code comes from the immutable image; only state is mounted
volumes:
  - ./runtime/storage:/var/www/storage
  - ./runtime/bootstrap-cache:/var/www/bootstrap/cache
```

Never add `./:/var/www` to registry deployment: it hides the application, dependencies, and compiled assets baked into the image.

### Target states the agent may generate

1. **Full-source only:** root `nge`, source-mounted Compose, thin `.docker/image/Dockerfile`, Nginx config, and `.env.example.docker`.
2. **Registry only:** root `nge`, immutable root `Dockerfile`, pull-only Compose, runtime-state mounts, registry workflow, Nginx config, and `.env.example.docker`.
3. **Hybrid:** both build paths in the same Laravel repository, with explicitly named Compose files and commands so source mounts cannot accidentally replace registry image contents.

For hybrid deployment, the agent must ask which Compose filename is authoritative for each environment and document exact commands for local development, source-server deployment, image build, registry deployment, and rollback.

## How this guide maps into the Laravel project

The `examples/` directories are agent reference material. They are not intended to remain as a parallel directory tree beside the Laravel application.

| Guide reference | Destination in the Laravel project |
|---|---|
| example `nge` scripts | Merge and adapt into root `./nge` |
| selected Compose example | Root `./docker-compose.yml`, or explicitly named by deployment mode |
| shared/registry `Dockerfile` | Root `./Dockerfile` for registry builds |
| thin source runtime image | `./.docker/image/Dockerfile` |
| local runtime image/config | `./.docker/image/local/` |
| gateway `nginx.conf` | `./.docker/matter/gateway/nginx.conf` |
| selected `.env.example.docker` | Root `./.env.example.docker` |
| registry workflow example | `./.github/workflows/image.yml` or equivalent |

Every path referenced by Compose must match the generated target tree.

### Relationship to `loket.pay`

This architecture was derived from the inspected `loket.pay` Laravel repository: source-mounted Compose workflows, a thin Compose runtime image, a separate multi-stage registry Dockerfile, one shared `nge` interface, `.docker/matter/` gateway configuration, and GitHub deployment automation. Project names, credentials, domains, and application-specific behavior are intentionally excluded.

## Structure of this guide repository

`examples/full-source/`, `examples/registry/`, and `examples/shared/` contain source material for implementing the target structure above. Each mode is complete enough to study independently; shared files identify common Laravel building blocks. Agents must not leave unresolved alternatives or conflicting example files in the target project.

### Why examples are duplicated

Each mode is a copyable reference. The final Laravel project may use one mode or an intentional hybrid, according to the human's choice.

## Deliverables

| Path | Purpose |
|---|---|
| `examples/full-source/` | Simple deployment: source code is present on the host and bind-mounted into runtime containers. |
| `examples/registry/` | Complex deployment: CI/build host publishes an immutable image; deployment host pulls it and mounts only runtime state/config. |
| `examples/shared/Dockerfile` | Generic multi-stage Laravel production image example. |
| `examples/shared/nginx.conf` | Generic gateway configuration. |
| `requirements/agent-implementation.md` | Mandatory agent workflow and acceptance criteria. |

## Intended agent interaction

This repository is an **implementation guide**, not a preselected deployment. When a human says:

> Implement `nge-guide` in my project.

or:

> Implement `nge-guide` in `<public-repository-name>`.

the agent must first confirm that the target is Laravel, inspect its structure, and then ask the human the unresolved architecture questions **before writing deployment files**. At minimum:

1. **Deployment model:** full-source/easy deployment or container-registry deployment?
2. **Database:** PostgreSQL or MySQL?
3. **Environment:** local/development, staging, production, or more than one?
4. **Registry details** (registry mode only): registry hostname, image namespace/name, target platform, authentication mechanism, and tagging policy.
5. **Runtime details:** public port/domain, TLS boundary, persistent upload paths, queue/scheduler requirements, and acceptable migration downtime.

The agent should recommend a default based on evidence from the project, but must not silently decide the deployment model or database. Questions whose answers depend on the selected deployment model should be asked after that selection.

Example first response after inspecting the target:

> I confirmed this is a Laravel application with an Artisan scheduler, a queue worker, and persistent uploads. Before I implement the deployment, please choose:
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
- If file-backed secrets are required, the implementing agent should add and test an explicit `VAR_FILE`/Docker-secrets integration rather than assuming it exists.

## Verification standard

A deployment is not complete because `docker compose up -d` returned zero. It must prove:

- Compose interpolation/config validation succeeds;
- the generated Laravel deployment contains only the human-approved database implementation;
- containers become healthy and remain running;
- the HTTP health endpoint responds successfully;
- a migration runs successfully and only once per release;
- queue and scheduler processes use the same release image;
- writable bind mounts retain data across recreation;
- rollback to the previous immutable image tag works in registry mode;
- no project-specific names or secrets leaked into reusable templates.

See the full checklist in `requirements/agent-implementation.md`.

## Repository

This guide is maintained in the private repository `digital-entropy/nge-guide`.
