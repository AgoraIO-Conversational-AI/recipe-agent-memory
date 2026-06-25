# 05 · Workflows

> Step-by-step guides for the common changes in this recipe. Each ends with the narrowest verify command to run.

## Add or change a browser-facing route

1. Add the FastAPI handler in `server/src/server.py` (return the `{ code, msg, data }` envelope).
2. Add the `/api/<name>` → `/<name>` mapping in `web/next.config.ts` `rewrites()`.
3. Add a client helper in `web/src/services/api.ts`.
4. Extend `web/scripts/verify-api-contracts.ts` with the new path + envelope assertions.
5. Verify: `bun run verify:web` (and `bun run verify:local:fastapi` if it should go through the real backend).

## Change the LLM model or greeting

1. Greeting: set `AGENT_GREETING` (env) or edit the default in `server/src/agent.py`.
2. Model: set `OPENAI_MODEL` (default `gpt-4o-mini`).
3. Verify: `bun run verify:backend` (compile) + `cd server && pytest tests -v`.

## Change the STT or TTS vendor/settings

1. Edit the `stt` or `tts` construction in `Agent.start()` (`server/src/agent.py`).
2. Both vendors are Agora-managed; changing model or voice ID needs no new env var unless you want it configurable.
3. Verify: `bun run verify:backend` + `cd server && pytest tests -v`.

## Extend the memory logic

1. Memory functions live in `server/src/memory.py`. Keep the module free of `agora_agent` imports.
2. If you add a new function, add a test in `server/tests/test_memory.py`.
3. If you change `MAX_TURNS` behavior or the schema, update `server/.env.example` and [07_gotchas](07_gotchas.md).
4. Verify: `cd server && pytest tests/test_memory.py -v`.

## Change `system_messages` / memory injection

1. `BASE_SYSTEM` prompt is in `server/src/agent.py`.
2. `build_memory_system_message()` in `memory.py` formats prior turns into a `{"role":"system","content":...}` dict.
3. The list `[base_prompt, memory_context]` is assembled in `Agent.start()`.
4. Verify: `cd server && pytest tests -v`.

## Adjust session parameters (codec, VAD)

1. Edit the `parameters` dict or `turn_detection` config in `Agent.start()` (`audio_scenario`, `data_channel`, `enable_metrics`, VAD thresholds/durations).
2. `output_audio_codec` is accepted per-request via `parameters` on `POST /startAgent`.
3. Verify: `bun run verify:local:fastapi`.

## Run / debug locally

```bash
bun run dev              # both processes
bun run doctor:local     # check creds + .env.local before a live call
```

## Verify before finishing

| Change touches…              | Run                                                                  |
| ---------------------------- | -------------------------------------------------------------------- |
| Web only                     | `bun run verify:web`                                                  |
| Backend logic / vendor config| `bun run verify:backend` + `cd server && pytest tests -v`             |
| Memory logic only            | `cd server && pytest tests/test_memory.py -v`                         |
| Route/proxy boundary         | `bun run verify:web:proxy` and/or `bun run verify:local:fastapi`     |
| Anything end-to-end (local)  | `bun run verify:local`                                                |

## Deploy

1. Deploy `web/` as a Next.js app.
2. Deploy `server/` (or any reachable FastAPI host); the published backend-only image is `ghcr.io/AgoraIO-Conversational-AI/recipe-agent-memory` on `v*` tags.
3. Set `AGENT_BACKEND_URL` in the web deployment so rewrites reach the backend.
4. The SQLite file (`memory.db`) is local to the server process. For multi-instance deployments, point `MEMORY_DB_PATH` at a shared volume.

## Related Deep Dives

- [memory_store_schema](L2/memory_store_schema.md) — `memory.py` internals and cap behavior.
- [session_lifecycle](L2/session_lifecycle.md) — client-side join/teardown, stop fallback.
