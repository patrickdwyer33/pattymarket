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

Structurally similar to plugin-repo marketplace.json files, but the top-level `name` and `description` describe the marketplace itself, not any single plugin:

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
      "version": "1.0.4",
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

The `version` field is the trigger Claude Code uses to detect and deliver updates. It must stay in sync with captain's `plugin.json` version on every release. The initial value is `1.0.4` (captain's current version at time of migration).

**User install flow:**
```
claude plugin marketplace add patrickdwyer33/pattymarket
claude plugin install captain
```

**Note:** Users who previously added `patrickdwyer33/captain` as a marketplace will need to re-add `patrickdwyer33/pattymarket` to receive future updates. Migrating existing users is out of scope for this mission.

---

## Human-Readable Catalog (README.md)

One section per plugin with: description, install snippet, prerequisites, and a link to the source repo for full docs. No content is duplicated from source repos — the README links out to each plugin's own repository for skills lists, changelogs, and full documentation.

---

## Captain Migration

### Migration sequence

Perform changes in this order to avoid a window where both marketplace.json files exist simultaneously:

1. Add pattymarket as a git submodule in captain at `marketplace/`
2. Write `scripts/release.sh` (see Release Flow)
3. Remove `captain/.claude-plugin/marketplace.json`
4. Remove the existing `pre-push` git hook
5. Update captain's `README.md` install instructions
6. Run `./scripts/release.sh` to publish the first release via the new flow

### Changes to captain repo

- Add pattymarket as a **git submodule** at `marketplace/`
- Add `scripts/release.sh` (see below) — `scripts/generate.sh` is unrelated and stays untouched
- Remove `.claude-plugin/marketplace.json` (marketplace responsibility moves to pattymarket)
- Remove `.git/hooks/pre-push` (version bumping moves to `release.sh`; direct pushes to captain will no longer auto-bump the version — this is intentional, version management is now explicit)
- Update `README.md`: change install instructions from `claude plugin marketplace add patrickdwyer33/captain` to `claude plugin marketplace add patrickdwyer33/pattymarket`

### Changes to pattymarket repo

- Add `.claude-plugin/marketplace.json` with captain as the first entry
- Rewrite `README.md` to serve as the human catalog

---

## Release Flow

`scripts/release.sh` in the captain repo handles all version bumping and publishing.

**Usage:**
```bash
./scripts/release.sh           # auto-bumps patch: 1.0.4 → 1.0.5
./scripts/release.sh 2.0.0    # explicit version for major/minor releases
```

**Script behavior:**
1. Determine new version (arg if provided, else bump patch of current version in `plugin.json`)
2. Update version in captain's `.claude-plugin/plugin.json`
3. Update version in captain's `package.json`
4. Update the captain entry's `version` field in `marketplace/.claude-plugin/marketplace.json` (pattymarket submodule)
5. In the submodule (`marketplace/`): `git add`, `git commit -m "chore: bump captain to vX.Y.Z"`, `git push origin main`
6. In captain: `git add` the version files and updated submodule ref, `git commit -m "chore: release vX.Y.Z"`, `git push origin main`

---

## Out of Scope

- No static site / GitHub Pages — README is the human interface
- No per-plugin markdown files — each plugin's own repo is the source of truth
- No GitHub Actions — release script is the release mechanism
- Migrating existing users who added `patrickdwyer33/captain` as a marketplace
