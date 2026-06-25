---
title: "Features & Shortcuts"
tags: [guide, features, shortcuts]
---

# Features & Shortcuts

*The feature catalogue with its parity status, and the complete keyboard map.*


---

## Keyboard shortcuts

The global keydown handler lives in `src/lib/shortcuts.ts`. (`Ctrl` is `Cmd` on macOS.)

| Key | Action |
|-----|--------|
| `Esc` | close the active overlay, then the active panel |
| `F5` | toggle voice dictation |
| `F7` | add a split pane (up to 6) |
| `F8` | remove the last split pane |
| `Ctrl+T` | new tab (inherits the current cwd) |
| `Ctrl+W` | close the active tab |
| `Ctrl+P` | command palette |
| `Ctrl+,` | settings |
| `Ctrl+G` | toggle the git panel |
| `Ctrl+E` | toggle the files panel |
| `Ctrl+Y` | analytics overlay |
| `Ctrl+L` | force-refresh rate limits |
| `Ctrl+O` | open project (folder picker) |
| `Ctrl+B` | toggle the sessions sidebar |
| `Ctrl+Shift+A` | toggle the background-agents panel |
| `Ctrl+Tab` | cycle to the next tab |
| `Ctrl+Shift+Tab` | cycle to the previous tab |

The same list is shown in the Settings overlay (the `SHORTCUT_HINTS` reference).

---

## Feature catalogue

Status legend matches `yumi/PARITY.md`: ✅ done & verified · 🟢 done (lighter
verification) · 🟡 scoped/stub · ⬜ intentionally omitted.

### Core chat loop ✅
- Locate + spawn `claude`, stream-json parsing, `--resume`.
- Render user/assistant text, **thinking blocks**, collapsible **tool cards** (input + result), markdown + syntax highlighting.
- Send, **interrupt/stop** (tree-kills via the [process registry](backend-rust.md)), new session/tab.
- Tabs & sessions, sidebar grouping by project, resume, SQLite persistence.

### Settings ✅
Provider, model, theme (6 themes, OLED default), vim toggle, thinking toggle,
rate-limit toggle, **bash-monitor toggle**, auto-compact threshold. Persisted to the
DB. See [Data & Persistence](data-and-persistence.md).

### Split panes (F7/F8) ✅
Up to **6** panes in a CSS grid, each a full `ChatPane` (own header, messages,
composer). F7 adds, F8 removes; click a pane to make it active. Backed by
`store.splitTabs`.

### History rollback ✅
A rewind affordance on every user message (`store.rollbackTo`) truncates the
conversation back to that turn and reloads the prompt into the composer for
edit/resend, then re-persists.

### Background agents (Ctrl+Shift+A) ✅
Run a headless `claude -p` in an **isolated git worktree** (`%TEMP%\yumi-agents\`,
branch `yumi/agent-<id>`), commit the result, then review the diff and **merge** it
back. The Agents panel shows status + diff + Merge/Cancel/Remove. Requires a git
repo cwd. Implementation: [`agents.rs`](backend-rust.md).

### Voice dictation (F5) 🟢
`useDictation` wraps the Web Speech API; interim + final transcripts append to the
active composer with a "Listening…" indicator. Degrades gracefully when WebView2
has no speech backend.

### Live bash monitor 🟢 (opt-in, off by default)
With the setting on, shell runs through the [MCP bash server](mcp-bash-server.md)
and streams **live** into the tool card's LIVE pane.

### Multi-provider 🟡
A provider abstraction (`provider.rs`) can target Gemini / GPT (Codex) / Kiro by
locating their CLI and shaping args per provider. **Only Claude is verified**; the
others are best-effort, unverified shims.

### P1 surfaces 🟢
- **Rate-limit pills** (5h / 7d) — the in-stream `rate_limit_event` is parsed; the `get_usage_limits` command is currently a stub, so live percentages aren't populated yet.
- **Git panel** — status + working/staged diff.
- **Files panel + `@`-mentions** — folder navigation and file insertion.
- **Command palette** — fuzzy slash-command launcher.
- **Token/context bar** — usage % from `message_delta`.
- **Project picker + recent projects**.
- **Analytics dashboard** — real totals, by-model, by-date (see [Data & Persistence](data-and-persistence.md)).

### Per-message cost footer 🟢
Each finalized assistant message shows `⏱ duration` + `⚡ $cost`, attached via the
late `claude-cost` side channel. Shown live; **not** persisted to the DB (the
message schema has no cost column), so it does not survive a reload-from-history.

### Intentionally omitted ⬜
Licensing / payments / paywall, auto-update, and the VSCode companion extension.

## See also

- [Features in code](frontend-react.md) — the components behind each surface.
- `yumi/PARITY.md` — the authoritative, per-feature status matrix with evidence.
- [Reliability & Design](reliability-and-design.md) — why interrupt, finalize, and cost behave as they do.
