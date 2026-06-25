# 04 · Conventions

> Coding patterns shared across `server/` and `web/`. Follow these to keep local and deployed modes aligned.

## Boundary ownership

- Browser code calls only `/api/*`. Backend placement is hidden behind Next rewrites (`web/next.config.ts`).
- **Never** add `web/app/api/**/route.ts` for agent/token logic — `verify-api-contracts.ts` fails the build if a `route.ts` appears under `app/api`.
- Token generation and the App Certificate stay in `server/`.

## Backend (Python / FastAPI)

- Async throughout: route handlers are `async def`; the agent uses `AsyncAgora` and `create_async_session`.
- Request bodies are Pydantic models (`StartAgentRequest`, `StopAgentRequest`). Field names are **camelCase** (`channelName`, `rtcUid`, `userUid`, `userKey`) to match the browser client.
- Error mapping is centralized: `_to_http_error()` maps `ValueError → 400`, `RuntimeError → 500`, else 500. `_log_route_error()` logs with safe context + traceback. Raise plain `ValueError`/`RuntimeError`; let the route convert.
- Logging via `logging.getLogger("uvicorn.error")`.
- Env read with `os.getenv`; `.env.local` then `.env` loaded with `override=True`.

## Response envelope

All backend JSON responses use:

```json
{ "code": 0, "msg": "success", "data": { } }
```

`data` is present only when the route returns a payload. The browser client treats `code !== 0` (or missing `data`) as an error.

## `memory.py` purity rule

`server/src/memory.py` must never import `agora_agent`. This keeps the module testable without the Agora SDK installed. All memory logic (SQLite open/create, get, save, build system message) stays in this module; `agent.py` calls it.

## Vendor pipeline (cascading STT → LLM → TTS)

- STT: `DeepgramSTT(model="nova-3", language="en")` via `.with_stt()`.
- LLM: `OpenAI(model=OPENAI_MODEL, system_messages=..., greeting_message=..., temperature=0.7)` via `.with_llm()`.
- TTS: `MiniMaxTTS(model="speech_2_6_turbo", voice_id="English_captivating_female1")` via `.with_tts()`.
- `turn_detection` is set on `AgoraAgent(...)` directly (VAD mode with `speech_threshold`, `start_of_speech`, `end_of_speech`).

## Web (TypeScript / Next.js)

- Lint/format with Biome (`bun run lint`, `bun run lint:fix` in `web/`).
- RTC client creation must be StrictMode-safe (strict mode is on).
- Transcript speaker mapping uses real UIDs (`normalizeTranscript` maps `uid === '0'` to the local UID); do not heuristically guess speakers.
- API client lives in `src/services/api.ts`; UI never calls `fetch` to the backend directly.
- `startAgent(channelName, rtcUid, userUid, userKey?)` — `userKey` is optional; omit or pass empty to skip memory recall.

## Testing approach

- Backend: `pytest` in `server/`, standalone — `conftest.py` fakes env; `test_memory.py` (5 tests) runs against a temp SQLite file; `test_agent_construction.py` fakes the SDK session only.
- Web: contract/proxy/fastapi smoke scripts under `web/scripts/` run without live Agora calls.
- Run the **narrowest** relevant verify command before finishing (see [05_workflows](05_workflows.md)).

## Doc upkeep

When you change request/response contracts, env vars, memory schema, or workflow, update the web client, backend, contract checks, README, **and** the matching `docs/ai/L1/` file together, then bump `Last Reviewed` in [L0](../L0_repo_card.md).

## Related Deep Dives

- [memory_store_schema](L2/memory_store_schema.md) — SQLite schema detail and `memory.py` API.
