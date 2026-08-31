# Agent implementation requirements

## Mission

Adapt this blueprint to a target application while preserving two supported deployment patterns: full-source host deployment and registry artifact deployment. Do not blindly copy values from the examples.

## Discovery before editing

The agent must inspect and record:

1. Runtime/framework versions and package lock files.
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
- Only one of the `postgres` and `mysql` profiles is selected.
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

## Database stubs

Keep both stubs until the operator selects one, but hide them behind Compose profiles.

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

Before cutover, remove ambiguity from production operations: document the chosen profile, driver, backup command, restore drill, and migration procedure.

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
