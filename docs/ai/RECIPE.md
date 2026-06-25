---
recipe_version: 1.0.0
recipe_status: experimental
extension_points:
  - id: api.routes
    name: Browser-facing API routes
  - id: agent.vendor-config
    name: STT, LLM (model, system_messages, greeting), TTS, and session parameters
  - id: memory.store
    name: SQLite memory schema, MAX_TURNS, and system-message format
  - id: web.conversation-ui
    name: Conversation UI panels and controls (including name handle input)
  - id: verification.contracts
    name: Contract, proxy, and local FastAPI smoke verification
invariants:
  - id: api.rewrite-boundary
    summary: Browser calls stay on /api/* and Next rewrites to FastAPI; no Route Handlers for agent/token logic.
  - id: secrets.server-only
    summary: Agora App Certificate stays in the Python backend; OPENAI_API_KEY is optional (zero-key by default).
  - id: memory.pure-module
    summary: memory.py has no agora_agent import; it must remain unit-testable without the SDK.
  - id: memory.get-history-before-stop
    summary: session.get_history() is called before session.stop(); swapping the order loses history.
  - id: token.uid-concrete
    summary: Backend resolves missing, zero, or negative UIDs before issuing an RTC+RTM token.
stable_contracts:
  - id: env.required
    summary: AGORA_APP_ID and AGORA_APP_CERTIFICATE are required; AGENT_BACKEND_URL is required by deployed web rewrites.
  - id: api.core-routes
    summary: GET /api/get_config, POST /api/startAgent (with optional userKey), and POST /api/stopAgent remain the browser-facing contract.
  - id: response.envelope
    summary: Successful backend responses use { code, msg, data }.
  - id: memory.sqlite
    summary: Memory is stored in SQLite (one row per user_key, turns as JSON array); MEMORY_DB_PATH and MEMORY_MAX_TURNS control location and size.
---

# Recipe Contract

This base recipe defines the reusable surface for a Python-backed Agora Conversational AI **cross-session memory** quickstart: a cascading STT→LLM→TTS pipeline with per-user SQLite memory, behind a Next.js web client.

## Recipe Role

- Role: `base` recipe (self-contained, clone-and-run; no `Extends` pin).
- Target audience: developers building a persistent, context-aware voice agent that remembers users across sessions.
- Reuse model: clone, bind project, set credentials, run, then customize memory behavior, vendor pipeline, or browser UI.

## Recipe Scope

- Python FastAPI token generation and managed agent lifecycle.
- Cascading `DeepgramSTT` → `OpenAI` (Agora-managed, zero-key) → `MiniMaxTTS` vendors.
- Per-user cross-session memory: SQLite store (`memory.py`), `system_messages` injection, `get_history()` capture on stop.
- Next.js browser UI with RTC audio, RTM transcript/metrics, name handle input, connection status.
- Rewrite-only `/api/*` browser facade hiding backend placement.
- Contract, proxy, and local FastAPI smoke verification that need no live Agora calls.

## Baseline Implementation Guidance

Use this repo's source and progressive disclosure docs as the starting point, then customize. Do not recreate the Agora ConvoAI integration from memory — vendor schemas, SDK builder fields, token behavior, RTM details, and `get_history()` timing drift. Copy verified patterns from this repo.

## Extension Points

| ID | Surface | How to extend | Required follow-up |
| -- | ------- | ------------- | ------------------ |
| `api.routes` | `server/src/server.py`, `web/next.config.ts`, `web/src/services/api.ts` | Add FastAPI route, add rewrite, add browser fetch helper. | Extend `web/scripts/verify-api-contracts.ts`; add proxy/fastapi coverage if it belongs in local verification. |
| `agent.vendor-config` | `server/src/agent.py` | Change `OPENAI_MODEL`, STT model/language, TTS voice, `system_messages`, `greeting_message`, `temperature`, `turn_detection`, or the `parameters` dict. | Run `verify:backend` + `pytest tests`; document new env in `server/.env.example` (never add `PORT`). |
| `memory.store` | `server/src/memory.py`, `server/tests/test_memory.py` | Change schema, `MAX_TURNS` logic, or `build_memory_system_message` format. | Keep `memory.py` free of `agora_agent` imports; update or add tests in `test_memory.py`. |
| `web.conversation-ui` | `web/src/components/*`, `web/src/lib/conversation.ts` | Customize pre-call (name handle), transcript, metrics, connection status, mic, or visualizer UI. | Preserve RTC/RTM lifecycle ownership and transcript UID normalization. |
| `verification.contracts` | `web/scripts/*.ts`, root `package.json` | Add checks for new browser/backend boundaries. | Keep checks runnable without live Agora credentials. |

## Invariants

- Browser code calls only `/api/get_config`, `/api/startAgent`, and `/api/stopAgent` for the default flow.
- Next.js owns `/api/*` through rewrites only; no `web/app/api/**/route.ts` for agent/token logic.
- FastAPI owns token generation, `AGORA_APP_CERTIFICATE`, and agent lifecycle.
- `session.get_history()` is always called **before** `session.stop()`; reversing this order loses conversation history.
- `memory.py` has no `agora_agent` import; it must remain testable without the SDK.
- `memory.db` is never committed; it contains user conversation data.
- The backend issues one RTC+RTM-capable token for a concrete non-zero UID.

## Stable Contracts

| Contract | Stable shape |
| -------- | ------------ |
| Required backend env | `AGORA_APP_ID`, `AGORA_APP_CERTIFICATE` |
| Optional backend env | `OPENAI_MODEL`, `OPENAI_API_KEY`, `MEMORY_DB_PATH`, `MEMORY_MAX_TURNS`, `AGENT_GREETING`, `PORT` (env only) |
| Required web deploy env | `AGENT_BACKEND_URL` |
| `GET /api/get_config` | Query `channel?`, `uid?`; returns `data.app_id`, `data.token`, `data.uid`, `data.channel_name`, `data.agent_uid`. |
| `POST /api/startAgent` | Body `{ channelName, rtcUid, userUid, parameters?, userKey? }`; returns `data.agent_id`, `data.channel_name`, `data.status`. |
| `POST /api/stopAgent` | Body `{ agentId }`; returns `{ code: 0, msg: "success" }`. |
| Success envelope | `{ "code": 0, "msg": "success", "data": ... }` where the route has data. |
| Memory schema | Single SQLite table `memory(user_key TEXT PK, turns TEXT, updated_at REAL)`; `turns` is a JSON array of `{role, content}` objects. |
| Verification entry points | `bun run verify:web`, `bun run verify:backend`, `bun run verify:web:proxy`, `bun run verify:local:fastapi`, `bun run verify:local`. |

## Internal / Subject to Change

- Visual layout, component composition, Tailwind classes, and assets under `web/src/components/`.
- Exact model name, STT language, TTS voice, VAD timing, temperature, and greeting text, as long as they stay documented extension points.
- In-memory `Agent._sessions` and `Agent._agent_users` details; the stable behavior is start by channel/user, stop by returned `agent_id`.
- Verification internals under `web/scripts/`; the stable surface is the root script names and what they assert.
- `agora-agents` SDK minor-version behavior; this recipe lower-bounds `>=2.3.0` but does not freeze every field.

## Related Progressive Disclosure Docs

- `L1/01_setup.md` — setup, env, and commands.
- `L1/02_architecture.md` — request flow, memory flow, and topology.
- `L1/05_workflows.md` — common modification workflows.
- `L1/06_interfaces.md` — route, rewrite, env, and memory system-message contracts.
- `L1/L2/memory_store_schema.md` — full SQLite schema and `memory.py` detail.
- `L1/L2/session_lifecycle.md` — RTC/RTM/session orchestration and stop fallback.
