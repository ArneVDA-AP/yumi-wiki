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
  state.rs             shared AppState
  crash_recovery.rs    startup cleanup of stale stream logs
  agents.rs            background agents (git-worktree isolation)
  db.rs                SQLite schema + queries (see Data & Persistence)
  process/
    registry.rs        PID registry + Windows tree-kill
  claude/
    binary.rs          locate the claude binary + version
    spawner.rs         spawn claude, stream stdout → events
    session.rs         session-id helpers/validation
    stream_parser.rs   stream-json → ClaudeEvent (pure)
    proxy.rs           thinking-proxy launcher
    provider.rs        multi-provider abstraction
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
2. Opens the database (`Db::open()` → `~/.yumi/yumi.db`, creates tables if missing).
3. Creates `AppState { registry, proxy, db, agents }` as Tauri-managed state.
4. Registers the `tauri-plugin-opener` plugin.
5. Registers **all** commands via `generate_handler![]` (see the table below).

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
- `locate()` finds the provider's binary via the same PATH/`~/.local/bin` search.
- `build_args(prompt, model, resume)` builds the CLI arguments:
  - **Claude (verified):** `--output-format stream-json --verbose --include-partial-messages --dangerously-skip-permissions [--resume <uuid>] -p <prompt> --model <model>`.
  - **Gemini / Codex / Kiro (best-effort, unverified):** a minimal `-m <model> -p/<prompt>` shape mirroring Yume's shim. Only Claude is wired and tested.

### `spawner.rs` — `spawn_claude(app, opts) -> Result<String,String>`
The heart of the backend. `SpawnOpts` carries `session_id, cwd, prompt, model,
provider?, resume_id?, thinking?, bash_monitor?`.

Flow:
1. **Resolve provider** (`Provider::from_id`). `--resume <uuid>` added when `resume_id` is present (Claude only).
2. **Bash monitor (opt-in, Claude only).** Writes a temp MCP config at
   `%TEMP%\yumi-mcp-<session_id>.json` registering `resources/yumi-mcp-bash.cjs`
   (with `--session-id=` and `--stream-file=`), and passes `--mcp-config <path>
   --disallowedTools Bash` so the model routes shell through
   `mcp__yumi-bash__RunBash`.
3. **Thinking proxy (opt-in, Claude only).** `state.proxy.ensure_running()` returns
   an `ANTHROPIC_BASE_URL` only if the proxy is confirmed listening.
4. **Env sanitization.** Strips `CLAUDE_CODE_*`, `CLAUDECODE`, `ANTHROPIC_BASE_URL`,
   `CLAUDE_STREAM*` from the child so a parent Claude Code session can't leak in.
5. **Spawn** with `stdin: null`, `stdout/stderr: piped`, sets
   `CLAUDE_CODE_ENTRYPOINT=yumi`, `YUMI_STREAM_FILE=%TEMP%\yumi-bash-<sid>.log`,
   `YUMI_SESSION_ID=<sid>`. Registers the child PID in the registry.
6. **Stream loop.** Reads stdout line by line → `parse_line` → emits events.
   Finalizes on `turn_done`/`result`; afterwards keeps draining stdout silently
   (no more emits) so the lingering process's pipe never blocks.
7. **Analytics + late cost.** When the `result` line is drained, records an
   analytics row (`db.record_analytics`) and — if the turn was already finalized —
   emits a `claude-cost` event, **pid-gated** by `registry.is_current` so it can't
   attach to a newer turn.
8. **Exit.** `child.wait()`, abort the monitor task, delete the temp MCP config,
   `registry.unregister_if(sid, pid)` (reuse-safe).

Events emitted (payloads use camelCase):

| Event | Payload | When |
|-------|---------|------|
| `claude-session-id` | `{sessionId, claudeSessionId}` | on `system/init` (real uuid learned) |
| `claude-event` | `{sessionId, event}` | every parsed `ClaudeEvent` |
| `claude-complete` | `{sessionId, code}` | turn finalized / process exit |
| `claude-cost` | `{sessionId, cost, durationMs}` | late `result` after finalize |
| `claude-error` | `{sessionId, message}` | spawn/stream failure (pre-finalize) |

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
that app session. The script is located by walking up from the executable looking
for `resources/thinking-proxy.cjs`. See [Thinking Proxy](thinking-proxy.md).

---

## Process control (`process/registry.rs`)

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

> Note: `get_usage_limits` currently returns a **stub** (`UsageLimits` with all
> fields empty) — the rate-limit pills render from it but show no live percentages
> until it is wired to a real source. This is the honest status; see `PARITY.md`.

## See also

- [IPC Contract](ipc-contract.md) — exact command signatures, events, and types.
- [Stream Pipeline](stream-pipeline.md) — the `stream_parser` mapping in detail.
- [Data & Persistence](data-and-persistence.md) — the `db.rs` schema and queries.
- [Reliability & Design](reliability-and-design.md) — env sanitization, proxy probing, finalize-on-`end_turn`.
