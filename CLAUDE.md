# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A deployment directory for **self-hosted Supabase** via Docker Compose (the upstream `supabase/supabase` repo's `docker/` folder, installed by `setup.sh`). The deployed version is recorded in `.supabase-version` (currently `self-hosted/v0.8.3`). It is not a git checkout. There is no application code to build: the work here is configuration (`.env`, compose files, `volumes/`), POSIX shell scripts, SQL init scripts, and Deno edge functions.

## Commands

All scripts are POSIX `sh` unless noted. Run them from the project root.

```sh
sh run.sh start | stop | status | logs [svc] | pull
sh run.sh restart [svc] | restart --except <svc>...
sh run.sh recreate [svc]          # no args: down + up; with svc: force-recreate --no-deps
sh run.sh printenv <svc>          # env vars inside a running container
sh run.sh compose-config          # fully resolved compose config (check override layering)
sh run.sh config add|remove <name> # e.g. pg15, s3, kong, logs, pgbouncer, caddy, nginx
sh run.sh secrets                 # print key passwords/API keys from .env

sh update.sh --dry-run && sh update.sh   # in-place upgrade (back up the DB first)
sh reset.sh [-y]                  # DESTRUCTIVE: removes containers, volumes/db/data, volumes/storage, resets .env
```

Tests live in `tests/` and run from the project root. Most need a running stack, `jq`, and a populated `.env`:

```sh
sh tests/test-self-hosted.sh [base_url]   # end-to-end smoke test (default http://localhost:8000)
sh tests/test-auth-keys.sh                # API key types / asymmetric JWT auth
sh tests/test-container-logs.sh           # checks each service started cleanly
sh tests/test-s3.sh                       # needs the s3 override running
sh tests/test-s3-backend.sh               # needs s3 + tests/docker-compose.s3.test.yml
sh tests/test-update.sh                   # hermetic, no stack or network needed
sh tests/test-upgrades-manifest.sh        # validates upgrades.json, no stack needed
sudo bash tests/test-pg17-upgrade.sh      # needs a throwaway PG15 stack
```

## Architecture

**Compose layering.** `docker-compose.yml` is the base stack: studio, `api-gw` (Envoy, container `supabase-envoy`, with network alias `kong`), auth (GoTrue), rest (PostgREST), realtime, storage, imgproxy, meta, functions (edge-runtime), db (Postgres 17), and supavisor (pooler). The optional `docker-compose.<name>.yml` files are activated through the `COMPOSE_FILE=` colon list in `.env`. Plain `docker compose` reads that list too, so `run.sh config add/remove` is how to toggle them. Examples: `kong` swaps the gateway back to Kong, `pg15` pins Postgres 15, `logs` adds analytics (Logflare) and Vector, `pgbouncer` replaces supavisor (`!reset null`), and `s3`/`rustfs` add an S3 backend for storage. `docker-compose.override.yml` is user-owned and is never touched by updates. `dev/docker-compose.dev.yml` builds Studio from source in the upstream monorepo and adds inbucket mail. It only works inside that repo.

**Gateway routing.** All public traffic enters on port 8000 through the gateway. Routes are in `volumes/api/envoy/*.yaml`, with `lds.template.yaml` rendered by `docker-entrypoint.sh`, or in `volumes/api/kong.yml` when using Kong. When you add or change a public endpoint, update the gateway that is currently active.

**Auth keys.** There are two key generations side by side. The legacy ones are HS256 `JWT_SECRET`, `ANON_KEY` and `SERVICE_ROLE_KEY`. The newer ones are an EC P-256 pair (`JWT_KEYS`/`JWT_JWKS`) plus opaque `SUPABASE_PUBLISHABLE_KEY`/`SUPABASE_SECRET_KEY`. Generate them in order: `utils/generate-keys.sh`, then `utils/add-new-auth-keys.sh`. Rotate the opaque keys with `utils/rotate-new-api-keys.sh`. JWKS is passed into auth, realtime, storage and functions. `volumes/functions/main/index.ts` is the edge-runtime router: it verifies legacy and asymmetric JWTs, then dispatches to per-function workers under `volumes/functions/<name>/`.

**DB initialization.** The SQL files in `volumes/db/*.sql` are mounted into `/docker-entrypoint-initdb.d/`. They run **only on first init**, when `volumes/db/data` is empty. Editing them has no effect on an existing database. Apply changes manually, or reset with `reset.sh`. `utils/db-passwd.sh` changes the DB passwords across all service roles. `utils/reassign-owner.sh` reassigns ownership from `supabase_admin` to `postgres`.

**Update mechanism.** `update.sh` does a 3-way merge (`git merge-file`) of vendor files between the base ref (from `.supabase-version`) and the target `self-hosted/v*` tag. Paths matched by `.gitignore` are treated as user-owned and skipped. `.env` is never merged: missing keys from `.env.example` are appended to it. Breaking changes are gated by `upgrades.json`, which is hand-edited and keyed by bare semver. Add an entry there only for actions a file diff can't express, and keep it valid per `tests/test-upgrades-manifest.sh`. `CHANGELOG.md` and `versions.md` (the image version history) are for display only.

## Conventions

- Keep shell scripts POSIX `sh`-compatible. The exceptions are `upgrade-pg17.sh` and its test, which need bash for `pipefail`.
- `CONFIG.md` is the reference for every environment variable. Check it before adding or renaming a variable in `.env.example` or the compose files.
- Local state worth knowing: `docker-compose.yml` here differs from `docker-compose.yml.old`. The JWKS/`JWT_KEYS` env lines are uncommented for auth, realtime, storage and functions. `.env` and `volumes/db/data` hold live secrets and data. Never print `.env` contents or delete the data directory without asking.
