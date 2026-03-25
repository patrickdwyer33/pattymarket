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

The `version` field is the trigger Claude Code uses to detect and deliver updates. It must stay in sync with captain's `plugin.json` version on every release. The initial value is `1.0.4` — captain's current `plugin.json` version. Note: captain's existing `.claude-plugin/marketplace.json` shows `1.0.0` (it was never kept in sync); that file is being replaced entirely.

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

Perform in this order. The submodule must be verified working before the old marketplace.json is deleted, to avoid a gap where neither file is in place:

1. Add pattymarket as a git submodule in captain at `marketplace/` and verify it clones correctly
2. Write `scripts/release.sh`
3. Verify `scripts/release.sh` works end-to-end in a dry run (or on a test branch) before proceeding
4. Remove `captain/.claude-plugin/marketplace.json`
5. Remove the existing `pre-push` git hook
6. Update captain's `README.md` install instructions
7. Run `./scripts/release.sh` to publish the first release via the new flow

### Changes to captain repo

- Add pattymarket as a **git submodule** at `marketplace/`
- Add `scripts/release.sh` — `scripts/generate.sh` is unrelated and stays untouched
- Remove `.claude-plugin/marketplace.json` (only after submodule is verified)
- Remove `.git/hooks/pre-push` — direct pushes to captain will no longer auto-bump the version; this is intentional. `release.sh` is now the explicit release mechanism. Ad-hoc pushes (WIP, docs fixes) are not releases and should not bump the version.
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
6. In captain: `git add` version files and updated submodule ref, `git commit -m "chore: release vX.Y.Z"`, `git push origin main`

**Partial failure:** If step 5 (submodule push) succeeds but step 6 (captain push) fails, the marketplace will advertise a version not yet present in captain's remote. Recovery: re-run `./scripts/release.sh` with the same version — the script should be idempotent if the version is already set correctly, or the developer manually pushes captain.

---

## Out of Scope

- No static site / GitHub Pages — README is the human interface
- No per-plugin markdown files — each plugin's own repo is the source of truth
- No GitHub Actions — release script is the release mechanism
- Migrating existing users who added `patrickdwyer33/captain` as a marketplace
