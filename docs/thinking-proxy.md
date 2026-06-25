---
title: "Thinking Proxy"
tags: [subsystem, thinking-proxy, proxy]
---

# Thinking Proxy

*`resources/thinking-proxy.cjs` + `claude/proxy.rs` — a localhost proxy that injects interleaved-thinking config so adaptive models stream their reasoning into the UI.*


---

## Why it exists

The Claude CLI doesn't expose a flag to force interleaved/extended thinking on. To
get thinking blocks to stream, Yumi points the spawned `claude` at a small
localhost HTTP proxy via `ANTHROPIC_BASE_URL`. The proxy forwards every request to
the real Anthropic API but **injects the thinking configuration** into the request
body first. It is enabled by the `thinking` setting (default **on**) and is
Claude-only.

## Launcher: `claude/proxy.rs`

`ProxyState::ensure_running()` is lazy and cached:

1. Find a free port in **18800–18899**.
2. Locate the script by walking up from the executable for `resources/thinking-proxy.cjs` (works in both the bundled layout and `--no-bundle` dev — it never depends on the original app's install dir).
3. Spawn `node <script>` with `THINKING_PROXY_PORT=<port>`.
4. **Probe before use:** TCP-connect to the port (retrying ~100 ms, up to a 3 s budget). Only if the connection succeeds does it return `http://127.0.0.1:<port>`.
5. If `node` is missing or the probe times out → return `None` (the spawner then lets `claude` talk to the real API directly) and **do not retry** for the rest of the app session.

This probe-before-use is a deliberate reliability fix: returning the URL as soon as
`node` merely *started* (before it was listening) could leave `claude` pointed at a
dead URL and hang the whole chat. See [Reliability & Design](reliability-and-design.md).

## Injection: `thinking-proxy.cjs`

The proxy inspects the model in each request and injects the appropriate thinking block:

| Model class | Injected `thinking` | Extra |
|-------------|---------------------|-------|
| **Adaptive** (Fable, Opus 4.6+/4.7+/4.8+, Sonnet 4.6+) | `{ type: "adaptive", display: "summarized" }` | — |
| **Pre-4.6** | `{ type: "enabled", budget_tokens: <BUDGET> }` | adds header `anthropic-beta: interleaved-thinking-2025-05-14` |

- **Budget tokens:** `THINKING_BUDGET_TOKENS` env, default **10 000** (used only for the pre-4.6 path).
- **Port:** `THINKING_PROXY_PORT`, clamped to 18800–18899 (defends against a poisoned env value).
- **DNS cache:** upstream DNS results cached with a 60 s TTL.

## Security

The proxy is a network egress point, so it is hardened:

- **Hostname allowlist:** only `api.anthropic.com` (or a true subdomain) is accepted; lookalikes such as `evil.anthropic.com.attacker.io` are rejected.
- **DNS-rebinding defence:** resolved IPs that are private, loopback, CGNAT (100.64.0.0/10), ULA (`fc00::/7`), or IPv6 link-local (`fe80::/10`) are rejected.

## When it's off

If thinking is disabled, the proxy fails to start, or the probe fails, `claude` runs
against the real API with **no proxy** — the core chat is never blocked, you just
don't get the injected interleaved thinking. Thinking blocks that the model emits on
its own still render normally.

## See also

- [Backend (Rust)](backend-rust.md) — `claude/proxy.rs` and the spawner's `ANTHROPIC_BASE_URL` wiring.
- [Reliability & Design](reliability-and-design.md) — the probe-before-use fix.
- [Stream Pipeline](stream-pipeline.md) — how `thinking_delta` events render.
