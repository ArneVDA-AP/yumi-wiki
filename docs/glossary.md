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

**Provider** — an LLM CLI backend (`provider.rs`): Claude (verified) or
Gemini/Codex/Kiro (best-effort shims).

**`raw` event** — the non-lossy fallback `ClaudeEvent` for any stream line the
parser doesn't otherwise recognise.

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
