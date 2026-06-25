# Deep Dive — Session Lifecycle

> **When to Read This:** You are touching client-side join, token renewal, RTC/RTM wiring, transcript handling, mid-call control, or the stop fallback path. For the contracts these calls hit, see [06_interfaces](../06_interfaces.md).

The browser owns the full RTC/RTM client lifecycle; the backend owns tokens, the agent session, and memory persistence. The two meet only at `/api/*`.

## End-to-end flow

1. **Config** — `LandingPage.tsx` calls `getConfig()` (`web/src/services/api.ts`) → `GET /api/get_config`. Backend mints a Token007 (RTC+RTM, 3600s) for a concrete non-zero UID and returns `{ app_id, token, uid, channel_name, agent_uid }`.
2. **Join** — `ConversationComponent.tsx` joins the RTC channel with the returned token/UID, publishes the microphone, and logs in to RTM.
3. **Start agent** — user enters an optional name handle. `startAgent(channelName, rtcUid, userUid, userKey?)` → `POST /api/startAgent`. Backend loads prior memory for `userKey`, builds `system_messages`, starts the cascading-vendor session, and returns `agent_id`.
4. **Converse** — user audio flows through Deepgram STT → Agora-managed OpenAI (with memory context) → MiniMax TTS → back to the channel. RTM delivers transcript + metrics.
5. **Stop** — `stopAgent(agentId)` → `POST /api/stopAgent`. Backend captures history (`session.get_history()`) **before** stopping, persists to SQLite, then calls `session.stop()`. The client also releases RTC/RTM media on end-call.

## Backend session bookkeeping

`Agent` (`server/src/agent.py`) keeps two in-memory maps:

- `self._sessions[agent_id] = session` — the active session object.
- `self._agent_users[agent_id] = user_key` — the name handle (if provided) for memory capture on stop.

`stop(agent_id)`:
1. Pops both maps.
2. Calls `session.get_history()` if session + user_key both exist.
3. Persists turns via `save_memory()`.
4. Calls `session.stop()`.
5. If the session is missing (e.g. process restarted), falls back to `self.client.stop_agent(agent_id)` — the stateless cloud path. This is why stop is robust across restarts, but `_sessions` itself is **not** a durable store.

## Memory in the start path

Before starting the vendor session, `Agent.start()`:
1. Opens a SQLite connection via `memory.get_db()`.
2. Calls `memory.get_memory(conn, user_key)` to retrieve prior turns.
3. Calls `memory.build_memory_system_message(turns)` to build a `{"role":"system","content":...}` dict (or `None` for new users).
4. Assembles `system_messages = [BASE_SYSTEM, memory_msg]` and passes it to `OpenAI(system_messages=...)`.
5. Closes the connection.

## Transcript handling (`web/src/lib/conversation.ts`)

- `normalizeTranscript(transcript, localUid)` — maps `uid === '0'` to the local UID and runs `normalizeTranscriptSpacing` on text.
- `normalizeTimestampMs(ts)` — promotes second-precision timestamps to ms.
- `getMessageList` / `getCurrentInProgressMessage` — split finalized vs in-progress turns (by `TurnStatus.IN_PROGRESS`).
- `mapAgentVisualizerState(agentState, isConnected, connectionState)` — maps SDK state → UIKit visualizer state (`joining`, `listening`, `analyzing`, `talking`, `ambient`, `disconnected`).

## Token renewal

Tokens expire at 3600s. The client re-fetches config / renews as needed in `LandingPage.tsx`; renewal uses the same `get_config` contract. Keep renewal client-side — the backend stays stateless about who is connected.

## What stays where

- **Client owns:** RTC join, mic publish, RTM login, transcript/metrics/state listeners, token renewal, explicit end-call media release.
- **Backend owns:** token minting, vendor build, session start/stop, memory load/persist.
- Do not move token or memory logic into the web app or add Route Handlers for it (see [07_gotchas](../07_gotchas.md)).

## Related L1

- [02_architecture](../02_architecture.md) · [03_code_map](../03_code_map.md) · [06_interfaces](../06_interfaces.md)
