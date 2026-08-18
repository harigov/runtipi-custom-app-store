# OpenClaw (Hari's instance)

A personal instance of [OpenClaw](https://github.com/openclaw/openclaw) — an open-source, plugin-based AI assistant that runs locally and connects to LLM providers of your choice.

This instance is isolated from any other OpenClaw deployment in this Runtipi store: it has its own config and workspace directories under `${APP_DATA_DIR}/data/`, and its own gateway token.

## First-time setup

1. Install the app and copy the auto-generated **Gateway auth token** somewhere safe — you'll need it to log in.
2. Configure at least one LLM provider via the form fields:
   - **OpenRouter (recommended):** set `OPENROUTER_API_KEY`, leave `OPENAI_BASE_URL` blank.
   - **OpenAI direct:** set `OPENAI_API_KEY`, leave `OPENAI_BASE_URL` blank.
   - **Custom OpenAI-compatible endpoint:** set both `OPENAI_API_KEY` and `OPENAI_BASE_URL`.
3. Open the gateway URL from the Runtipi dashboard. Authenticate with the gateway token.
4. Set the **LLM model** field, or choose the default model from the gateway UI instead (see below).

## Choosing the model

The **LLM model** field sets the agent's primary model without opening the gateway UI. OpenClaw model refs are `provider/model`, split on the *first* slash — so an OpenRouter id keeps its vendor prefix inside the model half:

| Route | LLM model |
|---|---|
| OpenRouter | `openrouter/anthropic/claude-opus-4.7` |
| Anthropic direct | `anthropic/claude-opus-4-7` |
| OpenAI direct | `openai/gpt-5.6` |
| Local Ollama | `ollama/gemma4:26b` |

The same model is named differently per route, so the ref has to match whichever provider's key you configured above.

**How it's applied:** the container runs `openclaw config set agents.defaults.model.primary <ref>` on startup, before `openclaw gateway`. The write is skipped when the field is blank, so leaving it empty keeps whatever is in `openclaw.json`. Setting it also auto-enables that provider's plugin — the gateway log line to look for is `agent model: openrouter/anthropic/claude-opus-4.7`.

**The trade-off:** because the write happens on every start, this field wins over `openclaw models set` and `/model` — a container restart reverts to what's in the Runtipi form. Either fill it in and manage the model from Runtipi, or leave it blank and manage it from the gateway UI.

OpenClaw has no model env var to do this with (`OPENCLAW_MODEL` doesn't exist — the only `OPENCLAW_MODEL*` symbol in the bundle is an unrelated internal constant), which is why this goes through the config file.

## Matrix integration (one-time install)

Runtipi's dynamic compose schema does not support `build:` directives, so the Matrix plugin can't be baked into the image at build time. You install it once via `docker exec`, and it persists across restarts and container recreations because the OpenClaw config directory is a host bind mount at `${APP_DATA_DIR}/data/config/`.

```bash
docker exec -it openclaw-hari_personal-apps-openclaw-hari-gateway-1 \
  openclaw plugins install @openclaw/matrix

docker exec -it openclaw-hari_personal-apps-openclaw-hari-gateway-1 \
  openclaw channels add
```

> The `--dangerously-force-unsafe-install` flag this used to require is **no longer needed**. As of 2026.7.1 the install-time dangerous-code scanner has been removed upstream; passing the flag prints a deprecation notice and does nothing. Install-time policy is now `security.installPolicy` in `openclaw.json`.

The wizard prompts for: homeserver URL, access token (or user/password), device name, E2EE on/off, and room allowlist. For a Conduit instance running as a sibling Runtipi app, set `homeserver` to either:

- The internal Docker URL — `http://matrix-conduit:6167` if you're using the `matrix-conduit` Runtipi app from the migrated/legacy store (verified service name; joins `tipi_main_network` automatically). For other Conduit Runtipi apps, run `docker ps | grep -i conduit` to find the service name, **or**
- The external URL (e.g. `https://matrix.yourdomain.com`) — works regardless of network setup.

**Don't use a Tailscale `*.ts.net` URL** — Docker containers don't inherit Tailscale's MagicDNS resolver and the connection will fail with `ClientConnectorDNSError: Name or service not known`. The internal Docker URL avoids the public network entirely; the external HTTPS URL is simpler if your homeserver is already exposed.

**Why this is OK now (when it wasn't before):** the install writes plugin files into the bind-mounted host directory. As long as `${APP_DATA_DIR}/data/config/` exists on the host, the plugin survives container recreation, image upgrades, and host reboots.

## Timezone

Leave the **Timezone** field blank and the container inherits the Runtipi server's timezone (Settings → General → Timezone), which Runtipi exposes to every app as `TZ`. Fill the field in only if this instance should run on a different clock than the server.

This matters because the image ships with `/etc/localtime` symlinked to `Etc/UTC`. With no `TZ` set the agent reads the wall clock as UTC and gets "what time is it", relative dates, and scheduling wrong — even though the host clock is correct. Setting `TZ` to an IANA name is sufficient here: the image bundles full tzdata, and Node resolves the zone from `TZ` directly.

## Adding more plugins

Same pattern — `docker exec ... openclaw plugins install @openclaw/<name>`. Persists once installed. The available channel plugins are listed in <https://docs.openclaw.ai/tools/plugin>.

## Why 2026.7.1 (and not the newest tag)

The old `2026.4.20` pin is lifted. It existed because `@openclaw/matrix` was stuck at `2026.3.13` and targeted the pre-refactor plugin-SDK layout, which OpenClaw 2026.4.25 changed. The plugin has since been republished and now tracks core releases — `@openclaw/matrix@2026.7.1` declares `peerDependencies: { openclaw: ">=2026.7.1" }`, so the old pin is now the broken combination.

We pin **`2026.7.1`** rather than npm's `latest` (`2026.7.1-2`) deliberately: under semver a `-2` suffix parses as a *prerelease* of `2026.7.1`, so `2026.7.1-2` does **not** satisfy the plugin's `>=2026.7.1` peer range. `2026.7.1` matches cleanly. Verified: the matrix plugin installs and loads `enabled` on this tag.

The Docker tag scheme has no `v` prefix (e.g. `2026.7.1`, not `v2026.7.1`) — distinct from GitHub release tags. Note also that OpenClaw publishes an `extended-stable` channel (currently `2026.6.34`) if you'd rather track a slower-moving line.

## Updating to a newer OpenClaw version

Before bumping, check that `@openclaw/matrix`'s `peerDependencies.openclaw` range on npm still accepts the tag you're moving to — that pairing is what broke this app before.

1. Pick the new version from <https://github.com/openclaw/openclaw/releases> (or the GHCR tags list).
2. Update three places consistently:
   - `docker-compose.json` — `services[0].image` tag (no `v` prefix)
   - `config.json` — `version` field, and bump `tipi_version`
3. Commit and push to the store; Runtipi will detect the version bump.

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
