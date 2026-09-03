# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal **Runtipi custom app store**. Runtipi loads it as an additional app source so the owner can install and upgrade self-hosted services from the Runtipi dashboard. The point of owning the store is **version control** — existing third-party Runtipi apps for these tools fall behind upstream, so this store pins exact image tags and bumps them on the owner's schedule.

Apps in this store, all using **official upstream Docker images**:

- `openclaw-hari` — OpenClaw instance for Hari (host port 18789)
- `openclaw-vinuta` — OpenClaw instance for his wife Vinuta (host port 18799)
- `hermes-hari` — Hermes Agent for Hari (gateway + dashboard, host port 9119)
- `hermes-vinuta` — Hermes Agent for Vinuta (gateway + dashboard, host port 9219)
- `journiv` — Journiv private journal (host port 8000; PostgreSQL + Valkey + Celery)

Each per-user instance must be **fully isolated** — distinct host ports, container/service names (`<app>-<user>-<role>`, e.g. `hermes-vinuta-gateway`), and `${APP_DATA_DIR}` directories. They share no state. The Hari/Vinuta pairs use the same upstream image and form-field set; keep them in sync when bumping versions.

## `random` form fields must be named per-instance

Runtipi derives `random` fields as `sha256(env_variable_name + machine_seed)` (`packages/backend/src/modules/env/env.utils.ts`). The value depends on the **field name only** — not the app — so two apps declaring the same `random` env var get **byte-identical secrets**.

Verified on the live host: `hermes-hari` and `hermes-vinuta` share one `API_SERVER_KEY`, and `openclaw-hari`/`openclaw-vinuta` share one `OPENCLAW_GATEWAY_TOKEN`. That silently defeats the isolation this store is built around — either user's token opens the other's gateway.

So **prefix every `random` field with the instance**: `HARI_DASHBOARD_PASSWORD` / `VINUTA_DASHBOARD_PASSWORD`, not a shared `DASHBOARD_PASSWORD`. This is the one place the Hari/Vinuta pairs should deliberately *not* share a field name; everything else stays in sync.

`API_SERVER_KEY` and `OPENCLAW_GATEWAY_TOKEN` are still shared — renaming them rotates the credential and breaks existing API consumers, so fix those in a deliberate rotation, not as a drive-by.

Also note `min` is the *output length*, not a byte count: `min: 32` with `encoding: hex` yields a 32-character value.

## Runtipi custom app store layout

Each app lives at `apps/<app-id>/` with this shape:

```
apps/<app-id>/
  config.json
  docker-compose.json     # strict ARK schema, NOT Docker Compose YAML
  metadata/
    description.md
    logo.jpg
```

`<app-id>` must match the `id` field inside `config.json` and the folder name exactly.

## docker-compose format — strict ARK schema (what actually works)

Runtipi has three possible compose formats; **only one of them works reliably** in v4.5+:

| Format | Status |
|---|---|
| `docker-compose.json` (strict ARK schema) | ✅ Use this — proven, what the official Nextcloud uses |
| `docker-compose.yml` with top-level `x-runtipi: schema_version: 2` | ⚠️ Newer YAML dynamic format; the converter rejects valid Docker Compose YAML if `healthcheck.test` is an array, named volumes are used, etc. We hit this on first install. |
| `docker-compose.yml` traditional v3 (with `${APP_PORT}`, `tipi_main_network`, `traefik.*` labels) | Legacy; works but Runtipi auto-generates none of the routing/networking — every label has to be written by hand |

**This store uses `docker-compose.json` exclusively.** The schema is defined in `runtipi/runtipi/packages/common/src/schemas/dynamic-compose-ark.ts`.

### Required structure

```json
{
  "schemaVersion": 2,
  "services": [
    {
      "name": "my-service",
      "image": "registry/image:pinned-tag",
      "isMain": true,
      "internalPort": 80,
      "restart": "unless-stopped",
      "environment": [
        { "key": "FOO", "value": "bar" },
        { "key": "USER_VAR", "value": "${USER_VAR}" }
      ],
      "volumes": [
        { "hostPath": "${APP_DATA_DIR}/data/config", "containerPath": "/etc/myapp" }
      ]
    }
  ]
}
```

