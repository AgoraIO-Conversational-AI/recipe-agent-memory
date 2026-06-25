# 08 · Security

> Trust boundaries, secret handling, SQLite data privacy, and auth for the memory recipe.

## Trust boundaries

| Hop                            | Auth                                                                  |
| ------------------------------ | --------------------------------------------------------------------- |
| Browser → agent backend        | None in local dev (the `/api/*` rewrite is same-origin).              |
| Agent backend → Agora cloud    | Token007, generated from `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE`.    |
| Agora cloud → OpenAI           | Agora-managed key (transparent); optional `OPENAI_API_KEY` for BYO.   |

## Secret handling

- **Server-only secrets:** `AGORA_APP_CERTIFICATE` lives only in `server/.env.local` and never reaches the browser. The browser receives a short-lived token, never the certificate.
- `OPENAI_API_KEY` is optional; if provided, it stays in `server/.env.local` (server-side only).
- `server/.env.local` is gitignored; `server/.env.example` ships placeholders only.
- Tokens (`generate_convo_ai_token`) expire after 3600s and are minted per `get_config` call for a concrete non-zero UID.

## SQLite data privacy

- `memory.db` contains user conversation turns keyed by name handle (plain text). It is gitignored.
- **Never commit `memory.db`.** Do not loosen the gitignore for this file.
- For multi-instance deployments, `MEMORY_DB_PATH` should point to a shared volume with appropriate access controls.
- To reset all memory, delete `memory.db`; the store re-creates itself on next `get_db()` call.

## CORS

The backend sets `CORSMiddleware` with `allow_origins=["*"]` — open by design for a local/dev recipe. **Lock this down to known origins before any production deployment.**

## Validation

- `Agent.start()` rejects empty `channel_name` and non-positive `agent_uid`/`user_uid` before issuing tokens or starting a session.
- Route errors are sanitized: `_log_route_error` logs only non-`None` context; exceptions map to 400/500 without leaking internals to the client beyond the message.
- `userKey` is accepted as-is (no server-side sanitization beyond empty-string guard). Treat it as an untrusted identifier; avoid using it as a filesystem path.

## Deployment notes

- Set `AGENT_BACKEND_URL` only to a backend you control; the rewrite forwards browser requests there verbatim.
- The published Docker image is **backend-only** (`:8000`); it does not bundle secrets.
- In Docker, the default `MEMORY_DB_PATH` (`memory.db`) writes to the container filesystem. Mount a volume at `/tmp/memory.db` or override `MEMORY_DB_PATH` to persist memory across container restarts.

## Related Deep Dives

- None.
