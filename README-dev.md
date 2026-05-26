# OpenClaw — Akimtech Fork Dev Environment

> **⚠️ Tripwire:** Always use `-f docker-compose.dev.yml` for compose commands in this repo. Bare `docker compose ...` will target the upstream `docker-compose.yml` and operate on a different project.

This is the operator runbook for working in the `akimtech/openclaw` fork. It covers local setup, the daily dev loop, and known quirks. For project background, architectural decisions, and phase-by-phase migration history, see the Phase 4 plan document (kept separately).

---

## Prerequisites

- macOS with Docker Desktop, allocated **7.5 GB RAM and 8 CPUs** to the Docker VM
- Node 24 (optional locally — Docker handles the runtime; useful for IDE tooling)
- Local Ollama running on the Mac (port 11434), with `gemma3:1b` pulled
- This repo cloned to `~/Documents/appdev/openclaw` (or your chosen path)

## First-time setup

Skip to **Daily dev loop** if `~/.openclaw/openclaw.json` already exists.

### 1. Generate a gateway auth token

```bash
TOKEN=$(openssl rand -hex 32)
sed -i.bak "s/^OPENCLAW_GATEWAY_TOKEN=$/OPENCLAW_GATEWAY_TOKEN=$TOKEN/" .env
rm .env.bak
```

### 2. Run the onboard wizard (one-off interactive container)

```bash
docker compose -f docker-compose.dev.yml run --rm -it openclaw-gateway-dev \
  sh -c "pnpm install --frozen-lockfile && node openclaw.mjs onboard --mode local"
```

Wizard answers used in this fork:

| Prompt                                        | Answer                              |
| --------------------------------------------- | ----------------------------------- |
| Security acknowledgment (personal-by-default) | Yes                                 |
| Setup mode                                    | Quickstart                          |
| Ollama mode                                   | Local only                          |
| Ollama base URL                               | `http://host.docker.internal:11434` |
| Default model                                 | `ollama/gemma3:1b`                  |
| Search provider                               | Skip for now                        |
| Configure skills now                          | No                                  |
| Hatch your agent                              | Hatch later                         |

### 3. Manual correction — gateway bind mode

After the wizard writes `~/.openclaw/openclaw.json`, change `gateway.bind` from `"loopback"` to `"auto"`:

```bash
python3 -c "
import json, os
p = os.path.expanduser('~/.openclaw/openclaw.json')
with open(p) as f: d = json.load(f)
d['gateway']['bind'] = 'auto'
with open(p, 'w') as f: json.dump(d, f, indent=2)
"
```

This is required for the container's networking; without it, the gateway binds only to its own loopback and Compose can't proxy traffic in.

---

## Daily dev loop

### Gateway lifecycle

```bash
# Start (foreground — watch logs scroll)
docker compose -f docker-compose.dev.yml up

# Start (detached background)
docker compose -f docker-compose.dev.yml up -d

# Stop
docker compose -f docker-compose.dev.yml down

# Restart (faster than down/up — preserves the container, just re-execs the process)
docker compose -f docker-compose.dev.yml restart

# Status & logs
docker compose -f docker-compose.dev.yml ps
docker compose -f docker-compose.dev.yml logs --tail=50 openclaw-gateway-dev
```

### Access the dashboard

```
http://localhost:18789/#token=<OPENCLAW_GATEWAY_TOKEN from .env>
```

### Editing UI strings or components

**Important:** The gateway and the UI are independent build artifacts. **Restarting the gateway does NOT rebuild the UI.** To see UI source changes:

```bash
# After editing files in ui/src/
docker compose -f docker-compose.dev.yml exec openclaw-gateway-dev \
  sh -c "cd /app/ui && pnpm build"

# Then hard-refresh the browser (Cmd+Shift+R on Mac)
```

Build is ~4 seconds. Vite outputs to `dist/control-ui/` (configured in `ui/vite.config.ts` via `outDir`), which is exactly the directory the gateway serves — no copy step needed.

### Run commands inside the container

```bash
docker compose -f docker-compose.dev.yml exec openclaw-gateway-dev <command>

# Examples
docker compose -f docker-compose.dev.yml exec openclaw-gateway-dev sh
docker compose -f docker-compose.dev.yml exec openclaw-gateway-dev node openclaw.mjs devices list
```

---

## Pairing a new browser/device

The gateway token authorizes requests; pairing additionally authorizes a specific browser to drive the dashboard. Two independent gates.