### Hard constraints (will fail validation otherwise)

- `services` is an **array** of objects with a `name` field — not a Docker-Compose-style map.
- All field names are **camelCase**: `isMain`, `internalPort`, `addToMainNetwork`, `dependsOn`, `extraHosts`, `healthCheck`, `workingDir`, `stopSignal`, etc.
- `environment` is an array of `{ key, value }` objects. `value` must be non-empty (literal `${VAR}` is fine; an empty string is not).
- `volumes` is an array of `{ hostPath, containerPath, [readOnly], [shared], [private], [bind] }`. **Named volumes don't work** — paths resolve to host bind mounts. Use `${APP_DATA_DIR}/data/<subdir>` for persistent data.
- `command` and `entrypoint` are `string` or `string[]`.
- `healthCheck.test` is a **string**, not the standard Docker `["CMD", ...]` array. Skip healthcheck entirely if the natural form is an array.
- `restart` must be one of `"no"`, `"always"`, `"on-failure"`, `"unless-stopped"`.
- **`build:` is not in the schema** — the converter silently drops it. Pre-built images only. If you need a custom image, push it to a registry (ghcr.io, Docker Hub) before bumping the `image:` tag here.

### What Runtipi auto-generates

- `${APP_PORT}` host port mapping for the `isMain` service's `internalPort` — never declare `addPorts` for the main port.
- Traefik labels for the `isMain` service (HTTP/HTTPS routers, service, certresolver) when `exposed` or `exposedLocal` is set in the install form.
- A per-app Docker network (`<appName>_<storeId>_network`) that joins all services in a multi-service compose.
- Joining `tipi_main_network` for any service with `isMain: true` or `addToMainNetwork: true` — this is what enables cross-app DNS (e.g. `http://hermes-gateway:8642` from a sibling Runtipi app).

## config.json conventions

- `id` must match the folder name exactly.
- `port` — host port for the main service (Runtipi maps `${APP_PORT}` to `internalPort`). Must be unique across the entire store.
- `tipi_version` — integer; bump every time `config.json` or `docker-compose.json` changes. Runtipi uses it to detect upgrades.
- `version` — informational, the upstream app's version string.
- `dynamic_config: true` — required so Runtipi reads `docker-compose.json` (the dynamic format).
- `min_tipi_version: "4.5.0"` — minimum version that supports the strict ARK schema.
- `form_fields[]` — install-time form. Values become env vars available for substitution in `docker-compose.json` `environment` blocks. **Do not set a default OpenRouter URL on `OPENAI_BASE_URL`** — a real OpenAI key with the OpenRouter URL silently 401s. Leave the default blank.

## Timezone convention (every app must set this)

Both upstream images ship `/etc/localtime` symlinked to `Etc/UTC`. With no `TZ` set, the agent reads the wall clock as UTC and gets dates, "what time is it", and cron scheduling wrong even though the host clock is correct. Every app in this store therefore sets:

```json
{"key": "TZ", "value": "${<APP>_TZ:-${TZ}}"}
```

- The bare `${TZ}` is **Runtipi's own** timezone setting. `packages/backend/src/common/helpers/env-helpers.ts` writes `TZ` into the system `.env` (from Settings → General → Timezone, defaulting to the host zone), and `app.helpers.ts` seeds each app's `app.env` from that file — so `${TZ}` is always populated for app compose interpolation.
- `${<APP>_TZ}` is an optional per-app form field with **no `default`**, so leaving it blank falls through to the server zone. Docker Compose's `:-` treats empty *and* unset alike, and nested defaults (`${A:-${B}}`) work — Runtipi only interpolates `{{RUNTIPI_APP_ID}}` itself and passes `${...}` through to Compose verbatim.
- **Hermes also needs `HERMES_TIMEZONE`**, set to the same expression. Per `hermes_time.py` the agent's clock resolves `HERMES_TIMEZONE` → `timezone` in `config.yaml` → server local time; `TZ` alone fixes Python/libc but not Hermes's agent-facing sense of "now".

