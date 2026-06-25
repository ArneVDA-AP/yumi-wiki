---
title: "Yumi Plugin"
tags: [subsystem, plugin, agents]
---

# Yumi Plugin

*`resources/yumi-plugin/` — a bundled Claude Code plugin (agents, slash commands, and a safety guard hook) that mirrors Yume's orchestration workflow.*


---

## What it is

A standard Claude Code plugin shipped with Yumi and intended to be synced into
`~/.claude` so the spawned `claude` picks up its agents, commands, and guard hook.
It encodes the "architect → implementer → guardian" workflow plus a set of
convenience slash commands.

`.claude-plugin/plugin.json`:
```json
{
  "name": "yumi",
  "description": "commands, agents, and security guard for yumi",
  "version": "3.0.0"
}
```

## Agents (`agents/*.md`)

Four focused subagents, each a markdown file with frontmatter:

| Agent | Role |
|-------|------|
| **yumi-architect** | Plans architecture, decomposes a task into steps, identifies dependencies and risks. Use first when a task has 3+ steps. |
| **yumi-explorer** | Read-only codebase exploration and context gathering — "where is X?", structure, conventions. |
| **yumi-implementer** | Makes small, focused code edits, executing the architect's plan. |
| **yumi-guardian** | Reviews bugs/security/performance after changes; also handles tests, docs, and devops chores. |

## Commands (`commands/*.md`)

Eight slash commands:

| Command | Purpose |
|---------|---------|
| `/commit` | Create a concise, lowercase commit |
| `/compact` | Compact context with preservation hints (task, files, decisions) |
| `/implement` | Implement a planned feature end-to-end |
| `/init` | Initialize context (load memory, scan the project) |
| `/plan` | Create a read-only implementation plan |
| `/research` | Research a topic/problem/resource (read-only) |
| `/review` | Review changes or the codebase (supports a `--fix` flag) |
| `/swarm` | Coordinate a team of agents on a complex task |

## Guard hook (`hooks/guard.md`)

A **`PreToolUse`** hook that blocks dangerous operations before they run. It refuses, among others:

- **Destructive file ops:** `rm -rf`, `dd`, `format`, `shred`, fork bombs.
- **Privilege escalation:** `sudo`, `chmod +s`, edits to `/etc/sudoers`.
- **System control:** `shutdown`, `systemctl disable`, `iptables`, `crontab -r`.
- **Remote code execution:** `curl | sh`, `eval`, `nc -e`, `bash -i >/dev/tcp`.
- **Dangerous git:** `git push -f`, `git reset --hard`, `git branch -D main`.
- **Windows destructive:** `format`, `diskpart`, `bcdedit`.
- **Protected paths / secrets:** access to `.ssh/`, `.aws/`, `.kube/`, `.env`, credential/secret/`.pem`/`id_rsa` files.

## Relationship to Yumi's own subagents

These plugin agents are the ones the **spawned** `claude` can dispatch. They are
distinct from (but parallel to) the agents Yumi-the-app surfaces in its own
tooling. The four roles (architect/explorer/implementer/guardian) match the
operating workflow used throughout the project.

> The installed plugin contains **8 commands** and **4 agents** as listed above
> (verified from `resources/yumi-plugin/`). Some older summaries in `PARITY.md`
> mention "5 commands" — treat the files in `resources/yumi-plugin/commands/` as
> the source of truth.

## See also

- [Overview](overview.md) — the clone's relationship to Yume's plugin.
- [Backend (Rust)](backend-rust.md) — background agents (`agents.rs`) are a separate, app-level feature.
