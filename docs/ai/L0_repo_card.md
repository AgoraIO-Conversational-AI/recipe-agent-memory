# recipe-agent-memory — Repo Card

> Next.js web client + Python FastAPI backend for an Agora Conversational AI voice agent with cross-session memory. A user's conversation history is stored in SQLite per name handle and re-injected via `system_messages` on subsequent sessions. Fully zero-key (Agora-managed OpenAI, no `OPENAI_API_KEY` required).

## Identity

| Field          | Value                                                                        |
| -------------- | ---------------------------------------------------------------------------- |
| Repo           | `AgoraIO-Conversational-AI/recipe-agent-memory`                              |
| Type           | `distributed-system` (single repo, two co-located processes)                 |
| Language       | Python 3.10+ (FastAPI + uvicorn) backend + Next.js 16 / React 19 web         |
| Deploy Target  | `web/` as Next.js app, `server/` as a reachable FastAPI service              |
| Owner          | Agora Conversational AI DevEx                                                |
| Last Reviewed  | 2026-06-25                                                                   |
| Recipe Role    | `base`                                                                       |
| Recipe Version | `1.0.0`                                                                      |
| Recipe Status  | `experimental`                                                               |

## L1 — Summaries

The Audience column helps agents prioritise: **Use** = consuming the recipe's behavior, **Maintain** = modifying internals.

| File                                     | Purpose                                                                              | Audience       |
| ---------------------------------------- | ------------------------------------------------------------------------------------ | -------------- |
| [01_setup](L1/01_setup.md)               | bun + venv + pip setup, env vars (memory-specific: `MEMORY_DB_PATH`, `MEMORY_MAX_TURNS`), commands | Use & Maintain |
| [02_architecture](L1/02_architecture.md) | Two-process topology, `/api/*` rewrite proxy, memory flow, STT→LLM→TTS pipeline     | Maintain       |
| [03_code_map](L1/03_code_map.md)         | `web/` and `server/` trees including `memory.py` and its tests                      | Maintain       |
| [04_conventions](L1/04_conventions.md)   | Python async + FastAPI patterns, Biome, JSON envelope, `memory.py` purity rule       | Maintain       |
| [05_workflows](L1/05_workflows.md)       | Add a route, change LLM/STT/TTS config, extend memory logic, verify, deploy          | Use            |
| [06_interfaces](L1/06_interfaces.md)     | FastAPI route contracts, rewrites, env vars, `StartAgentRequest.userKey` field       | Use & Maintain |
| [07_gotchas](L1/07_gotchas.md)           | `get_history()` capture window, idle-timeout loss, `memory.py` import rule, `PORT`   | Maintain       |
| [08_security](L1/08_security.md)         | Token007, App Certificate server-only, SQLite file privacy, CORS, `memory.db` gitignore | Maintain   |

## Recipe Profile

This repo declares `Recipe Role: base`. See [RECIPE.md](RECIPE.md) for extension points, invariants, and stable contracts before changing reusable surfaces.
