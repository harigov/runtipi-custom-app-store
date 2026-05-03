# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal **Runtipi custom app store**. Runtipi loads it as an additional app source so the owner can install and upgrade self-hosted services from the Runtipi dashboard. The point of owning the store is **version control** — existing third-party Runtipi apps for these tools fall behind upstream, so this store pins exact image tags and bumps them on the owner's schedule.

Planned apps, all using **official upstream Docker images**:

- `openclaw-hari` — OpenClaw instance for Hari
- `openclaw-vinuta` — separate OpenClaw instance for his wife Vinuta (independent data, URL, and volumes)
- `hermes-agent` — Hermes Agent instance

The two OpenClaw instances must be fully isolated — distinct service names, host ports, named volumes. They share no state.

## Runtipi custom app store layout

Each app lives at `apps/<app-id>/` with this shape:

```
apps/<app-id>/
  config.json
  docker-compose.yml
  metadata/
    description.md
    logo.jpg
```

`<app-id>` must match the `id` field inside `config.json` and the folder name exactly. All four files are required — Runtipi hides apps with missing metadata.

## config.json conventions (Runtipi v4+, schema 2)

- `"dynamic_config": true` — required so Runtipi reads `x-runtipi` blocks from `docker-compose.yml`.
- `port` — host port Runtipi binds. **Must be unique across the entire store**, so the two openclaw instances need different ports.
- `tipi_version` — integer; bump every time `config.json` or `docker-compose.yml` changes. Runtipi uses it to detect upgrades.
- `version` — the upstream app's version string (informational).
- `exposable: true` only if the app should be reachable on a public domain via Traefik.
- `form_fields[]` — install-time form. Each entry has `type`, `label`, `env_variable`, and usually `required`. Supported types include `text`, `password`, `random`, `boolean`, `number`, `fqdn`, `email`. Values become env vars available to the compose file.

For all apps in this store, expose at minimum:

- `OPENAI_API_KEY` — `type: password`, `required: false` (one of the LLM-provider keys must be set)
- `OPENAI_BASE_URL` — `type: text`, `required: false`, **no default** — defaulting to OpenRouter's URL silently misroutes real OpenAI keys. Hint should explain when to set it; prefer the dedicated `OPENROUTER_API_KEY` field for OpenRouter use.

The exact env-var **names** must match what the upstream image actually reads. Check the image's docs before committing — don't assume generic `OPENAI_*` names work for every project.

## docker-compose.yml conventions

Runtipi v4+ uses dynamic compose with `schema_version: 2`. Minimal shape:

```yaml
services:
  <service>:
    image: <official-image>:<pinned-tag>   # never :latest
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - OPENAI_BASE_URL=${OPENAI_BASE_URL}
    volumes:
      - <named-volume>:/data
    x-runtipi:
      is_main: true
      internal_port: <container-port>

volumes:
  <named-volume>:

x-runtipi:
  schema_version: 2
```

Rules:

- **Always pin image tags.** Bypassing this defeats the reason the store exists.
- Exactly one service per app gets `is_main: true`. `internal_port` is the container port Traefik routes to.
- Form-field values are interpolated as `${VAR}` in compose — never hardcode secrets.
- Per-app named volumes — the two openclaw instances must use distinct volume names.

## Before scaffolding any app, verify

1. The official image's tag exists on the registry and the user wants that exact version pinned.
2. The env-var names the image actually reads for API key and base URL (names vary across projects).
3. Which container port the image's web UI listens on (for `internal_port`).
4. Which paths the image expects to be persisted as volumes.

When uncertain about any of these, ask before committing a guess — a wrong env-var name leaves the app broken at install time, and a wrong `internal_port` makes the dashboard's "Open" button dead.

## References

- config.json reference: https://runtipi.io/docs/reference/config-json
- Custom app store guide: https://runtipi.io/docs/guides/create-your-own-app-store
- Dynamic compose guide: https://runtipi.io/docs/guides/dynamic-compose-guide
