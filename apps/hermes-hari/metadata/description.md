# Hermes Agent (Hari's instance)

A personal instance of [Hermes Agent](https://github.com/NousResearch/hermes-agent) by Nous Research — a self-improving open-source AI agent with persistent memory, autonomous skills, and a long list of messaging integrations (Matrix, Telegram, Discord, Slack, WhatsApp, Signal, and more).

This instance is fully isolated from any other Hermes deployment in this Runtipi store: it has its own data directory under `${APP_DATA_DIR}/data/hermes/` and its own API server auth key.

This deployment runs two services from the same image:

- **gateway** (`hermes-hari-gateway`) — the agent runtime + OpenAI-compatible API server on port `8642`. Joined to Runtipi's shared `tipi_main_network` via `addToMainNetwork: true` so other Runtipi apps can call it as an LLM provider at `http://hermes-hari-gateway:8642`.
- **dashboard** (`hermes-hari-dashboard`) — the web UI on port `9119`. This is what Runtipi's "Open" button points at.

Both services share the same persistent data directory at `${APP_DATA_DIR}/data/hermes/` (mounted into the container at `/opt/data`, equivalent to `~/.hermes` in the upstream docs).

## Dashboard login

The dashboard is password-protected. Set **Dashboard password** at install time (username defaults to `admin`) and log in with those credentials at the app URL.

This is **required, not optional**: as of the June 2026 hardening, Hermes refuses to serve the dashboard on a non-loopback bind unless an auth provider is registered, and Runtipi has to bind `0.0.0.0` for Traefik to route to it. The old `--insecure` flag is now a deprecated no-op — with no password configured the dashboard container logs `Refusing to bind dashboard to 0.0.0.0` and exits.

Credentials are wired through `HERMES_DASHBOARD_BASIC_AUTH_USERNAME` / `_PASSWORD`, with `_SECRET` (the auto-generated **Dashboard session secret**) signing session tokens so logins survive container restarts.

**Still be careful about exposure.** The dashboard holds your API keys. Password auth is a real gate, but LAN-only or VPN-only access remains the safer default; if you do expose it publicly, consider stacking Authelia or Cloudflare Access in front.

## Timezone

Leave the **Timezone** field blank and both services inherit the Runtipi server's timezone (Settings → General → Timezone), which Runtipi exposes to every app as `TZ`. Set it only if this instance should run on a different clock than the server.

Two variables get set from it, and both matter:

- `TZ` — the OS-level zone Python's `datetime.now()` and the shell read.
- `HERMES_TIMEZONE` — Hermes's own setting. Per `hermes_time.py` the resolution order is `HERMES_TIMEZONE` → the `timezone` key in `config.yaml` → the server's local time. This is what the agent uses for "what time is it", relative dates, and cron scheduling.

Without these the container falls back to UTC — the image ships `/etc/localtime` symlinked to `Etc/UTC` — and the agent confidently reports UTC as local time even though the host clock is fine.

## First-time setup

1. Install. Copy the auto-generated **API server auth key** — that's the bearer token for the gateway's OpenAI-compat API.
2. Set at least one LLM provider key:
   - **OpenRouter (recommended):** set `OPENROUTER_API_KEY`, leave `OPENAI_BASE_URL` blank.
   - **OpenAI direct:** set `OPENAI_API_KEY` only.
   - **OpenAI-compatible endpoint:** set both `OPENAI_API_KEY` and `OPENAI_BASE_URL`.
3. Open the dashboard. On first launch it walks through model selection, terminal backend choice, and any messaging integrations.

## Matrix integration with on-server Conduit

The form exposes the core Matrix env vars (`MATRIX_HOMESERVER`, `MATRIX_ACCESS_TOKEN`, `MATRIX_USER_ID`, `MATRIX_ALLOWED_USERS`, `MATRIX_ENCRYPTION`). Setup steps:

1. In your Conduit instance, register a bot account (e.g. `@hermes-hari:matrix.yourdomain.com`) and grab its access token. **Use a different account from the Vinuta instance** — both bots speaking from the same account would be confusing.
2. Set `MATRIX_HOMESERVER` to **either**:
   - `http://conduit:6167` if the Conduit Runtipi app's service name is `conduit` and it's joined `tipi_main_network` (verify with `docker network inspect runtipi_tipi_main_network`), **or**
   - `https://matrix.yourdomain.com` (or whatever Conduit's exposed URL is) — works regardless of internal networking.
3. Set `MATRIX_USER_ID` to the bot's full ID and `MATRIX_ALLOWED_USERS` to a comma-separated list of users allowed to talk to it (your own ID at minimum).
4. Optionally enable `MATRIX_ENCRYPTION`. If you do, the agent will need to be cross-signed on first run — check the dashboard logs for the verification prompt.

## Calling Hermes as an OpenAI provider from other Runtipi apps

The gateway exposes an OpenAI-compatible API on `http://hermes-hari-gateway:8642/v1` over `tipi_main_network`. Use `API_SERVER_KEY` as the bearer token. Useful for routing Home Assistant, n8n, or any OpenAI-SDK app through Hermes (which gives them memory, skills, and provider failover for free).

## MCP servers (stub)

A separate Runtipi app in this store will host MCP servers (Obsidian, Google Workspace, etc.). Once that app is installed, Hermes can reach those servers over `tipi_main_network` using the MCP-app's service name as hostname.

Configure MCP servers by editing `${APP_DATA_DIR}/data/hermes/config.yaml` (the persistent config — survives restarts):

```yaml
# Stub — fill in once the mcp-servers app exists.
# Verify exact key path against the Hermes MCP docs for the version you're running.
mcp:
  servers:
    # obsidian:
    #   transport: http
    #   url: http://mcp-servers:8001/obsidian
    # gworkspace:
    #   transport: http
    #   url: http://mcp-servers:8001/gworkspace
    #   headers:
    #     Authorization: "Bearer <token>"
```

## Updating

Bump both `image:` tags in `docker-compose.json` to a newer dated tag from <https://hub.docker.com/r/nousresearch/hermes-agent/tags> and increment `tipi_version` in `config.json`. Update **both** `apps/hermes-hari/` and `apps/hermes-vinuta/` together to keep them in sync.

## Migrating from the old `hermes-agent` app

If you previously had this installed under the old `hermes-agent` id (before the Hari/Vinuta split), uninstall the old app first, then move the data dir on the host:

```bash
sudo mv /runtipi/app-data/personal-apps/hermes-agent /runtipi/app-data/personal-apps/hermes-hari
sudo chown -R 10000:10000 /runtipi/app-data/personal-apps/hermes-hari/data
```

Then install `hermes-hari` from the dashboard. Existing config (`config.yaml`, memories, skills, sessions) will be preserved.
