# Progressive Disclosure — Test Results

> Test run for `recipe-agent-memory` progressive disclosure docs.
> Date: 2026-06-25 · Standard: AgoraIO-Community/ai-devkit progressive-disclosure.

## Step 1 — Structural checks

| Check                                                      | Result           |
| ---------------------------------------------------------- | ---------------- |
| `L0_repo_card.md` ≤ 50 lines                               | Pass (36)        |
| All 8 L1 files present                                     | Pass             |
| Each L1 has purpose blockquote + Related Deep Dives        | Pass (8/8)       |
| L1 line counts in 80–200 target                            | **Below target** (45–85) — see note |
| L2 `_index.md` present                                     | Pass             |
| Each L2 opens with "When to Read This" callout             | Pass (2/2)       |
| Relative links resolve (`docs/ai/` + AGENTS.md)            | Pass (41/41, 0 broken) |
| AGENTS.md has How to Load / Git Conventions / Doc Commands | Pass             |

**Note on L1 line counts:** files are table-dense and information-complete but run 45–85 lines, mostly under the 80–200 soft target. The standard favors tables over prose and warns against bloat, so they were left concise rather than padded. Accepted deviation; revisit if a section needs more depth.

## Step 2/3 — Question runs

Questions span the five standard categories. Each answer was checked against the repo source before being marked Pass. "Level" is the lowest disclosure level that fully answers the question.

### Setup & Build

| # | Question | Expected answer | Source of truth | Level | Status |
|---|----------|-----------------|-----------------|-------|--------|
| 1 | How do I install and run it locally? | `bun run setup` then `bun run dev` (backend :8000 + web :3000). | `L1/01_setup.md` ↔ `package.json` scripts | L1 | Pass |
| 2 | Which env vars are required? | `AGORA_APP_ID`, `AGORA_APP_CERTIFICATE`. `OPENAI_API_KEY` is optional (zero-key). | `L1/01_setup.md`, `06_interfaces.md` ↔ `agent.py`, `.env.example` | L1 | Pass |
| 3 | Is this zero-key? | Yes — OpenAI is Agora-managed; `OPENAI_API_KEY` is optional. | `L1/01_setup.md` ↔ `README.md`, `agent.py` | L1 | Pass |

### Test & Run

| # | Question | Expected answer | Source of truth | Level | Status |
|---|----------|-----------------|-----------------|-------|--------|
| 4 | How do I run backend tests without cloud creds? | `cd server && pytest tests -v`; `conftest.py` fakes env; `test_memory.py` uses temp SQLite. | `L1/04_conventions.md`, `01_setup.md` ↔ `tests/conftest.py`, `test_memory.py` | L1 | Pass (ran: 6 passed) |
| 5 | What's the narrowest gate for a memory-only change? | `cd server && pytest tests/test_memory.py -v`. | `L1/05_workflows.md` ↔ `package.json` | L1 | Pass |
| 6 | What does `verify:local:fastapi` do? | Spawns real FastAPI with `FakeAgent` and proxies routes through the rewrite map. | `L1/03_code_map.md`, `05_workflows.md` ↔ `web/scripts/verify-local-fastapi.ts` | L1 | Pass |

### Conventions

| # | Question | Expected answer | Source of truth | Level | Status |
|---|----------|-----------------|-----------------|-------|--------|
| 7 | What response shape do backend routes use? | `{ code, msg, data }`; `data` only when there's a payload. | `L1/04_conventions.md`, `06_interfaces.md` ↔ `server.py` | L1 | Pass |
| 8 | Why must `memory.py` not import `agora_agent`? | Keeps the module testable without the SDK installed. | `L1/04_conventions.md`, `07_gotchas.md` ↔ `memory.py`, `test_memory.py` | L1 | Pass |
| 9 | What are the commit/branch conventions? | Conventional commits `type: description`; branches `type/short-description`; no AI tool names. | `AGENTS.md` Git Conventions | L1 | Pass |

### Development

| # | Question | Expected answer | Source of truth | Level | Status |
|---|----------|-----------------|-----------------|-------|--------|
| 10 | How do I add a new browser-facing route? | Add FastAPI handler → add rewrite in `next.config.ts` → add client helper → extend `verify-api-contracts.ts`. | `L1/05_workflows.md` ↔ source | L1 | Pass |
| 11 | Where does memory recall happen and when? | `Agent.start()` opens SQLite, calls `get_memory()`, builds `system_messages`, passes to `OpenAI(...)`. | `L1/02_architecture.md`, `06_interfaces.md` ↔ `agent.py` | L1 | Pass |
| 12 | Where does memory persist and when? | `Agent.stop()`: calls `session.get_history()` BEFORE `session.stop()`, then `save_memory()`. | `L1/07_gotchas.md` ↔ `agent.py` | L1 | Pass |

### Deep Dive

| # | Question | Expected answer | Source of truth | Level | Status |
|---|----------|-----------------|-----------------|-------|--------|
| 13 | What is the SQLite schema and how are turns merged? | Table `memory(user_key PK, turns TEXT, updated_at REAL)`; `save_memory` appends then takes `[-MAX_TURNS:]`. | `L2/memory_store_schema.md` ↔ `memory.py` | L2 | Pass |
| 14 | What happens if `session.get_history()` is called after `session.stop()`? | History is unavailable — the capture window closes when the session stops. Idle-timeout sessions bypass the capture block. | `L2/memory_store_schema.md`, `07_gotchas.md` ↔ `agent.py` | L2 | Pass |
| 15 | How does stop survive a backend restart? | `_sessions` is in-memory; missing session falls back to `client.stop_agent(agent_id)`. Memory for that session is lost (no session → no history capture). | `L2/session_lifecycle.md` ↔ `agent.py` | L2 | Pass |

## Step 4 — Analysis

- All 15 questions answered at the expected disclosure level (12 at L1, 3 at L2). No "correct but needed L2 unnecessarily" or "wrong/missing L2" cases.
- No missing-coverage findings; no broken references.
- One soft deviation: L1 line counts below the 80–200 target (accepted; concise/table-dense).

## Step 5 — Summary

| Category       | Questions | Pass | Notes |
| -------------- | :-------: | :--: | ----- |
| Setup & Build  | 3 | 3 | — |
| Test & Run     | 3 | 3 | backend tests executed: 6 passed |
| Conventions    | 3 | 3 | — |
| Development    | 3 | 3 | — |
| Deep Dive      | 3 | 3 | resolved at L2 as designed |
| **Total**      | **15** | **15** | — |

## Step 6 — Fixes / Retest

No failing questions; no fixes required. Evidence executed during this run:

- `pytest tests -v` in throwaway venv `/tmp/v_memory` → `6 passed`.
- Relative link check → `41 checked, 0 broken`.
