# ai-gateway

// screenpipe — AI that knows everything you've seen, said, or heard
// https://screenpipe.com

Cloudflare Worker that fans out chat/embedding/audio requests to multiple LLM providers with unified auth, cost accounting, and A/B routing. Deployed at `api.screenpi.pe` and `api.screenpipe.com`.

## Providers

Selected in `src/providers/index.ts::createProvider()` by model-string substring match:

| Provider | Model match | Backend |
|---|---|---|
| Anthropic | `claude*` | Anthropic API (`ANTHROPIC_API_KEY`) |
| Gemini / Vertex | `gemini*` | Vertex AI service-account when `VERTEX_SERVICE_ACCOUNT_JSON` set, else `GEMINI_API_KEY` |
| Vertex MaaS | `glm-*`, `kimi-*`, `llama-*`, `qwen-*` | Vertex AI Model-as-a-Service (`VERTEX_SERVICE_ACCOUNT_JSON`) |
| Tinfoil | `tinfoil-*` | Tinfoil TEE (`TINFOIL_API_KEY`) |
| Screenpipe enclave | `screenpipe-enclave-*` | Tinfoil-hosted CVM (`SCREENPIPE_ENCLAVE_API_KEY` or `TINFOIL_API_KEY`) |
| Screenpipe event classifier | `screenpipe-event-classifier` | Self-hosted vLLM (`EVENT_CLASSIFIER_URL`) |
| OpenAI (default) | anything else | OpenAI API (`OPENAI_API_KEY`) |
| OpenRouter | via aliasing in `resolveModelAlias` | `OPENROUTER_API_KEY` |

Provider tokens load from Cloudflare Worker secrets (`wrangler secret put`) and non-secret vars from `wrangler.toml` `[vars]`. Local dev copies `.dev.vars.example` → `.dev.vars`.

## Endpoints

Main handlers live in `src/handlers/`:

- `POST /v1/chat/completions` — OpenAI-compatible chat, routed by model
- `POST /v1/messages` — Anthropic-compatible chat (Vertex proxy, see `VERTEX_PROXY_README.md`)
- `POST /v1/embeddings` — OpenAI-compatible embeddings
- `POST /v1/audio/transcriptions` — audio transcription; A/B/C split across Deepgram / Whisper / Parakeet via `DEEPGRAM_TRAFFIC_PCT` / `WHISPER_TRAFFIC_PCT` / `PARAKEET_TRAFFIC_PCT` env vars
- `POST /v1/realtime` — realtime meeting transcription (Deepgram)

## Auth

Clerk-backed via `@clerk/backend` (`CLERK_SECRET_KEY`). Tiers gate per-user daily $ cost (`MAX_DAILY_COST_PER_USER` × tier multiplier: subscribed 7×, logged_in 0.64×, anonymous 0.32×).

## Difficulty routing

`ROUTER_MODE` env var (Worker Vars, no redeploy) controls interactive-auto model selection:

- `off` — always `glm-5` (default)
- `heuristic` — regex tiering, zero latency
- `embedding` — bge-base-en-v1.5 centroid classifier via Workers AI (`AI` binding)

`ROUTER_SAMPLE_PCT` selects the A/B split % (deterministic by device hash).

## Local dev

```bash
cd packages/ai-gateway
bun install
cp .dev.vars.example .dev.vars   # fill in secrets
bun run dev                       # wrangler dev on :8787
```

## Deploy

```bash
bun run deploy         # bash scripts/deploy.sh — uploads source maps to Sentry
# or
bun run deploy:no-sourcemaps
```

Routes bound in `wrangler.toml`: `api.screenpi.pe/*` and `api.screenpipe.com`.

## Related

- `packages/screenpipe-mcp/` — MCP server that talks to the local screenpipe API (not this gateway)
- `crates/screenpipe-connect/` — Rust client for the gateway
- `VERTEX_PROXY_README.md` — Vertex AI `/v1/messages` proxy details
