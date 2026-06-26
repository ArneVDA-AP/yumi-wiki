---
title: "Backend (Rust)"
tags: [reference, rust, tauri, backend]
---

# Backend (Rust)

*A module-by-module tour of `src-tauri/src` — the Tauri host that spawns `claude`, parses its stream, persists sessions, and runs background agents.*


---

## Layout

```
src-tauri/src/
  main.rs              entry point
  lib.rs               Tauri builder + command registry
  platform.rs          CREATE_NO_WINDOW + now_millis (shared constants)
  state.rs             shared AppState
  crash_recovery.rs    startup cleanup of stale stream logs
  agents.rs            background agents (git-worktree isolation)
  db.rs                SQLite schema + queries (see Data & Persistence)
  process/
    registry.rs        PID registry + Windows tree-kill
    guard.rs           process-wide PID set + panic/exit kill_all
  claude/
    binary.rs          locate the claude binary + version
    spawner.rs         spawn claude, stream stdout → events
    session.rs         session-id helpers/validation
    stream_parser.rs   stream-json → ClaudeEvent (pure)
    proxy.rs           thinking-proxy launcher
    provider.rs        multi-provider abstraction (build_spawn_plan)
    resources.rs       resolve bundled sidecar paths (resource_dir-first)
  mcp/
    monitor.rs         BASH-MONITOR: tail temp log → bash-output
  commands/
    claude.rs info.rs db.rs git.rs fs.rs settings.rs
```