Do **not** try to fix this by bind-mounting `/etc/localtime` or `/etc/timezone`. Tested and rejected: Node resolves the zone from the symlink *target name*, not file contents, so OpenClaw stays on UTC regardless; the mount silently overwrites the container's `UTC`/`Etc/UTC` tzdata entries; and hosts without an `/etc/timezone` file (Fedora/Arch) make the container fail to start outright. Setting `TZ` to an IANA name is sufficient — both images bundle complete tzdata.

## Hermes dashboard requires auth (v2026.6+)

Since the June 2026 hardening, Hermes refuses to serve the dashboard on a non-loopback bind unless an auth provider is registered — it logs `Refusing to bind dashboard to 0.0.0.0` and exits 1. Runtipi must bind `0.0.0.0` for Traefik, so a password is mandatory. `--insecure` is now a deprecated no-op; don't rely on it.

Wire it through the basic-auth plugin's env overrides (`plugins/dashboard_auth/basic`), which win over `config.yaml`:

- `HERMES_DASHBOARD_BASIC_AUTH_USERNAME`
- `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD` (plaintext; `_PASSWORD_HASH` also accepted)
- `HERMES_DASHBOARD_BASIC_AUTH_SECRET` — signs session tokens; without a fixed value every restart logs you out

## Model selection: config-file writes at container start, not env vars

Neither app can have its model set by an env var — verified against the running images, not just the docs:

- **Hermes.** `HERMES_MODEL` / `HERMES_INFERENCE_MODEL` are read only by `cron/scheduler.py`, `tui_gateway/`, and `hermes_cli/oneshot.py`. Nothing under `gateway/` reads them, so for the `gateway run` process behind Matrix/Slack they are silently inert. The gateway resolves `config.yaml`'s `model.default` / `model.provider`.
- **OpenClaw.** No model env var exists at all; the only `OPENCLAW_MODEL*` symbol in `/app/dist` is `OPENCLAW_MODEL_ID = "openclaw"`, an unrelated internal constant. The model lives at `agents.defaults.model.primary` in `openclaw.json`.

So the form field is applied by wrapping the service `command` in `sh -c`, running the app's own non-interactive config CLI, then `exec`ing the real process:

- Hermes: `hermes config set model.provider|model.default …; exec hermes gateway run`
- OpenClaw: `node openclaw.mjs config set agents.defaults.model.primary …; exec node openclaw.mjs gateway …`

Both writes land in the `${APP_DATA_DIR}` bind mount, so they persist. Guard each with `[ -n "$$VAR" ]` — blank means "don't touch the config" — and `|| echo WARNING >&2` rather than `&&`, so a bad value never blocks startup.

Rules this depends on:

- **Escape the shell's `$` as `$$` inside `command`.** Runtipi passes the string through to the generated compose file verbatim; Docker Compose then turns `$$VAR` into a literal `$VAR` for the shell to expand at runtime. Writing `${VAR}` instead would splice the value in textually at compose-parse time and break on any quote in it.
- **Name the form field's env var something the app doesn't read** (`HERMES_LLM_MODEL`, `OPENCLAW_LLM_MODEL`). Naming it `HERMES_MODEL` would also pin the cron scheduler to a value that stops tracking later `/model` switches.
- **Don't replace `entrypoint`.** The Hermes image entrypoint is `/opt/hermes/docker/entrypoint-dispatch.sh` → s6 `/init` → `main-wrapper.sh`, which drops root to uid 10000. It routes args as: first arg is an executable → exec it directly; otherwise → `hermes <args>`. That's exactly why `command: ["sh","-c",…]` works while keeping the supervision tree and the privilege drop. OpenClaw's entrypoint is `tini -s --`, which execs whatever it's given.
- These fields win on every restart, overriding a model picked in the Hermes dashboard / `openclaw models set` / `/model`. That's the documented trade-off in the hints and both descriptions — don't "fix" it by making the write first-run-only without saying so.

