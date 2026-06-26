---
title: "Frontend (React)"
tags: [reference, react, frontend, zustand]
---

# Frontend (React)

*The React 19 + TypeScript UI: the zustand store, the stream reducer, and the component map.*


---

## Layout

```
src/
  main.tsx             React root (StrictMode → App)
  App.tsx              shell: sidebar + topbar + chat area + panels + overlays
  styles.css           global styles + theme CSS variables
  lib/
    types.ts           FROZEN data/event contract
    ipc.ts             FROZEN typed wrappers over invoke()/listen()
    store.ts           zustand store (state + actions)
    streamParse.ts     ClaudeEvent → ChatMessage reducer
    themes.ts          6 themes + applyTheme
    shortcuts.ts       global keyboard map
    models.ts          contextWindowFor() — per-model context-window lookup
  components/
    chat/    MessageList, MessageItem, ChatPane, ToolCard, ThinkingBlock, RateLimitPills
    input/   PromptInput, TokenBar, MentionPopup, useDictation
    tabs/    Sidebar, TabBar
    panels/  GitPanel, FilesPanel, AgentsPanel
    overlays/ Overlay, SettingsOverlay, CommandPalette, AnalyticsOverlay, ProjectPicker
    common/  Icon, Markdown, CodeBlock, useCopy, fuzzy, hljsLanguages
```

`types.ts` and `ipc.ts` are the **frozen contract** with the Rust backend — they
are read-only for UI work and documented on the [IPC Contract](ipc-contract.md)
page. Everything else in `src/` is free to change.

---

## State: `lib/store.ts`

A single zustand store holds all UI state. Top-level fields:

| Field | Type | Meaning |
|-------|------|---------|
| `tabs` | `Tab[]` | open chat tabs (each has its own messages, draft, streaming flag, model, cwd, `claudeSessionId`); each tab also carries `lastUsage` (`inputTokens`, `outputTokens`, `cacheRead`, `cacheCreation`) filled live from `message_start` |
| `activeTabId` | `string \| null` | focused tab |
| `settings` / `settingsLoaded` | `Settings` / `bool` | persisted settings |
| `sessions` | `SessionMeta[]` | past sessions for the sidebar |
| `projects` | `RecentProject[]` | recent project list |
| `usage` | `UsageLimits \| null` | rate-limit pills source |
| `overlay` | `null \| "settings" \| "palette" \| "analytics"` | active modal |
| `panel` | `null \| "git" \| "files" \| "agents"` | active side panel |
| `sidebarOpen` | `bool` | sessions sidebar visibility |
| `splitTabs` | `string[]` | ordered tab ids for the split grid (≤1 ⇒ normal single view) |
| `voiceActive` | `bool` | voice dictation on/off |
| `bootstrapped` | `bool` | startup completed |

### Actions (grouped)

**Lifecycle / tabs**
- `bootstrap()` — load settings, install IPC listeners, load sessions/projects/usage, open the first tab.
- `newTab(opts?)`, `closeTab(id)`, `setActiveTab(id)`, `renameTab(id, title)`, `setDraft(id, draft)`.

**Conversation**
- `send(tabId, prompt)` — append the user message + an empty streaming assistant, then `spawnClaude(opts)` (opts include `settings.routerBaseUrl` so the spawner can set `ANTHROPIC_BASE_URL` for routed providers).
- `stop(tabId)` — `interruptClaude` and finalize.
- `resumeSession(meta)` — open a past session in a new tab and load its messages from the DB.
- `rollbackTo(tabId, messageId)` — rewind the conversation to before a user turn and reload that prompt into the draft for edit/resend, then re-persist.
- `refreshSessions()`, `deleteSession(id)`.

**Projects & settings**
- `setProject(path)`, `refreshProjects()`.
- `updateSettings(patch)` — merge, apply theme, persist to the backend.
- `refreshUsage(force?)`.

