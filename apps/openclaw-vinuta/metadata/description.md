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
docker exec -it openclaw-vinuta_personal-apps-openclaw-vinuta-gateway-1 \
  openclaw plugins install @openclaw/matrix

docker exec -it openclaw-vinuta_personal-apps-openclaw-vinuta-gateway-1 \
  openclaw channels add
```

> The `--dangerously-force-unsafe-install` flag this used to require is **no longer needed** as of 2026.7.1 — the install-time scanner was removed upstream and the flag is now a deprecated no-op.

The wizard prompts for: homeserver URL, access token (or user/password), device name, E2EE on/off, and room allowlist. For a Conduit instance running as a sibling Runtipi app, set `homeserver` to either:

- The internal Docker URL — `http://matrix-conduit:6167` if you're using the `matrix-conduit` Runtipi app from the migrated/legacy store (verified service name; joins `tipi_main_network` automatically), **or**
- The external URL (e.g. `https://matrix.yourdomain.com`) — works regardless of network setup.

**Don't use a Tailscale `*.ts.net` URL** — Docker containers don't inherit Tailscale's MagicDNS resolver. Use a **separate Matrix account** from the Hari instance — both bots speaking from the same account would be confusing.

## Adding more plugins

Same pattern — `docker exec ... openclaw plugins install @openclaw/<name>`. Persists once installed. The available channel plugins are listed in <https://docs.openclaw.ai/tools/plugin>.

## Why 2026.7.1 (not the newest tag)

See `apps/openclaw-hari/metadata/description.md` for the full explanation. Short version: the old `2026.4.20` pin is lifted because `@openclaw/matrix` has been republished and now requires `openclaw >= 2026.7.1`. We pin `2026.7.1` rather than npm's `latest` (`2026.7.1-2`) because the `-2` suffix parses as a semver prerelease and wouldn't satisfy that peer range.

## Timezone

Leave the **Timezone** field blank to inherit the Runtipi server's timezone (Settings → General → Timezone). Fill it in only if this instance should run on a different clock than the server. Without it the container reads the wall clock as UTC — the image ships `/etc/localtime` pointing at `Etc/UTC` — and the agent gets dates, "what time is it", and scheduling wrong.

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
