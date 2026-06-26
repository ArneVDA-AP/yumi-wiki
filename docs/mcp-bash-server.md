---
title: "MCP Bash Server"
tags: [subsystem, mcp, bash]
---

# MCP Bash Server

*`resources/yumi-mcp-bash.cjs` — the reimplemented MCP stdio server that gives Yumi a persistent shell, an AskUserQuestion popover, and the live bash tail in the tool card.*


---

## What it is and when it runs

A Node.js MCP (Model Context Protocol) server that speaks line-delimited JSON-RPC
2.0 over stdio. It is **opt-in**: it only participates when the **bash monitor**
setting is on. When enabled, the spawner (`claude/spawner.rs`) writes a temp MCP
config (`%TEMP%\yumi-mcp-<sid>.json`) that registers this script and passes
`--mcp-config <path> --disallowedTools Bash` to `claude`, so the model's shell runs
through `mcp__yumi-bash__RunBash` instead of the built-in `Bash` tool. That
redirection is what lets the Rust BASH-MONITOR tail and stream the output live.

The script path is resolved by `claude::resources::resolve_resource`, which tries
`app.path().resource_dir()/resources/yumi-mcp-bash.cjs` first (installed and
`target/debug/resources` layouts) and falls back to walking up from the executable
— so it works in a bundled install and in the `--no-bundle` dev build.

- **Protocol version:** `2024-11-05`.
- **Server info:** `{ name: "yumi", version: "1.0.0" }`.
- Session id comes from the `--session-id=` arg or `YUMI_SESSION_ID` env, validated against `^[A-Za-z0-9_-]{1,128}$`.

## Tools (exactly 5)

| Tool | Input | Notes |
|------|-------|-------|
| **RunBash** | `command` (required), `restart?`, `timeout?` (ms), `description?`, `run_in_background?` | run a shell command in the persistent session |
| **ListBackgroundProcesses** | — | list running background processes |
| **CancelBackgroundProcess** | `id` (e.g. `bg-1234567890`) | cancel one background process |
| **CancelAllBackgroundProcesses** | — | cancel them all |
| **AskUserQuestion** | `questions` (1–4 objects; each: `question`, `header` ≤12 chars, `multiSelect`, `options` 2–4 × `{label, description}`) | surface a choice popover in the UI |

`RunBash` defaults: **timeout 900 000 ms (15 min)**, capped at **1 200 000 ms (20
min)**. Output is capped at **2 MB**, after which `[output truncated at 2MB]` is
appended. The MCP test harness asserts exactly these 5 tool names.

## Persistent shell behaviour

- **Working directory** persists across calls: the server tracks `cd` commands (via a regex on the command) and keeps `currentWorkingDir` between invocations.
- **Background processes** are recorded in `~/.yumi/bg-processes.json`, with their captured output in `~/.yumi/bg-output/<id>.txt`.
- **Secret censoring:** environment values and arguments matching sensitive patterns are redacted before being echoed — `API_KEY`, `SECRET`, `TOKEN`, `PASSWORD`, `CREDENTIAL`, `AUTH`, `BEARER`, provider-specific keys (`OPENAI_*`, `GEMINI_*`, `GH_*`, `AWS_*`, `STRIPE_*`, `GOOGLE_*`, `KIRO_*`, `ANTHROPIC_*`), and short secret-bearing flags.

## The live tail (how output reaches the tool card)

This is the distinctive mechanism that makes shell output appear **live** in the UI:

```
 claude → mcp__yumi-bash__RunBash → yumi-mcp-bash.cjs runs the command
                                         │ writes output, line-batched
                                         ▼
                    %TEMP%\yumi-bash-<sid>.log   (preferred path from --stream-file=
                                         │        or $YUMI_STREAM_FILE; default in os.tmpdir())
                                         ▼  tail (poll ~150ms, offset cursor)
                    mcp/monitor.rs spawn_monitor → emit "bash-output" {sessionId, chunk}
                                         ▼
                    store._onBashOutput → ToolCard `live` pane (live terminal)
```

- The server flushes accumulated output to the stream file every **150 ms**, or sooner if more than **32 KB** is pending, splitting on newlines.
- The Rust monitor (`mcp/monitor.rs`) tails the file with an offset cursor and emits each new chunk on the `bash-output` channel.
- The store appends the chunk to the **last running tool card**, which `ToolCard` renders as a streaming **LIVE** pane.

This was verified end-to-end: a `for i in 1..6; do echo tick-$i; sleep 1` loop
streamed `tick-1..4` into the tool card mid-run (`docs/evidence/p2-bashmon-livetail.png`).

## AskUserQuestion file-IPC

`AskUserQuestion` round-trips through the filesystem so the GUI can answer:

- The server writes the pending question to `~/.yumi/askuser/<sid>.json` (mode `0o600` on POSIX) and **polls** for the answer at `~/.yumi/askuser/<sid>.answer.json`.
- Poll interval **250 ms**, timeout **5 minutes**.
- The UI watches for the pending file, renders the choice popover, and writes the answer file.

## Process lifetime

The MCP bash server is a child of the `claude` process (passed via `--mcp-config`),
not a separate long-lived sidecar. The spawned `claude` PID is registered with
`process::guard::track` in `spawner.rs`, so both `claude` and any processes it owns
are tree-killed if the host panics or exits — the server never leaks an idle node
process.

## Tests

`resources/tests/mcp_server.test.mjs` (run with `node`) drives the server over
stdio: `initialize` (asserts protocol version `2024-11-05` and `serverInfo.name ==
"yumi"`), `tools/list` (asserts exactly the 5 tool names), a `RunBash` echo,
**cwd tracking** across a `pwd → cd .. → pwd` sequence, a redirection + stream-file
check, and a background-process lifecycle (asserts a `bg-` id is returned and
listed). **12 assertions, all passing.**

## See also

- [Backend (Rust)](backend-rust.md) — `claude/spawner.rs` (config + `--disallowedTools`) and `mcp/monitor.rs` (the tail).
- [Stream Pipeline](stream-pipeline.md) — how `bash-output` reaches the tool card.
- [Data & Persistence](data-and-persistence.md) — the `~/.yumi` files the server writes.
- [Features & Shortcuts](features-and-shortcuts.md) — enabling the bash monitor.
