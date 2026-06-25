# 06 · Interfaces

> Boundary contracts: backend routes, the `/api/*` rewrite map, env vars, the response envelope, and the `StartAgentRequest.userKey` memory field.

## Backend routes (port 8000)

The browser calls these as `/api/<name>`; Next rewrites to the backend `/<name>`.

### `GET /get_config`

- Query (optional): `channel?: string`, `uid?: int` (≤ 0 or missing → backend generates one).
- Returns `data`: `{ app_id, token, uid (string), channel_name, agent_uid (string) }`.
- Token is a Token007 RTC+RTM token, expiry 3600s, for a concrete non-zero UID.

### `POST /startAgent`

- Body: `{ channelName: string, rtcUid: int, userUid: int, parameters?: object, userKey?: string }`.
  - `parameters.output_audio_codec?: string` is the only honored parameter field.
  - `userKey`: optional user handle string; triggers memory recall when present and non-empty.
- Returns `data`: `{ agent_id, channel_name, status: "started" }`.
- 400 if `channelName`/`rtcUid`/`userUid` invalid.

### `POST /stopAgent`

- Body: `{ agentId: string }`.
- Returns `{ code: 0, msg: "success" }` (no `data`).
- Before stopping, the backend captures `session.get_history()` and persists turns to SQLite (if `userKey` was provided on start).

## Response envelope

```json
{ "code": 0, "msg": "success", "data": { } }
```

`data` omitted when the route has no payload. Non-zero `code` or missing `data` = error on the client side.

## Rewrite map (`web/next.config.ts`)

| Browser path        | Backend destination |
| ------------------- | ------------------- |
| `/api/get_config`   | `/get_config`       |
| `/api/startAgent`   | `/startAgent`       |
| `/api/stopAgent`    | `/stopAgent`        |

`rewrites()` returns `[]` when `AGENT_BACKEND_URL` is unset. The contract is asserted by `verify-api-contracts.ts` and exercised by `verify-local-proxy.ts`.

## Browser API client (`web/src/services/api.ts`)

- `getConfig({ channel?, uid? }) → GetConfigResponse`
- `startAgent(channelName, rtcUid, userUid, userKey?) → agent_id`
  - `userKey` is trimmed before sending; empty string is treated as absent.
- `stopAgent(agentId) → void`

## Environment variables

| Variable                | Scope          | Required | Default           |
| ----------------------- | -------------- | :------: | ----------------- |
| `AGORA_APP_ID`          | backend        |    ✅    | —                 |
| `AGORA_APP_CERTIFICATE` | backend        |    ✅    | —                 |
| `OPENAI_MODEL`          | backend        |          | `gpt-4o-mini`     |
| `OPENAI_API_KEY`        | backend        |          | — (Agora-managed) |
| `MEMORY_DB_PATH`        | backend        |          | `memory.db`       |
| `MEMORY_MAX_TURNS`      | backend        |          | `20`              |
| `AGENT_GREETING`        | backend        |          | built-in line     |
| `AGENT_BACKEND_URL`     | web (deploy)   |    ✅\*   | `http://localhost:8000` (dev) |
| `PORT`                  | backend (env only) |      | `8000` — do **not** put in `.env.example` |

\* Required wherever the web app is deployed; rewrites are empty without it.

## Memory system message format

`build_memory_system_message(turns)` in `memory.py` produces:

```json
{
  "role": "system",
  "content": "You are talking to a RETURNING user. Here is what you remember from past conversations..."
}
```

Returns `None` when `turns` is empty (no prior memory for this handle). The LLM receives `system_messages = [BASE_SYSTEM, memory_msg]` (or just `[BASE_SYSTEM]` for new users).

## Related Deep Dives

- [memory_store_schema](L2/memory_store_schema.md) — full SQLite schema and `memory.py` function signatures.
