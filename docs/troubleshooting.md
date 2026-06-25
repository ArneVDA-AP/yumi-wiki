---
title: "Troubleshooting"
tags: [guide, troubleshooting]
---

# Troubleshooting

*Common failure modes and how to resolve them, with pointers to the responsible code.*


---

## Spawn / chat

**"claude binary not found."**
Yumi searches `PATH` then `%USERPROFILE%\.local\bin` for `claude.exe` / `.cmd` /
`.bat` / `claude` (`claude/binary.rs`). Ensure `claude` is installed and on `PATH`,
or present in `~/.local/bin`.

**The chat hangs for minutes / zombie `claude` processes appear.**
This was the env-leak bug. It is fixed by env sanitization in `spawner.rs`, but it
specifically occurred when launching Yumi **from inside** another Claude Code
session. If you see it, confirm you're on a build that strips `CLAUDE_CODE_*` /
`CLAUDECODE` / `ANTHROPIC_BASE_URL` / `CLAUDE_STREAM*`. See [Reliability & Design §1](reliability-and-design.md).

**No thinking blocks appear.**
Thinking requires the proxy (`thinking` setting, default on) **and** `node` on
`PATH`. If `node` is missing or the proxy doesn't come up within ~3 s, Yumi falls
back to the real API with no proxy — the chat still works, you just don't get
injected interleaved thinking. See [Thinking Proxy](thinking-proxy.md).

**The Stop button used to stay "Stop" long after the answer finished.**
Resolved: the turn now finalizes on `end_turn` (~10 s) rather than the
hook-delayed `result` (~47 s). If it regresses, check the `turn_done` path in
`stream_parser.rs`/`streamParse.ts`. See [Reliability & Design §3](reliability-and-design.md).

**Interrupt doesn't fully kill the process.**
On Windows the registry uses `taskkill /T /F` plus explicit child enumeration via
`wmic` for grandchildren (`process/registry.rs`). If a stray child survives, that's
where to look.

## Live bash output

**The tool card shows no live output.**
The bash monitor is **opt-in** and **off by default**. Turn on the bash-monitor
setting; that makes the spawner register `yumi-mcp-bash.cjs` and pass
`--disallowedTools Bash`, so shell routes through `mcp__yumi-bash__RunBash` and the
Rust tail can stream it. Without it, `claude` uses its built-in `Bash` tool and
nothing is written to the stream log. See [MCP Bash Server](mcp-bash-server.md).

**Output appears truncated.**
`RunBash` caps output at 2 MB and appends `[output truncated at 2MB]`. Long-running
commands also hit the 15-minute default timeout (20-minute cap).

## Analytics / cost

**The analytics dashboard shows zeros.**
Analytics are recorded from each turn's drained `result` line via
`record_analytics`. A brand-new DB has no rows; complete at least one turn. If it
stays zero after real turns, check the spawner's drain loop. See
[Data & Persistence](data-and-persistence.md).

**The per-message cost footer disappears after reload.**
Expected: cost is shown live via the `claude-cost` side channel but **not**
persisted (the `messages` schema has no cost column). The analytics dashboard,
which *is* persisted, still reflects it.

## Rate limits

**The 5h / 7d pills show nothing.**
The `get_usage_limits` command is currently a **stub** returning empty values, so
the pills render but have no live percentages. The in-stream `rate_limit_event` is
parsed, but the polled limits aren't wired yet. This is the honest status in
`PARITY.md`.

## Background agents

**"start_agent" fails or does nothing.**
Background agents require the tab's **cwd to be a git repo** — they create a
worktree under `%TEMP%\yumi-agents\` on a `yumi/agent-<id>` branch. In a non-git
directory the agent degrades with a clear status. See [`agents.rs`](backend-rust.md).

**A merge leaves a stray worktree/branch.**
`merge_agent_branch` retires the worktree on success and `remove_agent` cleans it
up; if one is orphaned, remove the worktree dir under `%TEMP%\yumi-agents\` and
delete the `yumi/agent-<id>` branch manually.

## Build

**Build fails asking for MSVC / `link.exe`.**
Yumi targets `windows-gnu`. Confirm `[lib] crate-type = ["rlib"]` in
`src-tauri/Cargo.toml` (not `cdylib`) and that you're on the gnu toolchain. See
[Build, Run & Test](build-run-test.md).

**Vite dev server won't start.**
Port `1420` is **strict** — it fails rather than picking another port. Free `1420`
(and `1421` for HMR) or stop the conflicting process.

**`cargo test` fails to link / WebView2 errors.**
The spawner/provider/agents modules are `#[cfg(not(test))]` precisely to keep
`cargo test` building on `windows-gnu` without the WebView2 loader. If you add a
backend module that pulls in Tauri runtime types, gate it the same way.

## See also

- [Reliability & Design](reliability-and-design.md) — the root causes behind several of these.
- [Build, Run & Test](build-run-test.md) — toolchain and commands.
- `PROGRESS.md` (repo root) — the live status anchor.