## s6-overlay images: `command` runs post-start and needs `with-contenv`

`calibre-web-nextgen` is an LSIO/s6-overlay image (`ENTRYPOINT ["/init"]`, no `CMD`), and it does **not** behave like the Hermes/OpenClaw pattern above. Verified by running the image, not by reading docs:

- **`command` does not replace the app.** s6 starts its supervised services first, *then* runs the container `CMD` as a one-shot. The log order is `[ls.io-init] done` → your command. So there is no `exec`-the-real-process step — write the command as a plain side-effect script with no `exec` tail.
- **The container stays up after the command exits.** s6 keeps supervising, so a short-lived `CMD` is safe here. Do not add `sleep infinity`.
- **s6 strips the container environment from `CMD`.** A bare `bash -c 'echo $FOO'` prints empty even with `-e FOO=bar`; the value lives in `/run/s6/container_environment/FOO`. This fails *silently* — the script runs, reads an empty var, and does nothing. Wrap the command in `/usr/bin/with-contenv`:

  ```json
  "command": ["/usr/bin/with-contenv", "bash", "-c", "…"]
  ```

- **Drop privileges yourself.** The one-shot runs as root, so writes to `/config` land root-owned unless prefixed with `s6-setuidgid abc` (uid/gid 1000, the `PUID`/`PGID` user).

The `$$`-escaping rule from the model-selection section still applies unchanged.

### Applying a value only when it changes

For anything a user can also edit inside the app (a password, a model), re-applying on every boot silently reverts their in-app change. Guard on a hash of the value instead of a plain "already ran" marker:

```sh
sig=$$(printf %s "$$PW" | sha256sum | cut -d" " -f1)
[ "$$(cat "$$sf" 2>/dev/null)" != "$$sig" ] && apply && printf %s "$$sig" > "$$sf"
```

A plain restart is then a no-op, while editing the Runtipi form field does rotate the value. `calibre-web-nextgen` uses this for the admin password, which also means the app never sits on the upstream default `admin123`. Note the app enforces its own password policy (8+ chars, upper, lower, digit, special) and Runtipi `random` fields cannot satisfy it — `hex` is lowercase and digits only, and `base64` has no guaranteed special character — so that field is a user-supplied `password`, not a `random`.

## Plugin / extension persistence (OpenClaw lesson)

Runtime-installed OpenClaw plugins (`docker exec ... openclaw plugins install @openclaw/<name>`) **do persist** across restarts and container recreation in this store, because OpenClaw's `~/.openclaw/` directory is bind-mounted to a host path under `${APP_DATA_DIR}/data/config/`. The "plugins disappear on restart" problem the user hit with third-party Runtipi apps was caused by anonymous/named Docker volumes that get wiped — the bind-mount approach avoids that entirely.

We previously tried baking plugins into a custom image via `Dockerfile` + `build:`. **That approach is incompatible with Runtipi's dynamic compose schema** because `build:` is not a supported field. If you want plugins baked in instead of installed at runtime, build the custom image elsewhere (CI, manual `docker build`) and push to a registry, then reference it as `image:`.

## Before scaffolding any app, verify

1. The official image's tag exists on the registry and the user wants that exact version pinned.
2. Env-var names the image actually reads (names vary across projects).
3. Container port the image's web UI listens on (for `internalPort`).
4. Paths the image expects to be persisted as bind mounts.
5. `healthCheck.test` (if used) can be expressed as a single string command.

## References

- Strict ARK schema: https://github.com/runtipi/runtipi/blob/develop/packages/common/src/schemas/dynamic-compose-ark.ts
- YAML→ARK converter: https://github.com/runtipi/runtipi/blob/develop/packages/common/src/schemas/utils/convert-legacy-schema.ts
- Working reference app (uses `docker-compose.json`): https://github.com/runtipi/runtipi-appstore/blob/master/apps/nextcloud/docker-compose.json
- Runtipi config.json reference: https://runtipi.io/docs/reference/config-json
