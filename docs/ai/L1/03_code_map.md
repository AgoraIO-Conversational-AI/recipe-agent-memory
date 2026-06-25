# 03 · Code Map

> Where things live. Two top-level modules: `web/` (Next.js client) and `server/` (FastAPI backend). Orchestration is in the root `package.json`.

## Root

| Path                  | Responsibility                                                        |
| --------------------- | --------------------------------------------------------------------- |
| `package.json`        | Bun workspace; `setup`, `dev`, `doctor*`, `verify*`, `clean` scripts. |
| `README.md`           | Setup, run modes, env, troubleshooting (including memory gotchas).    |
| `ARCHITECTURE.md`     | System shape, memory flow, and component boundaries.                  |
| `AGENTS.md`           | Coding-agent handbook + How to Load / Git Conventions / Doc Commands. |
| `Dockerfile`          | Backend-only image (`:8000`).                                         |
| `.github/workflows/`  | `ci.yml` (backend pytest matrix + web verify), `docker.yml`, `nightly.yml`. |

## `server/` — FastAPI backend (:8000)

| Path                              | Responsibility                                                                       |
| --------------------------------- | ------------------------------------------------------------------------------------ |
| `src/server.py`                   | FastAPI app, CORS, route handlers, error mapping, uvicorn entrypoint.                |
| `src/agent.py`                    | `Agent` class: `AsyncAgora` client, `start()`/`stop()`, `_sessions`, `_agent_users`. |
| `src/memory.py`                   | Pure SQLite memory store: `get_db`, `get_memory`, `save_memory`, `build_memory_system_message`. No `agora_agent` import. |
| `scripts/run_fake_server.py`      | Boots `server.app` with a `FakeAgent` for the local FastAPI smoke test.              |
| `tests/test_memory.py`            | 5 standalone unit tests for `memory.py`; no SDK, no cloud, no creds needed.         |
| `tests/test_agent_construction.py`| Builds the real `AgoraAgent`, fakes the SDK session, asserts start/stop shape.      |
| `tests/conftest.py`               | `fake_env` fixture; neutralises dotenv and injects `AGORA_APP_ID`/`AGORA_APP_CERTIFICATE`. |
| `.env.example`                    | Env template including `MEMORY_DB_PATH`, `MEMORY_MAX_TURNS` (do not add `PORT`).    |
| `requirements.txt`                | Runtime deps: FastAPI, uvicorn, agora-agents, python-dotenv, socksio.                |
| `requirements-dev.txt`            | Dev deps: pytest.                                                                    |

## `server/src/server.py` routes

- `GET /get_config` — token + channel/UID config.
- `POST /startAgent` — start the memory-enabled agent session (optional `userKey` triggers memory recall).
- `POST /stopAgent` — stop by `agent_id`, capture and persist memory before stopping.

## `web/` — Next.js client (:3000)

| Path                                      | Responsibility                                                    |
| ----------------------------------------- | ----------------------------------------------------------------- |
| `next.config.ts`                          | `/api/*` rewrites to `AGENT_BACKEND_URL`; strict mode; Turbopack root. |
| `src/services/api.ts`                     | Browser API client: `getConfig`, `startAgent` (with optional `userKey`), `stopAgent`. |
| `src/lib/conversation.ts`                 | Transcript normalization, timestamp/UID mapping, visualizer state.|
| `src/lib/agora.ts`                        | Agora RTC/RTM helpers.                                            |
| `src/components/LandingPage.tsx`          | Conversation entry: config fetch, name handle input, agent start, RTM login, teardown. |
| `src/components/ConversationComponent.tsx`| RTC join, mic publish, transcript/metrics/state listeners.        |
| `src/components/Quickstart*.tsx`          | Pre-call, transcript, metrics, layout panels.                     |
| `scripts/verify-api-contracts.ts`         | Asserts rewrites + client paths + response envelope (no network). |
| `scripts/verify-local-proxy.ts`           | Stub backend; proxies `/api/*` through the rewrite map.           |
| `scripts/verify-local-fastapi.ts`         | Spawns real FastAPI with `FakeAgent`; proxies routes end-to-end.  |
| `scripts/doctor.ts`                       | Web prerequisite check.                                           |

## Related Deep Dives

- None. For runtime flow see [02_architecture](02_architecture.md); for contracts see [06_interfaces](06_interfaces.md).