**View**
- `setOverlay(o)`, `togglePanel(p)`, `setSidebarOpen(v)`.
- `addPane()` / `removePane()` / `closePane(tabId)` — the split-pane grid (F7/F8), up to 6 panes.
- `toggleVoice()` / `setVoiceActive(v)`.

**Agents**
- `refreshAgents()`, `startAgent(prompt)`, `mergeAgent(id)`, `cancelAgent(id)`, `removeAgent(id)`, `getAgentDiff(id)`.

**Internal event handlers** (wired to `ipc.ts` listeners in `bootstrap`)
- `_onEvent(sessionId, event)` — route a `ClaudeEvent` through `reduceEvent`, updating messages / usage / cost.
- `_onComplete`, `_onError`, `_onSessionId` — process lifecycle.
- `_onCost(sessionId, cost, durationMs)` — attach cost + duration to the most recent finalized assistant message (handles the late `claude-cost` after `end_turn`).
- `_onBashOutput(sessionId, chunk)` — append live shell output to the last running tool card.
- `_onAgentEvent(agent)` — upsert agent status.
- `_persist(tabId)` — save the tab's conversation to the DB.

---

## Reducer: `lib/streamParse.ts`

`reduceEvent(current: ChatMessage | null, event: ClaudeEvent) → ReduceResult`
folds one event into the in-flight assistant message. The per-`kind` rules are on
the [Stream Pipeline](stream-pipeline.md) page; in brief:

- `text_delta` / `thinking_delta` → append to (or create) the active text/thinking block; `streaming = true`.
- `assistant_text` / `assistant_thinking` → commit the complete block.
- `tool_use` → push a tool card (`running: true`); `tool_result` → match by `toolUseId`, attach content, `running: false`.
- `usage` → surface token counts; `rate_limit` → surface raw payload.
- `turn_done` → finalize (`streaming: false`, close running tools); `result` → finalize with cost + duration (synthesizing a message from `resultText` if nothing streamed).
- `system` → surface `claudeSessionId`; `raw` → no-op (non-lossy).

Helpers: `newMessageId()`, `emptyAssistant()`, and
`serializeBlocks`/`deserializeBlocks` (JSON ⇄ `MessageBlock[]` for DB storage,
with a tolerant plain-text fallback).

---

## Themes: `lib/themes.ts`

Six themes: **`oled`** (pure-black `#000000`, default), `dark`, `dim`, `light`,
`nord`, `rose`. A theme is a set of CSS custom properties (`--bg`, `--text`,
`--accent`, `--thinking`, `--tool`, `--user-bubble`, `--code-bg`, semantic
`--success/--warn/--error`, etc.). `applyTheme(name)` writes the variables onto
`document.documentElement`, sets `data-theme`, and sets `colorScheme`;
`getTheme(name)` defaults to `oled` on an unknown name.

---

## Keyboard map: `lib/shortcuts.ts`

A global `window` keydown dispatcher. The full table is on
[Features & Shortcuts](features-and-shortcuts.md); highlights: `F5` voice,
`F7/F8` add/remove pane, `Ctrl+T/W` new/close tab, `Ctrl+P` palette, `Ctrl+,`
settings, `Ctrl+G/E` git/files panels, `Ctrl+Y` analytics, `Ctrl+Shift+A` agents,
`Ctrl+Tab` cycle tabs, `Esc` close overlay/panel.

---

## Components

### `App.tsx` + `main.tsx`
`main.tsx` renders `<App/>` in StrictMode. `App.tsx` is the shell: a CSS-grid
layout (driven by sidebar/panel visibility) composing the sessions sidebar, a
topbar (tab bar + rate-limit pills + panel/settings buttons), the chat area
(single view **or** the split-pane grid), the side panels, and the overlays. It
owns `bootstrap()` and installs the keyboard shortcuts.

