# Yumi Wiki

Developer documentation for **[Yumi](https://github.com/ArneVDA-AP/yumi)** — a
personal, open desktop GUI for the Claude Code CLI (Tauri 2 + React 19).

📖 **Published site:** <https://arnevda-ap.github.io/yumi-wiki/>

This is a [MkDocs](https://www.mkdocs.org/) + [Material](https://squidfunk.github.io/mkdocs-material/)
site. The source pages live in [`docs/`](docs/); navigation and theme are in
[`mkdocs.yml`](mkdocs.yml).

## Contents

- **Overview** & **Architecture** — what Yumi is and how the pieces fit.
- **Reference** — Backend (Rust), Frontend (React), the IPC contract, the stream pipeline, data & persistence.
- **Subsystems** — the MCP bash server, the thinking proxy, the bundled plugin.
- **Guide** — features & shortcuts, build/run/test, reliability & design, troubleshooting.
- **Glossary** and a **Tags** index.

Every load-bearing claim is grounded in the Yumi source and was adversarially
fact-checked against the code.

## Local development

Requires Python with `mkdocs` and `mkdocs-material`:

```bash
pip install mkdocs mkdocs-material

# live preview at http://127.0.0.1:8000
mkdocs serve

# build the static site into ./site (validated)
mkdocs build --strict
```

## Deploy

The site is published to GitHub Pages from the `gh-pages` branch:

```bash
mkdocs gh-deploy
```

---

© 2026 Arne Van Den Abbeele.
