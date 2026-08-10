# Hermes Agent (Vinuta's instance)

A personal instance of [Hermes Agent](https://github.com/NousResearch/hermes-agent) by Nous Research — a self-improving open-source AI agent with persistent memory, autonomous skills, and a long list of messaging integrations (Matrix, Telegram, Discord, Slack, WhatsApp, Signal, and more).

This instance is fully isolated from any other Hermes deployment in this Runtipi store: it has its own data directory under `${APP_DATA_DIR}/data/hermes/` and its own API server auth key. Vinuta and Hari should each install their own instance — do not share a single deployment.

This deployment runs two services from the same image:

- **gateway** (`hermes-vinuta-gateway`) — the agent runtime + OpenAI-compatible API server on port `8642`. Joined to Runtipi's shared `tipi_main_network` via `addToMainNetwork: true` so other Runtipi apps can call it as an LLM provider at `http://hermes-vinuta-gateway:8642`.
- **dashboard** (`hermes-vinuta-dashboard`) — the web UI on port `9119` (host port `9219` to avoid colliding with the Hari instance). This is what Runtipi's "Open" button points at.

Both services share the same persistent data directory at `${APP_DATA_DIR}/data/hermes/` (mounted into the container at `/opt/data`, equivalent to `~/.hermes` in the upstream docs).

## Dashboard login

The dashboard is password-protected. Set **Dashboard password** at install time (username defaults to `admin`) — use a different password from the Hari instance. Log in with those credentials at the app URL.

This is **required, not optional**: as of the June 2026 hardening, Hermes refuses to serve the dashboard on a non-loopback bind unless an auth provider is registered, and Runtipi has to bind `0.0.0.0` for Traefik. The old `--insecure` flag is now a deprecated no-op — with no password set the dashboard container exits on startup.

**Still be careful about exposure.** The dashboard holds your API keys; LAN-only or VPN-only access remains the safer default.

## Timezone

Leave the **Timezone** field blank and both services inherit the Runtipi server's timezone (Settings → General → Timezone). Set it only if this instance should run on a different clock than the server.

It sets both `TZ` (the OS-level zone Python reads) and `HERMES_TIMEZONE` (Hermes's own setting, which drives the agent's sense of "now" and its cron scheduling). Without them the container falls back to UTC — the image ships `/etc/localtime` pointing at `Etc/UTC` — and the agent reports UTC as local time. See `apps/hermes-hari/metadata/description.md` for the full detail.

## First-time setup

1. Install. Copy the auto-generated **API server auth key** — that's the bearer token for the gateway's OpenAI-compat API.
2. Set at least one LLM provider key:
   - **OpenRouter (recommended):** set `OPENROUTER_API_KEY`, leave `OPENAI_BASE_URL` blank.
   - **OpenAI direct:** set `OPENAI_API_KEY` only.
   - **OpenAI-compatible endpoint:** set both `OPENAI_API_KEY` and `OPENAI_BASE_URL`.
3. Open the dashboard. On first launch it walks through model selection, terminal backend choice, and any messaging integrations.

## Matrix integration with on-server Conduit

The form exposes the core Matrix env vars (`MATRIX_HOMESERVER`, `MATRIX_ACCESS_TOKEN`, `MATRIX_USER_ID`, `MATRIX_ALLOWED_USERS`, `MATRIX_ENCRYPTION`). Setup steps:

1. In your Conduit instance, register a **separate** bot account from the Hari instance (e.g. `@hermes-vinuta:matrix.yourdomain.com`) and grab its access token. Two bots on the same account would clash.
2. Set `MATRIX_HOMESERVER` to **either**:
   - `http://matrix-conduit:6167` if you're using the `matrix-conduit` Runtipi app from the migrated/legacy store (canonical service name, joins `tipi_main_network` automatically), **or**
   - `https://matrix.yourdomain.com` (Conduit's externally-exposed URL).

   **Don't use a Tailscale `*.ts.net` URL** — Docker containers don't inherit Tailscale's MagicDNS resolver.
3. Set `MATRIX_USER_ID` to the bot's full ID and `MATRIX_ALLOWED_USERS` to a comma-separated list of users allowed to talk to it (Vinuta's ID at minimum).
4. Optionally enable `MATRIX_ENCRYPTION`. If you do, the agent will need to be cross-signed on first run — check the dashboard logs for the verification prompt.

## Calling Hermes as an OpenAI provider from other Runtipi apps

The gateway exposes an OpenAI-compatible API on `http://hermes-vinuta-gateway:8642/v1` over `tipi_main_network`. Use `API_SERVER_KEY` as the bearer token.

## MCP servers (stub)

Same pattern as `hermes-hari` — edit `${APP_DATA_DIR}/data/hermes/config.yaml`:

```yaml
# Stub — fill in once the mcp-servers app exists.
mcp:
  servers:
    # obsidian:
    #   transport: http
    #   url: http://mcp-servers:8001/obsidian
```

## Updating

Bump both `image:` tags in `docker-compose.json` and increment `tipi_version`. Keep `apps/hermes-hari/` and `apps/hermes-vinuta/` in sync — same image tag, same form-field set.