### `chat/`
| Component | Renders |
|-----------|---------|
| `MessageList` | windowed scroll list (caps the tail at ~120 messages with a "load earlier" button); auto-sticks to the bottom while streaming unless the user scrolls up |
| `MessageItem` | one message row — user messages are right-aligned bubbles with a **rewind** button; assistant messages render their blocks plus a cost/duration footer |
| `ChatPane` | one pane in the split grid; renders a single tab's chat + composer; clicking it activates the tab |
| `ToolCard` | collapsible tool call — name, pretty-printed input JSON, running spinner → ✓/✗, result body, and a **LIVE** pane for streamed bash output |
| `ThinkingBlock` | collapsible dimmed thinking block (auto-expands while streaming, collapses after) |
| `RateLimitPills` | 5h / 7d rate-limit pills; when `usage.live` is false (no real signal yet) renders greyed `—` pills with an "unavailable" tooltip — never fake numbers; when a live `rate_limit_event` arrives (`usage.live = true`), shows real status, window type, and reset time; renders nothing when `usage` is null |

### `input/`
| Component | Renders |
|-----------|---------|
| `PromptInput` | the composer: token bar + auto-growing textarea + send/stop, `@`-mention file picker, voice toggle, error banner, model label |
| `TokenBar` | thin context-usage bar — dynamic window via `contextWindowFor(tab.model)` (`models.ts`; 200K for Claude, 1M for Gemini, etc.); sums `inputTokens + cacheRead + cacheCreation + outputTokens`; fills live mid-stream from `message_start` usage (not only end-of-turn); colour-coded by the auto-compact threshold |
| `MentionPopup` | floating fuzzy-filtered `@`-mention file list, keyboard navigable |
| `useDictation` | hook wrapping the Web Speech API for F5 voice dictation; degrades gracefully when WebView2 has no speech backend |

### `tabs/`
| Component | Renders |
|-----------|---------|
| `Sidebar` | past sessions grouped by project, search filter, resume/delete, footer links to Settings/Analytics |
| `TabBar` | horizontal tab strip with status dots, titles, hover-close, and a "+" |

### `panels/`
| Component | Renders |
|-----------|---------|
| `GitPanel` | git branch + ahead/behind, staged/unstaged/untracked sections, working/staged diff toggle with highlighting |
| `FilesPanel` | file browser rooted at the tab's cwd, navigate in/out (clamped at root), text preview (capped ~40k chars) |
| `AgentsPanel` | background-agent launcher + status dashboard (status badges, cancel/merge/diff/remove) |

### `overlays/`
| Component | Renders |
|-----------|---------|
| `Overlay` | shared modal shell (dim backdrop + centered card, optional title, wide/bare modifiers) |
| `SettingsOverlay` | provider/model select, theme swatches, behaviour toggles (vim, thinking, rate-limit, bash monitor), auto-compact slider, shortcut reference |
| `CommandPalette` | fuzzy command list (new/close tab, settings, analytics, git, files, open project, refresh usage, `/clear`, `/compact`, theme switch) |
| `AnalyticsOverlay` | totals, per-model bar chart, per-date trend; tolerant empty state |
| `ProjectPicker` | empty-state panel when the tab has no cwd: browse + recent projects |

### `common/`
| Module | Provides |
|--------|----------|
| `Icon` | dependency-free inline SVG icon set (stroke-based, `currentColor`) |
| `Markdown` | `react-markdown` + `remark-gfm`; fenced code → `CodeBlock`, links open in new tab |
| `CodeBlock` | `highlight.js` syntax highlighting + copy button + language label |
| `useCopy`, `fuzzy`, `hljsLanguages` | clipboard hook, fuzzy scorer, highlight.js language registration |

## See also

- [IPC Contract](ipc-contract.md) — the `types.ts`/`ipc.ts` contract these consume.
- [Stream Pipeline](stream-pipeline.md) — the reducer rules in full.
- [Features & Shortcuts](features-and-shortcuts.md) — what each feature does and how to reach it.
