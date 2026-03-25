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
    └── superpowers/
        └── specs/            ← design specs written during brainstorming sessions (this file)
```

---

## Machine-Readable Registry

**File:** `.claude-plugin/marketplace.json`

Structurally similar to plugin-repo marketplace.json files. The top-level `name` and `description` describe the marketplace itself; the `plugins` array contains one entry per plugin. The `source.source: "url"` field is a Claude Code type discriminant required by the plugin schema.

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

The `version` field in the marketplace entry must stay in sync with captain's own `plugin.json` version on every release. Claude Code compares the installed plugin's `plugin.json` version against this field to detect updates — both must be bumped together. Initial value: `1.0.4` (captain's current `plugin.json` version). Note: captain's existing `.claude-plugin/marketplace.json` shows `1.0.0` (it was never kept in sync); that file is being replaced entirely.

**User install flow** (`marketplace add` is a one-time global operation; `plugin install` is per-user):
```
claude plugin marketplace add patrickdwyer33/pattymarket
claude plugin install captain
```

**Note:** Users who previously added `patrickdwyer33/captain` as a marketplace will need to re-add `patrickdwyer33/pattymarket`. Migrating existing users is out of scope.

---

## Human-Readable Catalog (README.md)

One section per plugin with: description, install snippet, prerequisites, and a link to the source repo. Full docs (skill lists, changelogs, configuration) live in each plugin's own repo — the README does not reproduce them.

---

## Captain Migration

### Migration sequence

Perform in this order. The pre-push hook must be removed first — if it fires during any `git push` in the migration (including inside `release.sh`), it will auto-bump `plugin.json` without updating the marketplace submodule, causing a version mismatch. The submodule must also be verified working before the old `marketplace.json` is deleted.

1. Remove the existing `pre-push` git hook: `rm captain/.git/hooks/pre-push` (local filesystem delete — no git commit needed; `.git/hooks/` is not tracked)
2. In pattymarket: create `.claude-plugin/marketplace.json` with captain as the first entry, rewrite `README.md` as the human catalog, commit, and push to remote — the submodule add in the next step clones from the remote, so these changes must be live first
3. Add pattymarket as a git submodule in captain:
   ```
   git submodule add https://github.com/patrickdwyer33/pattymarket.git marketplace
   ```
   Verify the submodule clones and `marketplace/.claude-plugin/marketplace.json` exists.
4. Write `scripts/release.sh`
5. Remove `captain/.claude-plugin/marketplace.json` (only after submodule is verified)
6. Update captain's `README.md` — replace the Installation section and Auto-updates section to reference `patrickdwyer33/pattymarket` (see "User install flow" section for the exact install snippet)
7. Run `./scripts/release.sh` to publish the first release via the new flow

### Changes to captain repo

- Add pattymarket as a **git submodule** at `marketplace/`
- Add `scripts/release.sh` — `scripts/generate.sh` is unrelated and stays untouched
- Remove `.claude-plugin/marketplace.json` (only after submodule is verified)
- Remove `.git/hooks/pre-push` (local filesystem delete, no git commit) — direct pushes to captain will no longer auto-bump the version; this is intentional. `release.sh` is now the explicit release mechanism. Ad-hoc pushes (WIP, docs fixes) are not releases and should not bump the version.
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
6. In captain: `git add .claude-plugin/plugin.json package.json marketplace`, `git commit -m "chore: release vX.Y.Z"`, `git push origin main`

**Partial failure:** If the submodule push succeeds but the captain push fails, run `git push origin main` in the captain repo. If the remote has diverged, resolve conflicts first.

---

## Out of Scope

- No static site / GitHub Pages — README is the human interface
- No per-plugin markdown files — each plugin's own repo is the source of truth
- No GitHub Actions — release script is the release mechanism
- Migrating existing users who added `patrickdwyer33/captain` as a marketplace
