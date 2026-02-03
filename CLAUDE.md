# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm test              # Run tests (vitest)
npm run test:watch    # Run tests in watch mode
npm run typecheck     # TypeScript type checking
npm run build         # Build worker + client (vite)
npm run deploy        # Build and deploy to Cloudflare
npm run start         # Local worker development (wrangler dev)
npm run dev           # Vite dev server (client only)
```

## Project Overview

This is a Cloudflare Worker that runs OpenClaw (formerly Moltbot/Clawdbot) personal AI assistant in a Cloudflare Sandbox container. The worker proxies HTTP/WebSocket requests to the OpenClaw gateway running inside the container.

**Repository:** Fork of [cloudflare/moltworker](https://github.com/cloudflare/moltworker) with local customizations (MINIMAX_API_KEY, health check, openclaw-data bucket).

**Important naming note:** The upstream CLI is still named `clawdbot`, so internal commands and config paths use that name despite the project being called "moltbot" or "OpenClaw".

## Architecture

```
Browser Request → Cloudflare Worker → Sandbox Container → OpenClaw Gateway (port 18789)
                      ↓
              Admin UI (/_admin/)
              API endpoints (/api/*)
              Debug routes (/debug/*)
```

Key architectural patterns:
- **Request flow**: Public routes → CF Access auth → Gateway proxy (catch-all)
- **WebSocket handling**: Custom message interception and error transformation
- **Storage**: R2 backup/restore pattern (not direct mount) via cron every 15 minutes
- **Auth layers**: Cloudflare Access (admin) → Gateway Token → Device Pairing

## Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Main Hono app, middleware chain, catch-all proxy |
| `src/gateway/process.ts` | Find/start gateway processes in container, health check |
| `src/gateway/env.ts` | Build container environment variables |
| `src/gateway/sync.ts` | R2 backup/restore sync logic |
| `src/utils/logging.ts` | Log redaction for sensitive parameters |
| `Dockerfile` | Container image (Node 22 + clawdbot CLI) |
| `start-moltbot.sh` | Container startup script |
| `wrangler.jsonc` | Cloudflare Worker configuration |
| `test/e2e/` | E2E tests with Playwright (from upstream) |

## Testing

Tests use Vitest with colocated test files (`*.test.ts` next to source). Run a single test file:
```bash
npx vitest run src/auth/jwt.test.ts
```

## Development

For local dev, create `.dev.vars`:
```bash
ANTHROPIC_API_KEY=sk-ant-...
DEV_MODE=true           # Skips CF Access auth + device pairing
DEBUG_ROUTES=true       # Enables /debug/* routes
E2E_TEST_MODE=true      # Skips CF Access auth only (keeps device pairing)
```

**Additional env vars:**
- `MINIMAX_API_KEY` - MiniMax API provider support
- `TELEGRAM_DM_ALLOW_FROM` - Comma-separated Telegram user IDs allowlist
- `R2_BUCKET_NAME` - Custom R2 bucket name (default: `openclaw-data`)

Note: WebSocket proxying has issues with `wrangler dev`. Deploy to Cloudflare for full functionality.

## Sandbox Container Lifecycle

The `Sandbox` class in `src/index.ts` extends `@cloudflare/sandbox` with error handling for internal alarms.

**Container sleep behavior** is controlled by `SANDBOX_SLEEP_AFTER`:
- `never` (default): Container stays alive indefinitely (`keepAlive: true`)
- Duration (e.g., `10m`, `1h`): Container sleeps after inactivity period

**Important**: The cron backup runs every 15 minutes. If `SANDBOX_SLEEP_AFTER < 15m`, the container will cold start on each backup. Keep it `> 15m` or use the default `never`.

**Known issue - Alarm race condition**: The `@cloudflare/sandbox` package has a race condition where two alarms fire simultaneously (~15ms apart). The first succeeds (~3min), the second crashes immediately (~12ms) with `TypeError: Cannot read properties of undefined (reading 'isRetry')`.

The custom `Sandbox` class in `src/index.ts`:
- Provides default `alarmInfo` values to fix crashes when `alarmInfo` is undefined
- Wraps the alarm handler in try/catch to suppress errors

**Note**: Some alarm errors still appear in logs because they occur at the Cloudflare runtime level *before* our code executes. These errors don't affect functionality - the first alarm always succeeds. This is a bug in `@cloudflare/sandbox` that cannot be fixed from our code.

## R2 Backup Sync

The cron job (`*/15 * * * *`) syncs data from container to R2. Before syncing, a sanity check verifies:
1. `clawdbot.json` exists (`test -s` - not empty)
2. File starts with `{` (valid JSON)

This prevents overwriting good backups with corrupted/empty data. If the check fails, sync is skipped and an error is logged (expected on cold starts before gateway runs).

**Tip**: Enable R2 versioning in the Cloudflare dashboard to recover from accidental overwrites.

## Syncing with Upstream

This repo is a fork of `cloudflare/moltworker`. To sync:
```bash
git fetch upstream
git merge upstream/main
git push origin main
```

**Local customizations to preserve when merging:**
- `MINIMAX_API_KEY` support in `src/gateway/env.ts` and `start-moltbot.sh`
- `openclaw-data` R2 bucket name in `src/config.ts`
- Gateway health check in `src/gateway/process.ts`
- Sandbox alarm fix in `src/index.ts`

## Additional Guidelines

See `AGENTS.md` for detailed patterns including:
- CLI command conventions (always use `--url ws://localhost:18789`)
- Environment variable mappings
- R2 storage gotchas
- Config schema requirements
- Common task guides (adding endpoints, env vars)
