---
title: "Overview"
tags: [yumi, overview]
---

# Overview

*What Yumi is, where it came from, and the boundaries of the project.*


---

## What Yumi is

**Yumi** is a native desktop GUI for the **Claude Code CLI**. Instead of talking to
the Anthropic API directly, it launches your installed `claude` binary as a child
process for every turn, reads its `--output-format stream-json` output line by
line, and renders that stream — assistant text, **thinking blocks**, tool calls,
tool results, usage, and the final result — in a chat UI.

Because the CLI does the work, Yumi inherits everything the CLI already supports:

- **Auth** is the CLI's: Yumi uses your existing Claude Pro/Max subscription, no API key wiring.
- Your **`CLAUDE.md`**, **MCP servers**, **hooks**, **skills**, and **subagents** all apply unchanged.
- Model selection, `--resume`, permissions, and tool use behave exactly as they do on the command line.

Yumi is essentially a **rich renderer and session manager** wrapped around the CLI,
plus a few sidecars (an MCP bash server, a thinking proxy, and a bundled plugin)
that add live-streaming shell output and interleaved thinking.

## Relationship to Yume

Yumi is a from-scratch, faithful clone of **Yume** (`aofp/yume`), a commercial
Tauri 2 + React 19 desktop GUI for Claude Code. The clone rebuilds the **same
stack** and the **same architecture** — spawn the real `claude`, parse
`stream-json`, stream live shell output via a temp log tailed by a Rust monitor,
inject thinking via a localhost proxy, and sync a plugin to `~/.claude`.

The names mirror this lineage: where Yume used `yume.exe`, `yume-mcp-bash.cjs`,
`%TEMP%\yume-bash-<session>.log`, and `yume/agent-<id>` branches, Yumi uses
`yumi.exe`, `yumi-mcp-bash.cjs`, `%TEMP%\yumi-bash-<session>.log`, and
`yumi/agent-<id>`.

The original was reverse-engineered from the installed build to recover its spawn
command, IPC surface, event channels, and the BASH-MONITOR streaming mechanism;
see `PLAN.md` at the repo root for that reconnaissance.

## Scope & philosophy

Yumi is a **personal** project with one deliberate subtraction from Yume:

- **No commercial surface.** Licensing, payments, paywalls, demo limits, and
  auto-update are **intentionally not cloned**. There is simply no gate — Yumi is
  free and open. (Cloning a paywall in order to defeat it would be wrong; the
  clone instead has no paywall at all.)

Everything else aims for honest parity. The project tracks its claims rigorously:

- A **parity matrix** (`yumi/PARITY.md`) labels each feature `✅ done & verified`,
  `🟢 done (lighter verification)`, `🟡 scoped/stub`, or `⬜ intentionally omitted`,
  so the clone never overclaims downstream.
- Features verified in the **real GUI** carry screenshot evidence in
  `yumi/docs/evidence/`.

## Feature summary

A high-level catalogue (full details and status in
[Features & Shortcuts](features-and-shortcuts.md) and `yumi/PARITY.md`):

**Core loop**
- Locate + spawn `claude`; stream-json parsing into typed events; `--resume` support.
- Chat render: user/assistant text, thinking blocks, collapsible tool cards, results, markdown + syntax highlighting.
- Prompt input, send, interrupt/stop, new session/tab.
- Tabs & sessions with SQLite persistence and resume.
- Settings (model, theme, toggles, auto-compact threshold), persisted.

**Productivity**
- Up to **6 split panes** (F7/F8), each a full chat pane.
- **History rollback** — rewind to a past user turn and re-edit.
- **Background agents** — run a headless `claude -p` in an isolated git worktree, then merge the branch back.
- **Voice dictation** (F5) via the Web Speech API.
- Git panel, files panel + `@`-mentions, command palette, analytics dashboard, rate-limit pills.

**Sidecars**
- **MCP bash server** with live output tailing into the tool card.
- **Thinking proxy** for interleaved thinking on adaptive models.
- **Yumi plugin** (4 agents, 8 commands, 1 guard hook).

**Multi-provider** (experimental) — a provider abstraction can target Gemini / GPT (Codex) / Kiro; only the **Claude** path is verified.

## Parity tiers

The build was organised into three tiers (from `PLAN.md`):

- **P0 — core loop:** must work end-to-end and be verified. (All ✅.)
- **P1 — important:** implemented with lighter verification (git/files panels, command palette, analytics, rate limits, `@`-mentions, plugin). (All 🟢.)
- **P2 — parity-noted:** scoped, stubbed, or honestly labelled. Most P2 items
  ended up fully built and verified (split panes, history rollback, background
  agents, bash monitor); multi-provider is partly stubbed; licensing/payments and
  auto-update/VSCode companion are **intentionally omitted**.

## See also

- [Architecture](architecture.md) — how the pieces fit together.
- [Features & Shortcuts](features-and-shortcuts.md) — the full feature catalogue with status.
- [Reliability & Design](reliability-and-design.md) — the design decisions that make it robust.