A note on the build: spawner/provider/agents are gated `#[cfg(not(test))]` so that
`cargo test` (which can't initialise the WebView2 loader on `windows-gnu`) still
compiles and runs the pure unit tests. The crate is `[lib] crate-type = ["rlib"]`
— see [Build, Run & Test](build-run-test.md).

---

## App bootstrap

### `main.rs`
Entry point; calls `yumi_lib::run()`. Disables the console window on Windows release builds.

### `lib.rs` — `pub fn run()`
Builds the Tauri application:
1. `crash_recovery::cleanup_stale_streams()` removes stale `yumi-bash-*.log` files (> 1 day old) before anything else.
2. `process::guard::install_panic_hook()` wraps the default panic handler so all tracked child PIDs are tree-killed before the process unwinds — ensuring a host panic never orphans a spawned `claude` or thinking-proxy.
3. Opens the database (`Db::open()` → `~/.yumi/yumi.db`, creates tables if missing) and immediately calls `db.prune(500, 365_days_ms, now_ms)` to cap growth: keeps the newest 500 sessions and ages out analytics rows older than one year.
4. Creates `AppState { registry, proxy, db, agents }` as Tauri-managed state.
5. Registers the `tauri-plugin-opener` plugin.
6. Registers **all** commands via `generate_handler![]` (see the table below).
7. On `RunEvent::Exit` calls `process::guard::kill_all()` so any still-running children are cleaned up on normal window close.

### `state.rs` — `AppState`
The shared, Tauri-managed state struct:

| Field | Type | Purpose |
|-------|------|---------|
| `registry` | `ProcessRegistry` | sessionId → running claude PID |
| `proxy` | `ProxyState` | thinking-proxy launcher (lazy, cached) |
| `db` | `Db` | database handle |
| `agents` | `AgentRegistry` (`#[cfg(not(test))]`) | background-agent registry |

`AppState::new(db)` constructs the registry/proxy/agents fresh.

### `crash_recovery.rs`
Minimal, best-effort startup cleanup. Scans the temp dir for files named
`yumi-bash-*.log` and deletes any older than `86_400` seconds (1 day). All errors
are silently ignored — it must never block startup.

---

## The Claude pipeline (`claude/`)

### `binary.rs` — locating the CLI
- `locate_claude() -> Option<PathBuf>` resolves the `claude` binary.
- `locate_on_path_or_local(names)` probes each candidate **on `PATH` first**, then in `~/.local/bin`.
- Windows candidate names: `claude.exe`, `claude.cmd`, `claude.bat`, `claude`; Unix: `claude`.
- `get_claude_version() -> Result<String,String>` runs `claude --version` (through `cmd /C` for `.cmd`/`.bat` shims) and trims the output, falling back to stderr for shims that print there.

### `provider.rs` — multi-provider abstraction
`enum Provider { Claude, Gemini, Codex, Kiro }`.

- `from_id(Option<&str>)` maps `"claude"`→Claude, `"gemini"`→Gemini, `"codex"|"gpt"`→Codex, `"kiro"`→Kiro; anything else defaults to **Claude**.
- `build_spawn_plan(provider, prompt, model, resume, router_base_url) -> Result<SpawnPlan, String>` builds the args and resolves the routing strategy. Every provider drives the **same** `claude` binary with the same `stream-json` flags (`--output-format stream-json --verbose --include-partial-messages --dangerously-skip-permissions [-p <prompt>] [--resume <uuid>] [--model <model>]`). `--resume` is included for **all** providers when a resume id is present — the transcript replay works against whatever endpoint `ANTHROPIC_BASE_URL` points at. `SpawnPlan { args, base_url }` carries the args plus an optional `ANTHROPIC_BASE_URL` override: `None` for Claude (real Anthropic API), `Some(router_url)` for a routed provider. A non-Claude provider with an empty `router_base_url` returns a clear `Err` — never a fake success.

There is no `locate()`, `build_args()`, or per-provider binary lookup in v1. Spawning foreign CLIs (`gemini`, `codex`) was explicitly removed: those CLIs don't emit `stream-json` and could never render.

### `spawner.rs` — `spawn_claude(app, opts) -> Result<String,String>`
The heart of the backend. `SpawnOpts` carries `session_id, cwd, prompt, model,
provider?, router_base_url?, resume_id?, thinking?, bash_monitor?`.

Flow:
1. **Resolve provider + plan** (`Provider::from_id` → `build_spawn_plan`). `--resume <uuid>` is added for **all** providers when `resume_id` is present — the transcript replay works against whatever endpoint `ANTHROPIC_BASE_URL` points at. A non-Claude provider with no `router_base_url` returns a clear `Err` here.
2. **Bash monitor (opt-in).** `write_bash_mcp_config` locates `yumi-mcp-bash.cjs` via `resolve_resource`, writes a temp MCP config at `%TEMP%\yumi-mcp-<session_id>.json`, and passes `--mcp-config <path> --disallowedTools Bash` so the model routes shell through `mcp__yumi-bash__RunBash`. The bash monitor IS wired and functional.
3. **Base-URL resolution.** A routed provider's `plan.base_url` takes priority (set by `build_spawn_plan`). For Claude with thinking enabled, `resolve_resource` locates `thinking-proxy.cjs` and passes the resolved `PathBuf` to `state.proxy.ensure_running(script)`, which returns an `ANTHROPIC_BASE_URL` only if the proxy is confirmed listening.
4. **Env sanitization.** Strips `CLAUDE_CODE_*`, `CLAUDECODE`, `ANTHROPIC_BASE_URL`, `CLAUDE_STREAM*` from the child so a parent Claude Code session can't leak in.
5. **Spawn** with `stdin: null`, `stdout/stderr: piped`, sets `CLAUDE_CODE_ENTRYPOINT=yumi`, `YUMI_STREAM_FILE`, `YUMI_SESSION_ID`. Registers the child PID in the session registry **and** calls `process::guard::track(pid)` so the child is killed on a host panic (not only on a clean registry interrupt).
6. **Stream loop.** Reads stdout line by line → `parse_line` → emits events. Finalizes on `turn_done`/`result`; afterwards keeps draining stdout silently so the lingering process's pipe never blocks.
7. **Analytics + late cost.** When the `result` line is drained, records an analytics row (`db.record_analytics`) and — if the turn was already finalized — emits a `claude-cost` event, **pid-gated** by `registry.is_current` so it can't attach to a newer turn.
8. **Exit.** `child.wait()`, abort the monitor task, delete the temp MCP config, `registry.unregister_if(sid, pid)` (reuse-safe), `guard::untrack(pid)`.

Events emitted (payloads use camelCase):

| Event | Payload | When |
|-------|---------|------|
| `claude-session-id` | `{sessionId, claudeSessionId}` | on `system/init` (real uuid learned) |
| `claude-event` | `{sessionId, event}` | every parsed `ClaudeEvent` |
| `claude-complete` | `{sessionId, code}` | turn finalized / process exit |
| `claude-cost` | `{sessionId, cost, durationMs}` | late `result` after finalize |
| `claude-error` | `{sessionId, message}` | spawn/stream failure (pre-finalize) |

### `resources.rs` — sidecar resolution
`resolve_resource(app, name) -> Option<PathBuf>` locates a bundled sidecar in two steps: (1) `app.path().resource_dir()/resources/<name>` — the layout a real installed bundle ships; (2) ancestor-walk from the executable for a sibling `resources/<name>` — the dev `--no-bundle` layout. The resource-dir check comes **first** so a bundled install never falls back to the dev walk. The pure `ancestor_resource(start, name)` helper is unit-tested in `cargo test`.

### `session.rs` — session-id helpers
- `resume_arg(Option<&str>)` trims and drops empty resume ids.
- `is_valid_session_id(&str)` enforces `[A-Za-z0-9_-]{1,128}` — important because the session id becomes part of the bash stream-file name.

### `stream_parser.rs` — `parse_line(line) -> Vec<ClaudeEvent>`
The pure, non-lossy mapping from one stdout JSON line to typed events. It is the
single most important file for understanding the data model; the full mapping has
its own page: **[Stream Pipeline](stream-pipeline.md)**. The `ClaudeEvent` enum
serializes with `#[serde(tag = "kind", rename_all = "snake_case")]`, so e.g.
`TurnDone` → `{"kind":"turn_done", ...}` with camelCase fields. Covered by the
parser unit tests in `cargo test`.

### `proxy.rs` — `ProxyState`
Lazily launches `resources/thinking-proxy.cjs` via `node` on a free port in
**18800–18899**, then **TCP-probes that it is actually listening** (up to 3 s)
before returning `http://127.0.0.1:<port>`. If node is missing or the probe times
out it returns `None` (→ claude talks to the real API directly) and never retries
that app session.

`ensure_running(script: PathBuf)` takes the already-resolved path to `thinking-proxy.cjs` — the caller (`spawner.rs`) resolves it via `resolve_resource` first (resource-dir-first, ancestor-walk fallback). The proxy PID is tracked via `process::guard::track(pid)` so it is killed with the host on panic or normal exit, not just on a clean tree-kill. See [Thinking Proxy](thinking-proxy.md).

---

## Process control (`process/`)

### `process/registry.rs`

`ProcessRegistry` is a `Mutex<HashMap<String, u32>>` mapping sessionId → PID.

| Method | Purpose |
|--------|---------|
| `register(sid, pid)` | record the running PID |
| `unregister(sid)` | remove and return the PID |
| `unregister_if(sid, pid)` | remove **only if** the stored PID matches — prevents a lingering turn from unregistering a newer turn that reused the sessionId |
| `pid_for(sid)` | look up the PID |
| `is_current(sid, pid)` | is this PID still the registered one? (gates late events) |
| `interrupt(sid)` | tree-kill the process and unregister |
| `kill_process_tree(pid)` | platform tree-kill |

On **Windows**, `kill_process_tree` enumerates children via `wmic process where
(ParentProcessId=<pid>) get ProcessId /format:list`, runs `taskkill /PID <pid> /T
/F`, and also explicitly kills enumerated children as a belt-and-suspenders for
grandchildren. On other platforms it falls back to `kill -TERM`. The reuse-safe
`unregister_if`/`is_current` pair is what lets a late, hook-delayed `result` be
processed without corrupting a newer turn.

### `process/guard.rs`

A process-wide guard that ensures spawned children (both `claude` sessions and the thinking proxy) die with the host — not only on a clean registry interrupt.

- `track(pid)` / `untrack(pid)` — insert/remove a PID from a global `Mutex<HashSet<u32>>`.
- `kill_all()` — drain the set and tree-kill every PID (best-effort).
- `install_panic_hook()` — wraps the existing panic handler with a `kill_all()` call, installed once at startup via `OnceLock`. This is what prevents a host panic from orphaning spawned `claude` processes.

`spawner.rs` calls `guard::track(pid)` immediately after spawning and `guard::untrack(pid)` on clean process exit. `proxy.rs` calls `guard::track(pid)` after the proxy is confirmed listening. `lib.rs` calls `guard::kill_all()` in the `RunEvent::Exit` handler for normal window close.

---

## Shared utilities (`platform.rs`)

`platform.rs` holds cross-cutting constants and helpers used across multiple modules:

- `CREATE_NO_WINDOW: u32` (`#[cfg(windows)]`) — the Win32 creation flag that hides the console window for all spawned child processes. Declared once here and imported by the seven modules that spawn children: `claude/{binary,proxy,spawner}.rs`, `agents.rs`, `process/registry.rs`, and `commands/{fs,git}.rs`.
- `now_millis() -> i64` — wall-clock epoch milliseconds (used by analytics recording and `db.prune`).

---

## Background agents (`agents.rs`)

Runs a headless `claude -p` turn in an **isolated git worktree** so it can't
disturb the working tree, then commits the result for review/merge.

`AgentInfo` fields: `id, title, prompt, cwd, branch, worktree, model, status,
detail?, created_at`. `status` ∈ `running | done | failed | merged | cancelled`.

Commands (all `#[tauri::command]`):

| Command | Behaviour |
|---------|-----------|
| `list_agents()` | sorted snapshot of all agents |
| `start_agent(cwd, prompt, model)` | create worktree `%TEMP%\yumi-agents\<id>` on branch `yumi/agent-<id>`, spawn a headless `claude -p` (env-sanitized like the spawner), drain its stdout, then `git add -A` and commit `yumi agent: <prompt>` if dirty |
| `merge_agent_branch(id)` | merge the agent branch back into the working tree and retire the worktree |
| `cancel_agent(id)` | tree-kill the running claude (`kill_process_tree`) and mark cancelled |
| `remove_agent(id)` | remove the worktree, delete the branch, unregister |
| `agent_diff(id)` | diff between the agent branch and its base commit |

State changes are pushed to the UI via the `agent-event` Tauri event
(`{agent: AgentInfo}`). Agents require a **git repo** as the cwd and degrade with a
clear status otherwise.

---

## BASH-MONITOR (`mcp/monitor.rs`)

When the bash monitor is enabled, `spawn_monitor(app, session_id)` starts a tokio
task that **tails** `%TEMP%\yumi-bash-<session_id>.log` (written by the MCP server),
polling every ~150 ms with an offset cursor, and emits each new chunk as a
`bash-output` event `{sessionId, chunk}`. The store appends the chunk to the last
running tool card's `live` field. See [MCP Bash Server](mcp-bash-server.md).

---

## Command surface

`lib.rs` registers these `#[tauri::command]`s (the exhaustive list — full
signatures and the matching `ipc.ts` wrappers are on the [IPC Contract](ipc-contract.md) page):

- **Claude:** `get_claude_version`, `spawn_claude`, `interrupt_claude`, `get_usage_limits`
- **Settings:** `get_settings`, `save_settings`, `set_theme`
- **DB/sessions:** `db_list_sessions`, `db_get_session`, `db_save_session`, `db_delete_session`, `db_get_analytics`
- **Projects/FS:** `list_projects`, `add_recent_project`, `pick_directory`, `list_directory`, `read_text_file`
- **Git:** `git_status`, `git_diff`
- **Agents:** `list_agents`, `start_agent`, `merge_agent_branch`, `cancel_agent`, `remove_agent`, `agent_diff`

> `get_usage_limits` returns an **honestly-inert** `UsageLimits { rateLimited: false, live: false }` — there is no reliable local rolling-window source, so the command signals "no data" rather than guessing. The rate-limit pills grey out on startup and light up live when an in-stream `rate_limit_event` arrives (forwarded by the spawner even after the turn finalizes, pid-gated so it can't attach to a newer turn).

## See also

- [IPC Contract](ipc-contract.md) — exact command signatures, events, and types.
- [Stream Pipeline](stream-pipeline.md) — the `stream_parser` mapping in detail.
- [Data & Persistence](data-and-persistence.md) — the `db.rs` schema and queries.
- [Reliability & Design](reliability-and-design.md) — env sanitization, proxy probing, finalize-on-`end_turn`.
