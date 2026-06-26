---
title: "IPC Contract"
tags: [reference, ipc, contract, stream-json]
---

# IPC Contract

*The frozen Rust⇄React interface: the `ClaudeEvent` union, the Tauri command surface, the event channels, and the `ipc.ts` wrappers.*


---

## The rule

`src/lib/types.ts` and `src/lib/ipc.ts` are **authoritative** for all shapes and
names. Rust mirrors them:

- Every Rust DTO and the `ClaudeEvent` enum use `#[serde(rename_all = "camelCase")]`, so the JSON the frontend receives already has camelCase field names (`claudeSessionId`, `inputTokens`, `durationMs`).
- Tauri command **argument** names are camelCase on the JS side (`sessionId`, not `session_id`) — `ipc.ts` passes them that way.
- The `ClaudeEvent` enum serializes with `#[serde(tag = "kind", ...)]`, so each event is `{ "kind": "...", ... }` — matching the discriminated union in `types.ts`.

---

## `ClaudeEvent` — the stream event model

The discriminated union emitted by Rust (`claude/stream_parser.rs`) and consumed by
the reducer (`src/lib/streamParse.ts`). Each variant is tagged by `kind`:

| `kind` | Fields | Meaning |
|--------|--------|---------|
| `system` | `subtype, model?, claudeSessionId?, cwd?, tools?, data?` | a `type:"system"` line (init/hook/etc.); carries the real session uuid |
| `assistant_text` | `text` | a **complete** assistant text block |
| `assistant_thinking` | `text` | a **complete** thinking block |
| `tool_use` | `id, name, input` | the model invoked a tool |
| `tool_result` | `toolUseId, content, isError?` | a tool's result (matched back to its `tool_use`) |
| `text_delta` | `text` | a streamed **partial** text token (live typing) |
| `thinking_delta` | `text` | a streamed partial thinking token |
| `usage` | `inputTokens?, outputTokens?, cacheRead?, cacheCreation?` | token usage snapshot |
| `rate_limit` | `data` | a `rate_limit_event` line (raw payload) |
| `turn_done` | `stopReason?, inputTokens?, outputTokens?, cacheRead?, cacheCreation?` | terminal `message_delta` — **finalizes the turn** before the late `result` |
| `result` | `isError, durationMs?, numTurns?, totalCostUsd?, resultText?, usage?, modelUsage?` | the final `result` line (cost + totals) |
| `error` | `message` | an error line |
| `raw` | `line` | unparseable or unknown line — **non-lossy fallback** |

How each `kind` is produced is detailed on the [Stream Pipeline](stream-pipeline.md) page; how each mutates a message is in the reducer table there.

### Envelope & sibling event payloads

```ts
ClaudeEventEnvelope = { sessionId: string, event: ClaudeEvent }   // "claude-event"
ClaudeComplete      = { sessionId: string, code: number }         // "claude-complete"
ClaudeErrorEvt      = { sessionId: string, message: string }      // "claude-error"
ClaudeSessionId     = { sessionId: string, claudeSessionId: string } // "claude-session-id"
ClaudeCost          = { sessionId: string, cost?: number, durationMs?: number } // "claude-cost"
BashOutput          = { sessionId: string, chunk: string }        // "bash-output"
AgentEvent          = { agent: AgentInfo }                        // "agent-event"
```

### Other key types

- `Settings = { provider, routerBaseUrl, model, theme, vimMode, thinking, showRateLimit, bashMonitor, autoCompactThreshold }`. `DEFAULT_SETTINGS`: `provider:"claude"`, `routerBaseUrl:""`, `model:"claude-opus-4-8"`, `theme:"oled"`, `vimMode:false`, `thinking:true`, `showRateLimit:true`, `bashMonitor:false`, `autoCompactThreshold:0.75`.
- `SpawnOpts = { sessionId, cwd, prompt, model, provider?, routerBaseUrl?, resumeId?, thinking?, bashMonitor? }`.
- `MessageBlock` = a `text` | `thinking` | `tool_use` block (the tool block carries `id, name, input, result?, isError?, running?, live?`). `ChatMessage = { id, role, blocks[], streaming?, cost?, durationMs? }`.
- `SessionMeta`, `SessionDetail`, `StoredMessage`, `Analytics` (+ `ByModel`/`ByDate`), `RecentProject`, `GitStatus`, `DirEntry`, `UsageLimits` (`rateLimited, live?, windowType?, fiveHourPct?, sevenDayPct?, resetsAt?`), `AgentInfo`, `ProviderInfo`/`PROVIDERS`. (Field lists in `types.ts`; storage shapes on [Data & Persistence](data-and-persistence.md).)

---

## Tauri commands

The complete `#[tauri::command]` surface (registered in `lib.rs`), with the
matching `ipc.ts` wrapper. `app: AppHandle` is implicit and omitted below.

