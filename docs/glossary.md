---
title: "Glossary"
tags: [reference, glossary]
---

# Glossary

*Terms used throughout the Yumi wiki.*


---

**Adaptive thinking** — the thinking mode injected by the proxy for recent models
(`{type:"adaptive", display:"summarized"}`), versus the older `{type:"enabled",
budget_tokens}` form. See [Thinking Proxy](thinking-proxy.md).

**AppState** — the Tauri-managed shared state (`state.rs`): process registry, proxy
launcher, DB handle, agent registry.

**BASH-MONITOR** — the mechanism that streams live shell output into the tool card:
the MCP server writes a temp log, and `mcp/monitor.rs` tails it and emits
`bash-output`. See [MCP Bash Server](mcp-bash-server.md).

**Background agent** — a headless `claude -p` turn run in an isolated git worktree
(`agents.rs`), reviewable and mergeable. Not the same as a plugin subagent.

**`ClaudeEvent`** — the discriminated union (tagged by `kind`) that the Rust parser
emits and the React reducer consumes. The data model's backbone. See
[IPC Contract](ipc-contract.md).

**`claude-cost`** — a side-channel Tauri event carrying a turn's cost + duration,
emitted when the late `result` is drained after finalize. Pid-gated.

**`contextWindowFor`** — a TypeScript function in `src/lib/models.ts` that maps a
model id string to its context-window size in tokens (e.g. 200 K for Claude, 1 M
for Gemini/1M variants, 200 K default). Used by `TokenBar.tsx` to size the bar
dynamically instead of the old hardcoded 200 K.

**Finalize** — the moment a turn flips Stop → Send and re-enables input. Triggered
by `turn_done` (`end_turn`), not the late `result`. See [Reliability & Design](reliability-and-design.md).

**`include-partial-messages`** — the `claude` flag that makes the CLI emit
`stream_event` lines (the `text_delta` / `thinking_delta` tokens that drive live
typing).

**Interleaved thinking** — thinking tokens streamed alongside the answer; enabled
via the thinking proxy.

**MCP** — Model Context Protocol. Yumi reimplements an MCP **stdio server**
(`yumi-mcp-bash.cjs`) exposing `RunBash` and friends. See [MCP Bash Server](mcp-bash-server.md).

**Pid-gating / reuse-safety** — using `is_current` / `unregister_if` so a lingering
turn's late output can't corrupt a newer turn that reused the same session id. See
[Backend (Rust)](backend-rust.md).

**`process::guard`** — a process-wide child-PID set (`src-tauri/src/process/guard.rs`).
Every spawned child (claude turns + thinking-proxy) is `guard::track`ed; a
`std::panic` hook installed at startup and the Tauri `RunEvent::Exit` handler both
call `guard::kill_all`, tree-killing all tracked children so none leak on host
panic or clean exit.

**Provider** — an LLM routing abstraction (`claude/provider.rs`). Every provider
drives the same `claude` binary; a non-Claude provider (Gemini/Codex/Kiro) is
realized by pointing that `claude` process at a translating router via
`ANTHROPIC_BASE_URL` (`settings.routerBaseUrl`). No router configured → a clear
`Err`, never a fake session.

**`rate_limit_event`** — a `stream-json` line the CLI emits after `end_turn`,
carrying `{rate_limit_info:{status, resetsAt, rateLimitType}}`. The spawner
forwards it to the frontend even after the turn has finalized (pid-gated), so the
rate-limit pills can update live. See [IPC Contract](ipc-contract.md).

**`raw` event** — the non-lossy fallback `ClaudeEvent` for any stream line the
parser doesn't otherwise recognise.

**`resolve_resource`** — the sidecar-resolution function in
`src-tauri/src/claude/resources.rs`. Tries `app.path().resource_dir()/resources/<name>`
first (the bundled install layout), then falls back to a dev ancestor-walk from the
executable's directory. Called by the spawner and proxy so sidecars are found in
both bundled and `--no-bundle` dev builds.

**Session id** — Yumi's own per-tab id (validated `[A-Za-z0-9_-]{1,128}`), distinct
from the **claude session id** (the real uuid the CLI reports via `system/init`,
used for `--resume`).

**`splitTabs`** — the store field holding the ordered tab ids shown in the split
grid (F7/F8); ≤1 means the normal single view.

**`stream-json`** — `claude`'s `--output-format stream-json`: one JSON object per
stdout line. The input to the [Stream Pipeline](stream-pipeline.md).

**Tauri command** — a Rust `#[tauri::command]` callable from the UI via
`invoke()`/`ipc.ts`. See [IPC Contract](ipc-contract.md).

**Thinking proxy** — the localhost HTTP proxy (`thinking-proxy.cjs`) that injects
thinking config into upstream requests; pointed at via `ANTHROPIC_BASE_URL`.

**`turn_done`** — the event derived from a terminal `message_delta` `stop_reason`;
the signal Yumi finalizes on.

**WAL** — SQLite's write-ahead logging journal mode, enabled on `~/.yumi/yumi.db`.

**Worktree isolation** — running a background agent in a throwaway git worktree
under `%TEMP%\yumi-agents\` so it can't disturb the working tree.

**Yume** — the commercial app Yumi is a personal, ungated clone of. See [Overview](overview.md).

---
