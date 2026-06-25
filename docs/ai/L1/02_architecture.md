# 02 · Architecture

> Two co-located processes. The browser talks only to Next.js `/api/*`, which rewrites to the FastAPI agent backend. The backend owns Agora tokens, agent lifecycle, and a SQLite per-user memory store. OpenAI is Agora-managed (zero-key).

## Topology

```
Browser (localhost:3000)
  │  fetch /api/*
  ▼
Next.js (web/)  ──rewrite──▶  Agent backend (server/, :8000)
                                 │  1. loads prior turns from SQLite for userKey
                                 │  2. builds system_messages = [base_prompt, memory_context]
                                 │  3. starts session: DeepgramSTT → OpenAI(system_messages) → MiniMaxTTS
                                 ▼
                              Agora ConvoAI Cloud
                                 │  user speech → Deepgram STT (managed, nova-3, en)
                                 │  text → OpenAI (Agora-managed, keyless, gpt-4o-mini default)
                                 │  response → MiniMax TTS (managed, English_captivating_female1)
                                 ▼
                              User hears warm, context-aware responses
                              RTM transcript + metrics → web UI

On stop:  session.get_history() ──▶  SQLite (memory.db)
          (captured BEFORE session.stop())
```

- **`web/`** — Next.js 16 / React 19 / TypeScript. Owns UI plus the RTC/RTM client lifecycle. Calls only `/api/*`.
- **`server/`** — Python FastAPI (:8000). Owns Agora token generation, agent session lifecycle, and the SQLite memory store. SDK: `agora-agents>=2.3.0` (`import agora_agent`).
- No `llm/` service — OpenAI is Agora-managed and requires no API key.

## Request lifecycle

1. Browser `GET /api/get_config` → Next rewrites to backend `/get_config`; backend mints a Token007 from `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE` and returns channel + UIDs.
2. User enters an optional name handle in the UI. Browser `POST /api/startAgent` with `userKey`.
3. Backend opens the SQLite store, calls `get_memory(conn, userKey)` for prior turns, builds `system_messages`, and starts the cascading-vendor agent session.
4. Agora routes user audio through Deepgram STT → Agora-managed OpenAI → MiniMax TTS → back to the channel.
5. RTM delivers transcript + metrics to the web UI.
6. `POST /api/stopAgent { agentId }` triggers history capture: `await session.get_history()` **before** `session.stop()`, then `save_memory()` persists turns to SQLite.

## Why no `llm/` service

This recipe uses the **managed OpenAI vendor** (`agora_agent.agentkit.vendors.OpenAI`). Agora holds the OpenAI API key on its cloud — no separate LLM service, no tunnel. The only required credentials are `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE`. `OPENAI_API_KEY` is optional (BYO if needed).

## Key abstractions

- **`Agent`** (`server/src/agent.py`) — async wrapper around `AgoraAgent`; owns the `AsyncAgora` client, `_sessions` map (keyed by `agent_id`), and `_agent_users` map (agent_id → user_key for memory capture).
- **`memory`** (`server/src/memory.py`) — pure module (no `agora_agent` import); functions: `get_db`, `get_memory`, `save_memory`, `build_memory_system_message`.
- **`turn_detection`** — configured on `AgoraAgent(...)` using VAD mode (start/end-of-speech with silence duration). This is a cascading-vendor recipe; `turn_detection` lives on the agent, not on the LLM vendor.
- **Rewrite proxy** (`web/next.config.ts`) — the only browser→backend boundary; no Next Route Handlers for agent/token logic.

## Tech decisions

- **Rewrites, not Route Handlers** — hides backend placement behind `/api/*` so the same client works locally and deployed (set `AGENT_BACKEND_URL`).
- **`memory.py` stays pure** — no `agora_agent` import, so it can be unit-tested without the SDK (`test_memory.py` runs standalone).
- **get_history before stop** — `session.get_history()` only works while the session is active; `agent.stop()` pops the session and captures history before calling `session.stop()`.
- **Cascading vendors** — DeepgramSTT + OpenAI + MiniMaxTTS wired via `.with_stt()`, `.with_llm()`, `.with_tts()`.

## Related Deep Dives

- [memory_store_schema](L2/memory_store_schema.md) — SQLite schema, `memory.py` API, `MAX_TURNS` cap, and the get_history capture window.
- [session_lifecycle](L2/session_lifecycle.md) — browser orchestration of config + start/stop, RTC/RTM, transcript mapping, and stop fallback.
