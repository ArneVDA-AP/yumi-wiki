---
title: "Stream Pipeline"
tags: [reference, stream-json, parser]
---

# Stream Pipeline

*How one line of `claude` `stream-json` becomes a rendered chat message — the parser mapping and the reducer rules.*


---

## The two halves

```
 claude stdout (one JSON object per line)
        │
        ▼  parse_line(line) -> Vec<ClaudeEvent>     src-tauri/src/claude/stream_parser.rs   (Rust, pure)
        │  emit "claude-event" {sessionId, event}    src-tauri/src/claude/spawner.rs
        ▼
 reduceEvent(current, event) -> ReduceResult        src/lib/streamParse.ts                  (TS, pure)
        │  fold into the in-flight ChatMessage
        ▼
 React renders the message blocks                   src/components/chat/*
```

Both halves are **pure functions** and both are **non-lossy**: any line the parser
doesn't recognise becomes a `raw` event rather than being dropped, and the reducer
simply ignores `raw`. The parser is unit-tested in `cargo test` against real
captured bytes.

---

## Half 1 — parser: `stream-json` line → `ClaudeEvent`

`parse_line` reads the top-level `type` field of the JSON line and routes:

| Line `type` | Handler | Produces |
|-------------|---------|----------|
| `system` | `parse_system` | `system { subtype, model, claudeSessionId (=session_id), cwd, tools, data }` |
| `assistant` | `parse_assistant` | iterate `message.content[]`: `text`→`assistant_text`, `thinking`→`assistant_thinking`, `tool_use`→`tool_use{id,name,input}` |
| `user` | `parse_user` | iterate `message.content[]`: `tool_result`→`tool_result{toolUseId,content,isError}` |
| `stream_event` | `parse_stream_event` | unwrap the inner `event` (see below) |
| `result` | `parse_result` | `result { isError, durationMs, numTurns, totalCostUsd, resultText (=result), usage, modelUsage }` |
| `rate_limit_event` | — | `rate_limit { data }` |
| `error` | — | `error { message }` |
| *(unknown / unparseable)* | — | `raw { line }` |

### `stream_event` unwrapping
`stream_event` lines carry an inner Anthropic streaming event, routed on `event.type`:

- **`message_delta`** → read `delta.stop_reason`:
  - If the stop reason is **terminal** (anything **except** `tool_use` and `pause_turn`) → emit **`turn_done`** with a usage snapshot. This is the signal Yumi finalizes the turn on (see below).
  - Otherwise → `raw` (non-terminal; e.g. a `tool_use` pause mid-turn).
- **`content_block_delta`** → read `delta.type`:
  - `text_delta` → `text_delta { text: delta.text }` (live typing).
  - `thinking_delta` → `thinking_delta { text: delta.thinking }`.
  - other → `raw`.
- other inner types (`message_start`, `content_block_start/stop`, `message_stop`, …) → `raw` (safely ignorable by the UI).

### Non-lossy details
- An `assistant` line with no recognisable content blocks still yields a single `raw` event.
- `tool_result` content is flattened: a string is used as-is; an array of `{type:"text", text}` blocks is concatenated; anything else is stringified.
- Unknown thinking shapes fall back from `delta.thinking` to `delta.text`.

### Why `turn_done` exists
`turn_done` is emitted from the terminal `message_delta` — which arrives **as soon
as the answer finishes** — whereas the `result` line only arrives **after** the
spawned `claude` runs the user's post-turn Stop hooks (potentially 30–60 s later).
Finalizing on `turn_done` is what keeps the UI responsive; the late `result` is
still parsed afterwards for analytics and cost. Full reasoning: [Reliability & Design](reliability-and-design.md).

---

## Half 2 — reducer: `ClaudeEvent` → `ChatMessage`

`reduceEvent(current, event)` folds an event into the in-flight assistant
`ChatMessage` and returns a `ReduceResult { message, finalized, claudeSessionId?,
usage?, rateLimit?, error? }`.

| Event `kind` | Effect on the message |
|--------------|-----------------------|
| `text_delta` | append to the active text block (create one if none); `streaming = true` |
| `thinking_delta` | append to the active thinking block (create if none); `streaming = true` |
| `assistant_text` | commit a complete text block (replace/append) |
| `assistant_thinking` | commit a complete thinking block |
| `tool_use` | push a new tool-card block, `running: true` |
| `tool_result` | find the tool card by `toolUseId`, attach `content`, set `running: false` |
| `usage` | surface `{inputTokens, outputTokens, cacheRead, cacheCreation}` (drives the token bar) |
| `rate_limit` | surface the raw payload |
| `turn_done` | **finalize**: `streaming: false`, close any still-running tool cards, mark `finalized` |
| `result` | **finalize** with `cost` + `durationMs`; if nothing streamed, synthesize the message from `resultText` |
| `system` | no message change; surface `claudeSessionId` |
| `error` | surface the error message |
| `raw` | no-op (non-lossy) |

### Live bash output
`bash-output` is **not** a `ClaudeEvent` — it is a sibling channel. The store's
`_onBashOutput(sessionId, chunk)` appends the chunk to the **last running tool
card's** `live` field, which `ToolCard` renders as a streaming terminal pane. This
is how the [MCP Bash Server](mcp-bash-server.md) output appears live mid-turn.

### Late cost
`claude-cost` is also a sibling channel. `_onCost(sessionId, cost, durationMs)`
attaches the cost + duration to the **most recent finalized** assistant message's
footer — handling the case where `result` (and thus cost) arrives after the turn
was already finalized on `turn_done`.

---

## Persistence of blocks
On finalize the store calls `_persist(tabId)`, which serialises each message's
`blocks` to JSON (`serializeBlocks`) and writes them to the `messages` table.
`deserializeBlocks` reverses this on resume, with a tolerant fallback to plain text
for any legacy/garbled content. See [Data & Persistence](data-and-persistence.md).

## See also

- [IPC Contract](ipc-contract.md) — the `ClaudeEvent` field reference.
- [Backend (Rust)](backend-rust.md) — `stream_parser.rs` and `spawner.rs`.
- [Frontend (React)](frontend-react.md) — the store and `streamParse.ts`.
- [Reliability & Design](reliability-and-design.md) — finalize-on-`end_turn` and the late-`result` handling.
