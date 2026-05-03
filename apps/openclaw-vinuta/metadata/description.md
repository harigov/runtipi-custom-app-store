# OpenClaw (Vinuta's instance)

A personal instance of [OpenClaw](https://github.com/openclaw/openclaw) — an open-source, plugin-based AI assistant that runs locally and connects to LLM providers of your choice.

This instance is isolated from any other OpenClaw deployment in this Runtipi store: it has its own config volume, workspace volume, and gateway token. Vinuta and Hari should each install their own instance — do not share a single deployment.

## First-time setup

1. Install the app and copy the auto-generated **Gateway auth token** somewhere safe — you'll need it to log in.
2. Configure at least one LLM provider via the form fields:
   - **OpenRouter (recommended):** set `OPENROUTER_API_KEY`, leave `OPENAI_BASE_URL` blank.
   - **OpenAI direct:** set `OPENAI_API_KEY`, leave `OPENAI_BASE_URL` blank.
   - **Custom OpenAI-compatible endpoint:** set both `OPENAI_API_KEY` and `OPENAI_BASE_URL`.
3. Open the gateway URL from the Runtipi dashboard. Authenticate with the gateway token.
4. From the gateway UI, choose your default model. For OpenRouter, use the form `openrouter/<provider>/<model>` (e.g. `openrouter/anthropic/claude-sonnet-4`).

## Matrix integration

The Matrix plugin (`@openclaw/matrix`) is **pre-baked into the custom image** via the `Dockerfile` in this app directory — no manual `docker exec plugins install` needed. To configure the channel after install:

```bash
docker exec -it openclaw-vinuta-gateway openclaw channels add
```

The wizard will prompt for: homeserver URL, access token (or user/password), device name, E2EE on/off, and room allowlist. If you're pointing at a Conduit instance running on this same Runtipi server, set `homeserver` to either:
- The internal Docker URL — typically `http://conduit:6167` if the Conduit app's service name is `conduit` and you've joined the same Docker network, **or**
- The external URL (e.g. `https://matrix.yourdomain.com`) — works regardless of network setup.

Use a separate Matrix account from the Hari instance — both bots speaking from the same account would be confusing.

## Adding more plugins

Append `RUN openclaw plugins install @openclaw/<name>` lines to `Dockerfile`, bump the image tag in both `Dockerfile` and `docker-compose.yml`, increment `tipi_version` in `config.json`, and reinstall the app from Runtipi.

Note: this app's `Dockerfile` should stay in sync with `apps/openclaw-hari/Dockerfile` — both Hari and Vinuta share the same image tag (`openclaw-custom:v2026.5.2`), so whichever app is installed first builds it and the other reuses the cached image.

## MCP servers (stub)

A separate Runtipi app in this store will host MCP servers (Obsidian, Google Workspace, etc.). Once that app is installed, OpenClaw can reach those servers via Runtipi's shared Docker network using the MCP-app's service name as hostname.

Configure MCP servers by editing `~/.openclaw/openclaw.json` (the persistent volume — survives restarts):

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

Reach the future MCP-servers app by its Runtipi service name, e.g. `http://mcp-servers:<port>` — no extra network config is needed since Runtipi v4+ joins all apps to a common bridge.

## Updating to a newer OpenClaw version

1. Pick the new version from <https://github.com/openclaw/openclaw/releases>.
2. Update three places consistently in **both** `apps/openclaw-hari/` and `apps/openclaw-vinuta/`:
   - `Dockerfile` — `ARG OPENCLAW_VERSION=...` default
   - `docker-compose.yml` — both the `image:` tag and `build.args.OPENCLAW_VERSION`
   - `config.json` — `version` field, and bump `tipi_version`
3. **Plugin upgrade caveat:** Docker pre-populates a fresh named volume from the image only on first creation. Existing config volumes won't pick up new plugin versions automatically on image upgrade. To refresh plugins after a version bump, either run `docker exec -it openclaw-vinuta-gateway openclaw plugins update` (if available), or wipe the volume and reinstall (loses local config).
