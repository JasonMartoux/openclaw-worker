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
| `src/gateway/process.ts` | Find/start gateway processes in container |
| `src/gateway/env.ts` | Build container environment variables |
| `src/gateway/sync.ts` | R2 backup/restore sync logic |
| `Dockerfile` | Container image (Node 22 + clawdbot CLI) |
| `start-moltbot.sh` | Container startup script |
| `wrangler.jsonc` | Cloudflare Worker configuration |

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
```

Note: WebSocket proxying has issues with `wrangler dev`. Deploy to Cloudflare for full functionality.

## Sandbox Container Lifecycle

The `Sandbox` class in `src/index.ts` extends `@cloudflare/sandbox` with error handling for internal alarms.

**Container sleep behavior** is controlled by `SANDBOX_SLEEP_AFTER`:
- `never` (default): Container stays alive indefinitely (`keepAlive: true`)
- Duration (e.g., `10m`, `1h`): Container sleeps after inactivity period

**Important**: The cron backup runs every 15 minutes. If `SANDBOX_SLEEP_AFTER < 15m`, the container will cold start on each backup. Keep it `> 15m` or use the default `never`.

**Known issue**: The `@cloudflare/sandbox` package has a race condition with internal alarms that causes `TypeError: Cannot read properties of undefined (reading 'isRetry')`. The custom `Sandbox` class wraps the alarm handler to catch and suppress these errors.

## Additional Guidelines

See `AGENTS.md` for detailed patterns including:
- CLI command conventions (always use `--url ws://localhost:18789`)
- Environment variable mappings
- R2 storage gotchas
- Config schema requirements
- Common task guides (adding endpoints, env vars)
