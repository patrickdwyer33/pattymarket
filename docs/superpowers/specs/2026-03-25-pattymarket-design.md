# pattymarket Design

**Date:** 2026-03-25
**Project:** pattymarket
**Mission:** Mission 1 — Add captain plugin listing

---

## Goal

Establish pattymarket as both a machine-readable Claude Code plugin registry and a human-browsable catalog, seeded with the captain plugin as its first entry. Migrate the marketplace responsibility away from captain's own repo into pattymarket.

---

## Structure

```
pattymarket/
├── .claude-plugin/
│   └── marketplace.json      ← Claude Code reads this for plugin discovery + update detection
├── README.md                 ← human catalog: one section per plugin, links to source repos
├── CLAUDE.md
├── MISSIONS.md
├── TODO.md
├── GAPS.md
├── IDEAS.md
└── docs/
    └── superpowers/specs/
```

---

## Machine-Readable Registry

**File:** `.claude-plugin/marketplace.json`

Format mirrors the existing captain marketplace.json convention:

```json
{
  "name": "pattymarket",
  "description": "Patrick Dwyer's Claude Code plugins",
  "owner": {
    "name": "Patrick Dwyer",
    "email": "patrick@patrickdwyer.com"
  },
  "plugins": [
    {
      "name": "captain",
      "description": "Task and project management skills for Claude Code. Requires superpowers plugin.",
      "version": "1.0.3",
      "source": {
        "source": "url",
        "url": "https://github.com/patrickdwyer33/captain.git"
      },
      "author": {
        "name": "Patrick Dwyer",
        "email": "patrick@patrickdwyer.com"
      }
    }
  ]
}
```

The `version` field is the trigger Claude Code uses to detect and deliver updates. It must stay in sync with captain's `plugin.json` version on every release.

**User install flow:**
```
claude plugin marketplace add patrickdwyer33/pattymarket
claude plugin install captain
```

---

## Human-Readable Catalog (README.md)

One section per plugin with: description, install snippet, prerequisites, and a link to the source repo for full docs.

```markdown
# pattymarket

Patrick Dwyer's Claude Code plugin marketplace.

## Add this marketplace

claude plugin marketplace add patrickdwyer33/pattymarket

## Plugins

### captain

Mission and project management for Claude Code.

**Install:** `claude plugin install captain`
**Requires:** superpowers plugin
**Docs:** https://github.com/patrickdwyer33/captain
```

The README links out to each plugin's source repo for skills lists, changelogs, and full documentation. No content is duplicated.

---

## Captain Migration

### Changes to captain repo

1. Remove `.claude-plugin/marketplace.json` from captain — marketplace responsibility moves to pattymarket.
2. Add pattymarket as a **git submodule** in captain at `marketplace/`.
3. Remove the existing `pre-push` git hook (version bumping moves to an explicit release script).
4. Add `scripts/release.sh` (see Release Flow below).
5. Update captain's `README.md` install instructions to reference `patrickdwyer33/pattymarket`.

### Changes to pattymarket repo

1. Add `.claude-plugin/marketplace.json` with captain as the first entry.
2. Update `README.md` to serve as the human catalog.

---

## Release Flow

`scripts/release.sh` in the captain repo handles all version bumping and publishing in one shot.

**Usage:**
```bash
./scripts/release.sh           # auto-bumps patch: 1.0.3 → 1.0.4
./scripts/release.sh 2.0.0    # explicit version for major/minor releases
```

**Script behavior:**
1. Determine new version (arg if provided, else bump patch of current version in `plugin.json`)
2. Update version in captain's `.claude-plugin/plugin.json`
3. Update version in captain's `package.json`
4. Update the captain entry's `version` field in `marketplace/` (pattymarket submodule) `.claude-plugin/marketplace.json`
5. Commit + push the submodule (`marketplace/`)
6. Commit the updated submodule ref + version files in captain
7. Push captain

---

## Out of Scope

- No static site / GitHub Pages — README is the human interface
- No per-plugin markdown files — each plugin's own repo is the source of truth
- No GitHub Actions — release script is the release mechanism
