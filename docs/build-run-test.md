---
title: "Build, Run & Test"
tags: [guide, build, windows-gnu]
---

# Build, Run & Test

*Toolchain, dependencies, the build/dev/test commands, and where the artifacts land.*


---

## Toolchain

- **Node** + **npm** for the React/Vite frontend.
- **Rust** (stable) with **Cargo** for the Tauri host.
- **Tauri 2** CLI (`@tauri-apps/cli`).
- **WebView2** runtime (present on Windows 11).

### Windows: builds on `windows-gnu`, no MSVC

The crate is declared `[lib] crate-type = ["rlib"]` in `src-tauri/Cargo.toml`
(**not** `cdylib`). This is the one setting that lets Tauri 2 build with the
`windows-gnu` toolchain without requiring MSVC. `build.rs` is the stock
`tauri_build::build()`. (Linking as an `rlib` rather than the default `cdylib`
is what avoids the MSVC-only link path on Windows.)

## Dependencies

### Frontend (`package.json`)
Runtime: `react ^19.1.0`, `react-dom ^19.1.0`, `@tauri-apps/api ^2`,
`@tauri-apps/plugin-opener ^2`, `zustand ^5.0.2`, `react-markdown ^9.0.1`,
`remark-gfm ^4.0.0`, `highlight.js ^11.10.0`.
Dev: `vite ^7.0.4`, `typescript ~5.8.3`, `@vitejs/plugin-react ^4.6.0`,
`@tauri-apps/cli ^2`, React type packages.

### Backend (`src-tauri/Cargo.toml`)
`tauri 2`, `tauri-plugin-opener 2`, `serde 1` (`derive`), `serde_json 1`,
`tokio 1` (`full`), `rusqlite 0.32` (`bundled`), `uuid 1` (`v4`), `dirs 5`,
`chrono 0.4` (`serde, clock`), `anyhow 1`. Build-dep: `tauri-build 2`.
Release profile: `strip = true`, `lto = false`.

## App configuration (`tauri.conf.json`)

- **productName** `Yumi`, **identifier** `com.yumi.app`, **version** `0.1.0`.
- **beforeDevCommand** `npm run dev`, **devUrl** `http://localhost:1420`; **beforeBuildCommand** `npm run build`, **frontendDist** `../dist`.
- **Window:** title `Yumi`, 1400×900 (min 900×600), `backgroundColor #000000`, decorations on, resizable.
- **Security:** CSP is `null` (permissive — a personal app).
- **Bundle:** target `nsis` (Windows installer); icons from `src-tauri/icons/`.
- **Capabilities** (`capabilities/default.json`): the `main` window is granted `core:default` and `opener:default`.

The Vite dev server runs on a **strict** port `1420` (HMR on `1421`),
`clearScreen: false` so Rust errors stay visible, and ignores `src-tauri/**` from
its watcher. TypeScript is `strict` with `noUnusedLocals`/`noUnusedParameters`,
`ES2020` target, `react-jsx` transform, `moduleResolution: bundler`.

## Commands

```bash
# install frontend deps
npm install

# build the desktop app (debug, no installer)  → src-tauri/target/debug/yumi.exe
npx tauri build --debug --no-bundle

# live dev (Vite + Tauri)
npm run tauri dev

# frontend only
npm run dev          # vite dev server
npm run build        # tsc && vite build  → dist/
```

### Tests
```bash
# Rust unit tests (parser + registry + analytics) — 25 tests
cd src-tauri && cargo test

# MCP server JSON-RPC test — 12 assertions
cd resources && node tests/mcp_server.test.mjs
```

## Artifacts

| Artifact | Path |
|----------|------|
| Debug exe (embedded UI) | `src-tauri/target/debug/yumi.exe` (~225 MB debug) |
| Frontend build | `dist/` |
| Release bundle | NSIS installer under `src-tauri/target/release/bundle/` (when bundling) |
| Database | `~/.yumi/yumi.db` (created at runtime) |

## Verification gates (last green 2026-06-25)

- `cargo test` → **25/25**.
- `node resources/tests/mcp_server.test.mjs` → **12/12**.
- `tsc && vite build` → **0 errors**.
- `npx tauri build --debug --no-bundle` → builds `yumi.exe`.
- Real-GUI round-trip: prompt → spawn → streamed thinking + markdown answer → finalize → persist. Screenshots in `yumi/docs/evidence/`.

## See also

- [Architecture](architecture.md) — what the build produces and how it runs.
- [Troubleshooting](troubleshooting.md) — build/run failure modes.
- `yumi/README.md` — the short build/run readme.
