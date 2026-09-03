# Journiv

A self-hosted private journal: mood tracking, writing prompts, media, analytics, and “on this day”. The [upstream project](https://github.com/journiv/journiv-app) is explicit that your entries stay on your machine — no telemetry, no hosted backend.

This store pins the official images rather than `latest`, and uses Journiv’s **production** compose (PostgreSQL + Valkey + Celery), not the one-container SQLite `docker run` they publish for a quick look.

## What this install runs

Five services on the per-app Docker network. Only the web UI is published.

| Service | Image | Role |
|---|---|---|
| **journiv** | `swalabtech/journiv-app:0.1.0-beta.24` | Web UI on container port `8000`. Runtipi maps `${APP_PORT}` here and points the Open button at it. |
| **journiv-celery-worker** | same Journiv tag | Background import / export |
| **journiv-celery-beat** | same Journiv tag | Scheduled jobs (export cleanup, etc.) |
| **journiv-postgres** | `postgres:18.1` | Database. Volume is `/var/lib/postgresql` — that is the Postgres 18 layout, not the older `/var/lib/postgresql/data`. |
| **journiv-valkey** | `valkey/valkey:9.0-alpine` | Cache, rate-limit store, and Celery broker |

Celery is required for import/export. The image entrypoint keys off `SERVICE_ROLE` (`app` / `celery-worker` / `celery-beat`) and runs migrations only in the `app` role, so the worker and beat containers do not race the schema.

The Journiv Plus sidecar overlay is not included. If you later subscribe to Plus, that is a separate compose overlay upstream; do not try to turn it on by adding env vars here.

## First login

Journiv has no install-time admin user. Leave **Disable new signups** off, open the app, create the first account, then turn that field on and restart. The official production checklist wants signup disabled once that account exists.

The **Secret key** and **PostgreSQL password** are generated at install. They live in this app’s `app.env` on the host. Rotating the secret key signs every session out; rotating the database password without also updating Postgres’s data directory will make the app fail to connect.

## Domain and reverse proxy

Journiv checks the `Host` header against `DOMAIN_NAME` and builds public URLs as `{DOMAIN_SCHEME}://{DOMAIN_NAME}/…`.

- **Public hostname** empty → `DOMAIN_NAME` is Runtipi’s `${APP_DOMAIN}` (exposed domain, local domain, or `ip:port`, depending on how you open the app).
- **Domain scheme** is `${APP_PROTOCOL}` (`https` when Traefik is serving TLS, `http` for a raw host port).

If the UI logs `Invalid Host Header`, you are opening the app at a host that is not `${APP_DOMAIN}`. Set **Public hostname** to the name in the browser address bar — no `https://`, no trailing slash.

Do not also set `DATABASE_URL`. This compose already passes `DB_DRIVER=postgres` and `POSTGRES_PASSWORD`; Journiv treats those as mutually exclusive with `DATABASE_URL` and exits if both are present.

## Timezone

Leave **Timezone** blank and every Journiv service inherits the Runtipi server’s timezone (Settings → General → Timezone). That value is applied as both `TZ` (libc / Python) and `CELERY_TIMEZONE` (the beat schedule). Without it the images stay on UTC — `/etc/localtime` is `Etc/UTC` — and entry dates plus “on this day” are off by your UTC offset.

Do not bind-mount `/etc/localtime`. The store already rejected that approach on other apps: Node and friends resolve the zone from the symlink *target name*, not the file contents, and the mount can overwrite tzdata.

## Immich

The main service joins `tipi_main_network`, so a Runtipi Immich install on the same host is reachable as `http://immich:2283` (or whatever that app’s service name is). Set **Immich base URL** to that and users can leave the URL blank in the Journiv UI. A per-user value in Settings still wins.

## Data on disk

All bind mounts sit under this app’s `${APP_DATA_DIR}`:

| Host path | Container path | Holds |
|---|---|---|
| `data/app` | `/data` | Media, logs, exports, imports. The Journiv process runs as uid `1000`. |
| `data/postgres` | `/var/lib/postgresql` | Postgres 18 data |
| `data/valkey` | `/data` | Valkey dump |

Back up `data/app` and `data/postgres` together. Media without the database (or the reverse) is not a usable restore.

## Upgrades

Images are pinned (`0.1.0-beta.24`, `postgres:18.1`, `valkey:9.0-alpine`). A restart will not silently pull a newer Journiv. Bump `version` and the image tags together, and increment `tipi_version`, when you want the update. Migrations run automatically in the `app` entrypoint on startup — back up first.
