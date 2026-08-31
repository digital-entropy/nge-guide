# Laravel agent implementation requirements

> **Scope: Laravel with Octane + Swoole.** Confirm the target repository is Laravel, then automatically install and configure Laravel Octane with the Swoole server. Octane + Swoole is an invariant of this guide, not a human-choice question. The examples also assume `artisan`, Composer, Laravel migrations, Laravel scheduler commands, and Laravel writable directories. Stop and tell the human if the target is not Laravel.

## Mission

Adapt this blueprint to a target Laravel application while preserving two supported deployment patterns: full-source host deployment and registry artifact deployment. Do not blindly copy values from the examples.

## Mandatory human interview

When asked to “implement nge-guide” (or equivalent), do not immediately copy the examples. First inspect the target project, summarize what can be inferred, and ask the human to decide:

1. **Full-source/easy**, **container-registry/complex**, or a deliberate **hybrid** like `loket.pay` that supports both from one Laravel repository.
2. **PostgreSQL** or **MySQL**.
3. Whether the application needs queue processing and Laravel Horizon. Preserve an existing Horizon setup unless changes are approved.
4. Target environment(s), domain/port, and TLS termination point.
5. Required scheduler, cache, uploads, and other persistent paths.
6. Migration downtime tolerance and rollback expectations.
7. For registry mode: registry coordinates, credentials mechanism, build platform, and immutable tag convention.

Recommend an option when project evidence supports it, but identify it as a recommendation. Do not silently choose the deployment model or database. Ask dependent registry questions only if registry mode is selected.

Once answered, generate a concrete deployment for those choices. Remove unselected alternatives from the target project rather than exposing architecture selection as a routine operator command.

Do **not** ask which application runtime to use. Install and configure Laravel Octane with Swoole automatically.

## Mandatory Octane + Swoole implementation

1. Inspect Laravel/PHP constraints and existing `laravel/octane` state.
2. Install a compatible `laravel/octane` when absent and configure the Swoole server noninteractively.
3. Use a container runtime with a compatible Swoole PHP extension; prove the loaded extension and version.
4. Make the core command execute `artisan octane:start --server=swoole`, bind to `0.0.0.0`, and expose a deliberate internal port.
5. Align Nginx upstream and health checks with that internal Octane port.
6. Review the Laravel application for long-lived-worker hazards and add appropriate reset/termination behavior.
7. Keep migrations and other one-shot deployment work outside the Octane process start command.
8. Test boot, gateway HTTP, health, restart behavior, and repeated requests for state leakage.
9. Never silently fall back to another PHP application server.

## Horizon decision and implementation

Ask the human whether queue processing/Horizon is required. If approved, install or preserve a compatible Horizon setup, use the same release image as core, configure Redis/queues/supervisors, and prove a queued smoke job is processed. If Horizon is not required, remove its service and commands. If queues are required without Horizon, ask for the desired worker strategy.

## Discovery before editing

The agent must inspect and record:

1. Laravel, PHP, Composer, Node.js, and package-manager versions and lock files.
2. Existing start command, queue worker, scheduler, and health endpoint.
3. Required PHP extensions and OS packages.
4. Existing writable paths, uploads, generated files, and symlinks.
5. Current DB driver and whether PostgreSQL or MySQL has been approved.
6. Existing reverse proxy, TLS termination, ports, networks, and DNS.
7. CPU architecture of build and runtime hosts.
8. Current release and rollback process.
9. Registry coordinates and authentication mechanism.
10. Secret source without printing secret values.

If any item changes architecture or data safety, ask the operator before implementation.

## Invariants

- `.env` is never committed, copied into an image, printed, or logged.
- Production image is self-contained except explicitly documented persistent/runtime mounts.
- All long-running application services use one exact image tag/digest per release.
- Registry tags used for production are immutable; `latest` is not a release identifier.
- The implemented target contains only the approved PostgreSQL or MySQL deployment path; the unused reference stub and CLI helper are removed.
- DB and cache ports are not publicly published by default.
- Runtime processes use a non-root user where supported.
- Logs are bounded by rotation settings.
- Deploy is serialized and failure stops the sequence.
- Migration happens after dependency readiness and before final health acceptance.
- Rollback never automatically reverses a destructive database migration.

## Full-source implementation

- Keep a Git checkout on the deployment host.
- Build an application runtime image locally.
- Bind-mount source into every service that executes application code.
- Decide whether dependencies and assets live on the host checkout or isolated named volumes; document the choice.
- Ensure host UID/GID matches the runtime user and writable directories.
- `code:update` must fetch/pull only the intended branch, install locked dependencies, build assets, migrate with explicit production flags, clear caches, and reload services.
- Preserve local changes detection: refuse deployment from a dirty checkout unless explicitly overridden.

## Registry implementation

- The Dockerfile must install locked production dependencies and compile frontend assets in build stages.
- `.dockerignore` must exclude `.git`, `.env*` except safe examples, tests if appropriate, local caches, dependency directories, and runtime data.
- Tag every image with a commit SHA or release version. Optionally publish a human alias in addition, never instead.
- CI/build host logs into the registry using scoped credentials; runtime host needs pull-only credentials.
- Runtime Compose must not bind-mount the source checkout over `/var/www`.
- Mount only data that must survive image recreation (for example storage/uploads and framework cache where required).
- `deploy` must pull, validate config, recreate, migrate once, verify health, and retain the previous image identifier for rollback.
- If multiple replicas are introduced, migrations require a separate one-shot job/lock.

## Database reference stubs

Both stubs exist here only to guide implementation. Ask the human which database will be used, implement that choice, and remove the other database service, volume, profile, environment placeholders, health check, and `nge` CLI helper from the target project.

### PostgreSQL

- Service name: `postgres`
- Container port: `5432`
- Persistent target must match the selected major image's documented data directory.
- Add `pg_isready` health check.
- CLI helper uses `psql` without exposing passwords in process output where practical.

### MySQL

- Service name: `mysql`
- Container port: `3306`
- Persistent target: `/var/lib/mysql`
- Add `mysqladmin ping` health check.
- Decide charset/collation and SQL modes based on application compatibility.

Before cutover, remove ambiguity from production operations: document the chosen database, driver, backup command, restore drill, and migration procedure. Database choice must be resolved during agent implementation, not exposed as a routine operator command.

## Required tests

- `bash -n nge`
- Shell lint with `shellcheck` when available.
- `docker compose --env-file .env.example.docker config` for each DB profile and mode.
- Build the relevant image on the actual target architecture or via declared buildx platforms.
- Start stack with disposable credentials/data.
- Wait for health with a bounded timeout.
- Issue HTTP request to health endpoint.
- Run framework smoke tests and a non-destructive DB query.
- Recreate services and prove persistent paths retain a sentinel file/test row.
- Registry mode: deploy two distinct tags and roll back to the first.
- Inspect effective mounts to prove source is mounted only in full-source mode.

## Handoff report

Report exact paths, selected mode and DB, image reference, commands run, real test output, health URL, persistent paths, secret source (not values), migration result, rollback procedure, and unresolved risks.