1. Open the tokenized URL. Dashboard shows "Device pairing required" with a per-browser UUID.
2. Click **Connect** once — this registers the pending pairing request.
3. From the terminal:
   ```bash
   docker compose -f docker-compose.dev.yml exec openclaw-gateway-dev \
     sh -c "node openclaw.mjs devices list"
   # Copy the pending request UUID, then:
   docker compose -f docker-compose.dev.yml exec openclaw-gateway-dev \
     sh -c "node openclaw.mjs devices approve <uuid>"
   ```
4. Click **Connect** again in the dashboard. Do NOT refresh — refreshing regenerates the UUID.

---

## Branch model

| Branch                    | Purpose                                                             | Direct commits?          |
| ------------------------- | ------------------------------------------------------------------- | ------------------------ |
| `main`                    | Tracks akimtech/openclaw default; mirrors upstream merges over time | No                       |
| `PROD`                    | Deploy branch — what's on the production server                     | No (PR merges only)      |
| `feature/*`, `cosmetic/*` | Work branches off `main`                                            | Yes; PR target is `PROD` |

To sync `main` with upstream:

```bash
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

---

## File locations

| What                                | Path                                |
| ----------------------------------- | ----------------------------------- |
| Dev compose definition              | `docker-compose.dev.yml`            |
| Dev Dockerfile                      | `Dockerfile.dev`                    |
| Host-side env (gitignored)          | `.env`                              |
| UI source                           | `ui/src/`                           |
| UI strings (English source)         | `ui/src/i18n/locales/en.ts`         |
| UI translation memories             | `ui/src/i18n/.i18n/*.tm.jsonl`      |
| Vite config                         | `ui/vite.config.ts`                 |
| UI build output (served by gateway) | `dist/control-ui/`                  |
| Gateway runtime config (host)       | `~/.openclaw/openclaw.json`         |
| Pairing & device registry (host)    | `~/.openclaw/devices/`              |
| Gateway logs (in container)         | `/tmp/openclaw/openclaw-<date>.log` |

---

## Known quirks

**`pnpm gateway:watch:raw` SIGTERM-loops on Docker Desktop for Mac.** We use `pnpm gateway:dev` (one-shot, no watch). The compose `command:` block pins this. Don't switch back to `watch:raw`.

**macOS host paths leak into the container via env vars.** All `OPENCLAW_*` path env vars are explicitly pinned to Linux paths in `docker-compose.dev.yml`. Don't remove these pins.

**Ollama bridge uses `host.docker.internal:11434`.** Enabled via `extra_hosts: ["host.docker.internal:host-gateway"]` in the compose file.

**`openclaw devices remove` may not persist.** The CLI reports success but the entry can reappear in `devices list`. The lingering entry has scope `operator.pairing` only and is functionally harmless.

**Security warning on every startup** for `allowInsecureAuth=true`. Expected for dev (HTTP loopback, no TLS). Will be turned off for prod.

**`emptyOutDir: true` in vite.config.ts** means `vite build` wipes `dist/control-ui/` before writing. Don't refresh the browser _during_ a build — wait for `✓ built in Xs`.

**Service worker caching** can persist stale UI bundles. The build pipeline stamps each build with a unique ID in `sw.js` to defeat this, so a hard refresh (`Cmd+Shift+R`) is normally enough. Escalation path: DevTools → Application → Service Workers → "Bypass for network" → reload.

---

## Troubleshooting quick-reference

| Symptom                                             | Likely cause                             | First diagnostic                                                                                          |
| --------------------------------------------------- | ---------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `docker compose down` operates on the wrong project | Missing `-f docker-compose.dev.yml` flag | `docker compose ls`                                                                                       |
| UI edits don't show after gateway restart           | Gateway doesn't rebuild UI               | Run `vite build` from `/app/ui/`                                                                          |
| `connection refused` to localhost:18789             | Gateway not started or not yet ready     | `docker compose -f docker-compose.dev.yml logs --tail=20 openclaw-gateway-dev`                            |
| Dashboard prompts for re-pairing after restart      | `~/.openclaw/devices/` not persisted     | `ls ~/.openclaw/devices/`                                                                                 |
| Browser shows stale UI after rebuild                | Service worker cache                     | Hard refresh; if needed, DevTools "Bypass for network"                                                    |
| `pnpm install` hangs on container start             | Named volume corrupted                   | `docker compose -f docker-compose.dev.yml down -v` (nukes node_modules cache; will re-install on next up) |
