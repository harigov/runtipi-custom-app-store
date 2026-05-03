# OpenClaw (Vinuta's instance)

A personal instance of [OpenClaw](https://github.com/openclaw/openclaw) — an open-source, plugin-based AI assistant that runs locally and connects to LLM providers of your choice.

This instance is isolated from any other OpenClaw deployment in this Runtipi store: it has its own config and workspace directories under `${APP_DATA_DIR}/data/`, and its own gateway token. Vinuta and Hari should each install their own instance — do not share a single deployment.

## First-time setup

1. Install the app and copy the auto-generated **Gateway auth token** somewhere safe — you'll need it to log in.
2. Configure at least one LLM provider via the form fields:
   - **OpenRouter (recommended):** set `OPENROUTER_API_KEY`, leave `OPENAI_BASE_URL` blank.
   - **OpenAI direct:** set `OPENAI_API_KEY`, leave `OPENAI_BASE_URL` blank.
   - **Custom OpenAI-compatible endpoint:** set both `OPENAI_API_KEY` and `OPENAI_BASE_URL`.
3. Open the gateway URL from the Runtipi dashboard. Authenticate with the gateway token.
4. From the gateway UI, choose your default model.

## Matrix integration (one-time install)

Runtipi's dynamic compose schema does not support `build:` directives, so the Matrix plugin can't be baked into the image at build time. You install it once via `docker exec`, and it persists across restarts and container recreations because the OpenClaw config directory is a host bind mount at `${APP_DATA_DIR}/data/config/`.

```bash
# Note the --dangerously-force-unsafe-install flag — required because OpenClaw's
# static analyzer flags @openclaw/matrix's child_process use (legitimate; needed
# to install matrix-sdk-crypto-nodejs runtime deps). Plugin is officially signed
# (channel=official verification=source-linked), so the override is appropriate.
docker exec -it openclaw-vinuta_personal-apps-openclaw-vinuta-gateway-1 \
  openclaw plugins install @openclaw/matrix --dangerously-force-unsafe-install

docker exec -it openclaw-vinuta_personal-apps-openclaw-vinuta-gateway-1 \
  openclaw channels add
```

The wizard prompts for: homeserver URL, access token (or user/password), device name, E2EE on/off, and room allowlist. For a Conduit instance running as a sibling Runtipi app, set `homeserver` to either:

- The internal Docker URL — typically `http://conduit:6167` if the Conduit Runtipi app's service name is `conduit` and it's joined Runtipi's shared `tipi_main_network`, **or**
- The external URL (e.g. `https://matrix.yourdomain.com`) — works regardless of network setup.

Use a **separate Matrix account** from the Hari instance — both bots speaking from the same account would be confusing.

## Adding more plugins

Same pattern — `docker exec ... openclaw plugins install @openclaw/<name>`. Persists once installed. The available channel plugins are listed in <https://docs.openclaw.ai/tools/plugin>.

## Why we're pinned to 2026.4.20 (not the latest)

See `apps/openclaw-hari/metadata/description.md` for the full explanation. Short version: `@openclaw/matrix@2026.3.13` (the latest on npm) targets the pre-refactor plugin-SDK shape, which was changed in OpenClaw 2026.4.25. `2026.4.24` has its own matrix-sync regression. `2026.4.20` is the safe latest.

## Updating to a newer OpenClaw version (when the plugin catches up)

1. Pick the new version from <https://github.com/openclaw/openclaw/releases>.
2. Update three places consistently in **both** `apps/openclaw-hari/` and `apps/openclaw-vinuta/`:
   - `docker-compose.json` — `services[0].image` tag (no `v` prefix)
   - `config.json` — `version` field, and bump `tipi_version`
3. Commit and push to the store.

## MCP servers (stub)

A separate Runtipi app in this store will host MCP servers (Obsidian, Google Workspace, etc.). Once that app is installed, OpenClaw can reach those servers over Runtipi's shared `tipi_main_network` using the MCP-app's service name as hostname.

Configure MCP servers by editing `${APP_DATA_DIR}/data/config/openclaw.json` (the persistent config — survives restarts):

```json5
{
  // Stub — fill in once the mcp-servers app exists.
  // Verify exact key path against https://docs.openclaw.ai for the version you're running.
  mcp: {
    servers: {
      // "obsidian": { transport: "http", url: "http://mcp-servers:8001/obsidian" },
      // "gworkspace": { transport: "http", url: "http://mcp-servers:8001/gworkspace", headers: { Authorization: "Bearer ..." } },
    },
  },
}
```
