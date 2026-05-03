# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal **Runtipi custom app store**. Runtipi loads it as an additional app source so the owner can install and upgrade self-hosted services from the Runtipi dashboard. The point of owning the store is **version control** — existing third-party Runtipi apps for these tools fall behind upstream, so this store pins exact image tags and bumps them on the owner's schedule.

Apps in this store, all using **official upstream Docker images**:

- `openclaw-hari` — OpenClaw instance for Hari
- `openclaw-vinuta` — separate OpenClaw instance for his wife Vinuta (independent data, URL, and volumes)
- `hermes-agent` — Hermes Agent (gateway + dashboard)

The two OpenClaw instances must be fully isolated — distinct host ports, container names, and `${APP_DATA_DIR}` directories. They share no state.

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
