# 07 · Gotchas

> Non-obvious pitfalls specific to the memory recipe. Read before changing the agent, memory store, env, or verify scripts.

## `get_history()` must be called before `session.stop()`

`session.get_history()` only works while the agent session is still active. `Agent.stop()` therefore:
1. Pops the session and user_key from `_sessions` / `_agent_users`.
2. Calls `await session.get_history()`.
3. Persists turns via `save_memory()`.
4. Calls `await session.stop()`.

**Do not reorder these steps.** Calling `stop()` first makes history unavailable.

## Idle-timeout sessions lose their history

If the session times out due to 30 s of silence, the SDK calls its own stop path — the history-capture block in `Agent.stop()` is bypassed. This is a known limitation: end conversations via the end-call button to ensure memory is saved. Do not attempt to work around this by increasing `idle_timeout` without testing the downstream SDK behavior.

## `memory.py` must never import `agora_agent`

The module is unit-tested in `test_memory.py` without the Agora SDK installed. Any `agora_agent` import will break that test run. Keep all Agora SDK usage in `agent.py`.

## Never commit `memory.db`

The SQLite file contains user conversation data. It is gitignored. Do not add it to commits or loosen the gitignore.

## `OPENAI_API_KEY` is optional, not required

This recipe is zero-key by default — Agora manages the OpenAI key. `Agent.__init__` reads `OPENAI_API_KEY` with `os.getenv` but does not raise if it is absent. Pass it only when your Agora account requires BYO keys.

## Do not put `PORT` in `server/.env.example`

`verify:local:fastapi` injects a random `PORT` and loads env with `load_dotenv(override=True)`. A `PORT` line in `.env.example` (copied to `.env.local`) would clobber the injected port and break the smoke test.

## Keep `/api/*` ownership in rewrites

Adding `web/app/api/**/route.ts` for agent/token logic breaks the boundary — `verify-api-contracts.ts` explicitly fails if a `route.ts` exists under `app/api`. Token logic belongs in `server/`.

## camelCase request fields

`StartAgentRequest` uses `channelName`, `rtcUid`, `userUid`, `userKey` (camelCase) to match the browser client. Renaming one side without the other breaks the contract tests.

## `MAX_TURNS` is module-level, read at import time

`memory.MAX_TURNS` is set once from `MEMORY_MAX_TURNS` when the module is imported. Changing the env var after server start has no effect without a restart.

## Local calls under a global proxy

Global proxies (Clash, etc.) can break `localhost`/RFC-1918 traffic. Configure the proxy to send `127.0.0.1`, `localhost`, and private ranges DIRECT, or `socksio` (in `requirements.txt`) plus `all_proxy` to route the backend through SOCKS.

## Related Deep Dives

- [memory_store_schema](L2/memory_store_schema.md) — `MAX_TURNS` cap and `save_memory` merge behavior.