### Claude lifecycle — `commands/claude.rs`
| Command | Args | Returns | `ipc.ts` |
|---------|------|---------|----------|
| `get_claude_version` | — | `String` | `getClaudeVersion()` |
| `spawn_claude` | `opts: SpawnOpts` | `String` (session id) | `spawnClaude(opts)` |
| `interrupt_claude` | `sessionId: String` | `()` | `interruptClaude(sessionId)` |
| `get_usage_limits` | `force: bool` | `UsageLimits` *(honestly inert: `{rateLimited:false, live:false}`)* | `getUsageLimits(force?)` |

### Settings — `commands/settings.rs`
| Command | Args | Returns | `ipc.ts` |
|---------|------|---------|----------|
| `get_settings` | — | `Settings` | `getSettings()` |
| `save_settings` | `settings: Settings` | `()` | `saveSettings(settings)` |
| `set_theme` | `theme: String` | `()` | `setTheme(theme)` |

### Sessions / analytics — `commands/db.rs`
| Command | Args | Returns | `ipc.ts` |
|---------|------|---------|----------|
| `db_list_sessions` | — | `Vec<SessionMeta>` | `dbListSessions()` |
| `db_get_session` | `id: String` | `SessionDetail` | `dbGetSession(id)` |
| `db_save_session` | `detail: SessionDetail` | `()` | `dbSaveSession(detail)` |
| `db_delete_session` | `id: String` | `()` | `dbDeleteSession(id)` |
| `db_get_analytics` | — | `Analytics` | `dbGetAnalytics()` |

### Projects & filesystem — `commands/info.rs`, `commands/fs.rs`
| Command | Args | Returns | `ipc.ts` |
|---------|------|---------|----------|
| `list_projects` | — | `Vec<RecentProject>` | `listProjects()` |
| `add_recent_project` | `path: String` | `()` | `addRecentProject(path)` |
| `pick_directory` | — | `Option<String>` | `pickDirectory()` |
| `list_directory` | `path: String` | `Vec<DirEntry>` | `listDirectory(path)` |
| `read_text_file` | `path: String` | `String` | `readTextFile(path)` |

### Git — `commands/git.rs`
| Command | Args | Returns | `ipc.ts` |
|---------|------|---------|----------|
| `git_status` | `cwd: String` | `GitStatus` | `gitStatus(cwd)` |
| `git_diff` | `cwd: String, staged: bool` | `String` | `gitDiff(cwd, staged?)` |

### Background agents — `agents.rs`
| Command | Args | Returns | `ipc.ts` |
|---------|------|---------|----------|
| `list_agents` | — | `Vec<AgentInfo>` | `listAgents()` |
| `start_agent` | `cwd, prompt, model` | `AgentInfo` | `startAgent(cwd, prompt, model)` |
| `merge_agent_branch` | `id` | `AgentInfo` | `mergeAgentBranch(id)` |
| `cancel_agent` | `id` | `AgentInfo` | `cancelAgent(id)` |
| `remove_agent` | `id` | `()` | `removeAgent(id)` |
| `agent_diff` | `id` | `String` | `agentDiff(id)` |

---

## Event channels (Rust → React)

Emitted via `app.emit(...)`; subscribed via `listen()` wrappers in `ipc.ts`.

| Channel | Payload | Emitter | `ipc.ts` subscriber |
|---------|---------|---------|---------------------|
| `claude-event` | `{sessionId, event}` | `spawner.rs` | `onClaudeEvent(cb)` |
| `claude-complete` | `{sessionId, code}` | `spawner.rs` | `onClaudeComplete(cb)` |
| `claude-error` | `{sessionId, message}` | `spawner.rs` | `onClaudeError(cb)` |
| `claude-session-id` | `{sessionId, claudeSessionId}` | `spawner.rs` | `onClaudeSessionId(cb)` |
| `claude-cost` | `{sessionId, cost, durationMs}` | `spawner.rs` | `onClaudeCost(cb)` |
| `bash-output` | `{sessionId, chunk}` | `mcp/monitor.rs` | `onBashOutput(cb)` |
| `agent-event` | `{agent}` | `agents.rs` | `onAgentEvent(cb)` |

All subscribers return a Tauri `UnlistenFn`. The store installs them in
`bootstrap()` and routes them to the `_on*` handlers (see [Frontend](frontend-react.md)).

## See also

- [Stream Pipeline](stream-pipeline.md) — where `ClaudeEvent`s come from and what they do.
- [Backend (Rust)](backend-rust.md) — the command implementations.
- [Data & Persistence](data-and-persistence.md) — `SessionMeta`/`SessionDetail`/`Analytics` storage.
- The live source of truth: `yumi/CONTRACT.md`.
