# Deep Dive — Memory Store Schema

> **When to Read This:** You are changing the SQLite schema, `MAX_TURNS` behavior, the format of stored turns, the `build_memory_system_message` output, or the `memory.py` test coverage. For the high-level picture, start at [02_architecture](../02_architecture.md).

The memory store is a single SQLite table managed by `server/src/memory.py`. The module has **no `agora_agent` import** so it can be tested standalone.

## SQLite schema

```sql
CREATE TABLE IF NOT EXISTS memory (
    user_key TEXT PRIMARY KEY,
    turns    TEXT NOT NULL,       -- JSON array of {role, content} objects
    updated_at REAL NOT NULL      -- Unix timestamp (time.time())
)
```

One row per user handle (`user_key`). `turns` is a JSON-serialised list; `updated_at` is set on every `save_memory` call.

## `memory.py` API

| Function | Signature | Purpose |
| -------- | --------- | ------- |
| `get_db` | `get_db(path=DB_PATH) → sqlite3.Connection` | Open (or create) the DB file and ensure the schema exists. Returns a live connection. Caller is responsible for `conn.close()`. |
| `get_memory` | `get_memory(conn, user_key) → list` | Return the stored list of `{role, content}` dicts for a handle, or `[]` if none. |
| `save_memory` | `save_memory(conn, user_key, new_turns) → None` | Merge new turns with existing turns, cap to `MAX_TURNS`, persist via `INSERT ... ON CONFLICT ... DO UPDATE`. |
| `build_memory_system_message` | `build_memory_system_message(turns) → dict \| None` | Return a `{"role":"system","content":"..."}` dict from turns, or `None` when turns is empty. |

## `MAX_TURNS` and merge behavior

`MAX_TURNS = int(os.getenv("MEMORY_MAX_TURNS", "20"))` — evaluated once at module import. The effective cap is set at startup; changing the env var requires a server restart.

`save_memory` merges as:

```python
merged = (get_memory(conn, user_key) + new_turns)[-MAX_TURNS:]
```

Turns are appended to the existing history, then the tail of `MAX_TURNS` entries is retained. Empty-content turns are filtered before merging.

## `build_memory_system_message` output

When turns are present:

```python
{
    "role": "system",
    "content": (
        "You are talking to a RETURNING user. Here is what you remember from past "
        "conversations with them — use it naturally; do not read it back verbatim:\n"
        + "\n".join(f"- {t['role']}: {t['content']}" for t in turns)
    )
}
```

Returns `None` for empty turns; the caller (`Agent.start()`) skips appending it in that case.

## `get_history()` capture window

`session.get_history()` returns an object with a `.contents` attribute (a list of message-like objects with `.role` and `.content`). This API is only available **while the session is running**.

`Agent.stop()` extraction:

```python
history = await session.get_history()
contents = getattr(history, "contents", None) or []
turns = [
    {"role": getattr(c, "role", "user"), "content": getattr(c, "content", "")}
    for c in contents
]
```

After calling `session.stop()`, `get_history()` is no longer available. The sequence in `stop()` (pop session → get_history → save_memory → session.stop) must not be reordered.

## Test coverage (`server/tests/test_memory.py`)

| Test | What it checks |
| ---- | -------------- |
| `test_empty_returns_no_turns_and_no_system_message` | Fresh DB returns `[]`; `build_memory_system_message([])` returns `None`. |
| `test_save_then_recall_roundtrip_across_connections` | Saved turns are readable via a new connection to the same path. |
| `test_memory_is_keyed_per_user` | Alice's turns don't bleed into Bob's. |
| `test_save_accumulates_and_caps_to_max` | Storing `MAX_TURNS + 10` entries leaves exactly `MAX_TURNS`; last entry is the most recent. |
| `test_build_system_message_contains_remembered_content` | System message contains the saved content text. |

All 5 tests use `tempfile.mkdtemp()` — no shared state, no cleanup needed.

## Related L1

- [02_architecture](../02_architecture.md) · [04_conventions](../04_conventions.md) · [07_gotchas](../07_gotchas.md)
