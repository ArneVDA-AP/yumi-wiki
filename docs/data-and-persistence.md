---
title: "Data & Persistence"
tags: [reference, sqlite, persistence]
---

# Data & Persistence

*The SQLite schema, the analytics aggregation, settings storage, and the on-disk layout.*


---

## Database

- **Engine:** SQLite via `rusqlite` with the `bundled` feature (no system SQLite needed).
- **Location:** `~/.yumi/yumi.db` (i.e. `%USERPROFILE%\.yumi\yumi.db`).
- **Pragmas:** WAL journal mode and foreign keys **on** (`db.rs` `open()`).
- **Schema creation:** tables are created if missing on startup (`Db::open()`), so there is no separate migration step.
- Defined in `src-tauri/src/db.rs`. A test-only `open_in_memory()` builds the same schema for unit tests.

### Schema

**`sessions`** — one row per chat session/tab.
| Column | Type | Notes |
|--------|------|-------|
| `id` | TEXT | PRIMARY KEY (the frontend session id) |
| `claude_session_id` | TEXT | the real `claude` uuid (learned from `system/init`) |
| `title` | TEXT | NOT NULL DEFAULT `''` |
| `cwd` | TEXT | NOT NULL DEFAULT `''` |
| `model` | TEXT | NOT NULL DEFAULT `''` |
| `created_at` | INTEGER | epoch millis, NOT NULL DEFAULT 0 |
| `updated_at` | INTEGER | epoch millis, NOT NULL DEFAULT 0 |

**`messages`** — one row per chat message.
| Column | Type | Notes |
|--------|------|-------|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT |
| `session_id` | TEXT | NOT NULL |
| `role` | TEXT | NOT NULL (`user` / `assistant`) |
| `content` | TEXT | NOT NULL — JSON-encoded `MessageBlock[]` |
| `created_at` | INTEGER | NOT NULL DEFAULT 0 |

Index: `idx_messages_session` on `messages(session_id)`.

**`analytics`** — one row per completed turn (recorded from the drained `result` line).
| Column | Type | Notes |
|--------|------|-------|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT |
| `session_id` | TEXT | |
| `model` | TEXT | NOT NULL DEFAULT `''` |
| `input_tokens` | INTEGER | NOT NULL DEFAULT 0 |
| `output_tokens` | INTEGER | NOT NULL DEFAULT 0 |
| `cost_usd` | REAL | NOT NULL DEFAULT 0 |
| `created_at` | INTEGER | epoch millis, NOT NULL DEFAULT 0 |

**`settings`** — key/value store.
| Column | Type | Notes |
|--------|------|-------|
| `key` | TEXT | PRIMARY KEY |
| `value` | TEXT | NOT NULL |

### Public functions (`db.rs`)

| Function | Purpose |
|----------|---------|
| `open()` / `open_in_memory()` | open the DB (file / in-memory test), enable WAL + FK, create schema |
| `list_sessions()` | all sessions, ordered by `updated_at DESC` |
| `get_session(id)` | one session + all its messages (`SessionDetail`) |
| `save_session(detail)` | upsert the session; replace its messages in a transaction |
| `delete_session(id)` | delete the session and its messages/analytics |
| `record_analytics(session_id, model, input, output, cost, created_at)` | insert one analytics row |
| `get_analytics()` | aggregate analytics into an `Analytics` (see below) |
| `get_setting(key)` / `set_setting(key, value)` | read/upsert a settings row |

### Analytics aggregation

`get_analytics()` returns an `Analytics { totalCostUsd, totalInputTokens,
totalOutputTokens, sessionsCount, byModel[], byDate[] }`:

- **Totals:** `SUM(cost_usd)`, `SUM(input_tokens)`, `SUM(output_tokens)`, `COUNT(DISTINCT session_id)`.
- **By model:** `GROUP BY model`, ordered by `SUM(cost_usd) DESC` → `ByModel { model, cost, tokens, count }`.
- **By date:** `GROUP BY strftime('%Y-%m-%d', created_at/1000, 'unixepoch')`, ordered `DESC` → `ByDate { date, cost, count }`.

Two unit tests guard this: `record_and_aggregate_analytics` (multi-model,
multi-day totals + ordering) and `empty_analytics_is_zeroed` (an empty DB returns
all zeros / empty arrays).

> **How analytics get recorded:** the spawner parses the turn's drained `result`
> line and calls `record_analytics` once per turn — even after the UI has
> finalized on `turn_done`. This is why the dashboard shows real totals without the
> UI waiting on the hook-delayed `result`. See [Reliability & Design](reliability-and-design.md).

---

## Settings

Settings are persisted as a single JSON blob in the `settings` table (loaded by
`get_settings`, written by `save_settings`). The current **theme** is also mirrored
to a separate settings key, and `set_theme` updates both. The `Settings` struct
uses `#[serde(rename_all = "camelCase")]`, and two fields carry serde defaults for
forward-compatibility with older blobs: `provider` (defaults to `"claude"`) and
`bash_monitor` (defaults to `false`).

| Field (TS) | Default | |
|------------|---------|--|
| `provider` | `"claude"` | provider id |
| `model` | `"claude-opus-4-8"` | |
| `theme` | `"oled"` | one of the 6 theme names |
| `vimMode` | `false` | |
| `thinking` | `true` | interleaved thinking via the proxy |
| `showRateLimit` | `true` | |
| `bashMonitor` | `false` | opt-in live bash tailing |
| `autoCompactThreshold` | `0.75` | |

## Recent projects

The recent-project list is **also** stored in the `settings` table (as JSON under
its own key), not a dedicated table. `add_recent_project(path)` dedupes by path,
refreshes the timestamp, and keeps at most **20** entries; `list_projects()`
returns them sorted by `lastOpened DESC` as `RecentProject { path, name, lastOpened }`.

---

## On-disk layout

What Yumi writes, and where:

| Path | Written by | Contents / lifetime |
|------|-----------|---------------------|
| `~/.yumi/yumi.db` | `db.rs` | sessions, messages, analytics, settings (persistent) |
| `~/.yumi/bg-processes.json` | MCP server | background-process registry |
| `~/.yumi/bg-output/<id>.txt` | MCP server | per-background-process output |
| `~/.yumi/askuser/<sid>.json` / `.answer.json` | MCP server | AskUserQuestion file-IPC |
| `%TEMP%\yumi-bash-<sid>.log` | MCP server | live bash stream tail; cleaned on exit and on next startup (> 1 day old) |
| `%TEMP%\yumi-mcp-<sid>.json` | spawner | temp MCP config for a bash-monitor turn; deleted on turn exit |
| `%TEMP%\yumi-agents\<id>\` | `agents.rs` | per-agent git worktree (branch `yumi/agent-<id>`); retired on merge/remove |

`crash_recovery.rs` sweeps stale `yumi-bash-*.log` files (older than a day) at
startup. The bundled `~/.claude` plugin is covered on [Yumi Plugin](yumi-plugin.md).

## See also

- [MCP Bash Server](mcp-bash-server.md) — the sidecar that writes the `~/.yumi` files above.
- [IPC Contract](ipc-contract.md) — `SessionMeta` / `SessionDetail` / `Analytics` shapes.
- [Backend (Rust)](backend-rust.md) — `db.rs` and the DB-backed commands.
