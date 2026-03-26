# pattymarket — Claude Code Context

## Project Docs

| File | Purpose |
| ---- | ------- |
| [MISSIONS.md](MISSIONS.md) | Structured mission backlog — numbered missions with phases and sub-items |
| [TODO.md](TODO.md) | Free-form scratchpad for quick notes, reminders, and in-progress thoughts |
| [GAPS.md](GAPS.md) | Known code stubs, unimplemented functions, and placeholder values |
| [IDEAS.md](IDEAS.md) | Long-term ideas and future directions, no commitment implied |

## Overview

pattymarket is a public Claude Code plugin marketplace. It serves two purposes:
1. **Machine-readable registry** — `.claude-plugin/marketplace.json` is read by Claude Code when users run `claude plugin marketplace add patrickdwyer33/pattymarket`
2. **Human-readable catalog** — `README.md` lists each plugin with install instructions and links to source repos

Currently hosts one plugin: **captain** (task/project management skills).

## Commands

No build/test commands — this is a content repo. To release a new version of a plugin, use the release script in that plugin's repo (e.g. `./scripts/release.sh` in captain).

## Key Patterns

- **Plugin entry format** in `marketplace.json`: each plugin needs `name`, `description`, `version`, `source.source: "url"`, `source.url`, and `author`
- **Version field is critical** — Claude Code uses it to detect updates; must stay in sync with the plugin's own `plugin.json`
- **captain is a git submodule** inside the captain repo at `marketplace/` — captain's `scripts/release.sh` updates the version here and pushes both repos atomically
- **README is the only human interface** — no static site; link to each plugin's own repo for full docs, don't duplicate content
