# pattymarket Mission 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish pattymarket as a Claude Code plugin marketplace registry and migrate captain's marketplace hosting from the captain repo into pattymarket, with an explicit release script replacing the old pre-push hook.

**Architecture:** pattymarket holds `.claude-plugin/marketplace.json` (machine-readable registry) and `README.md` (human catalog). Captain adds pattymarket as a git submodule at `marketplace/`. A new `scripts/release.sh` in captain bumps versions in both repos and pushes both in one shot, replacing the old pre-push hook.

**Tech Stack:** Bash, Python 3 (json module), git submodules, GitHub

**Spec:** `docs/superpowers/specs/2026-03-25-pattymarket-design.md`

---

## File Map

**pattymarket repo:**
- Create: `pattymarket/.claude-plugin/marketplace.json` — machine-readable plugin registry
- Modify: `pattymarket/README.md` — human-browsable catalog replacing the current stub

**captain repo:**
- Delete (filesystem, not git): `captain/.git/hooks/pre-push`
- Delete (git-tracked): `captain/.claude-plugin/marketplace.json`
- Create: `captain/scripts/release.sh` — explicit release script
- Add submodule: `captain/marketplace/` → pattymarket remote
- Modify: `captain/README.md` — update install instructions to reference pattymarket

---

## Task 1: Create pattymarket registry and catalog

**Repos:** pattymarket

**Files:**
- Create: `pattymarket/.claude-plugin/marketplace.json`
- Modify: `pattymarket/README.md`

- [ ] **Step 1: Create `.claude-plugin/` directory and `marketplace.json`**

Create `pattymarket/.claude-plugin/marketplace.json` with this exact content (version `1.0.4` matches captain's current `plugin.json`):

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

- [ ] **Step 2: Verify JSON is valid**

```bash
python3 -c "import json; json.load(open('pattymarket/.claude-plugin/marketplace.json')); print('valid')"
```

Expected: `valid`

- [ ] **Step 3: Rewrite `pattymarket/README.md` as the human catalog**

Replace the current stub README with:

```markdown
# pattymarket

Patrick Dwyer's Claude Code plugin marketplace.

## Add this marketplace

```
claude plugin marketplace add patrickdwyer33/pattymarket
```

## Plugins

### captain

Mission and project management skills for Claude Code.

**Install:**
```
claude plugin install captain
```

**Requires:** [superpowers](https://github.com/obra/superpowers) plugin installed first

**Docs & Skills:** [github.com/patrickdwyer33/captain](https://github.com/patrickdwyer33/captain)
```

- [ ] **Step 4: Commit and push pattymarket changes**

These must be pushed to the remote before adding pattymarket as a submodule in captain — `git submodule add` clones from the remote URL.

```bash
cd pattymarket
git add .claude-plugin/marketplace.json README.md
git commit -m "feat: add marketplace registry and human catalog"
git push origin main
```

Expected: push succeeds, both files appear in the GitHub repo.

---

## Task 2: Remove captain's pre-push hook

**Repo:** captain

The hook must be removed before any other captain changes that involve `git push`. If it fires, it will auto-bump `plugin.json` without updating the marketplace submodule, causing a version mismatch.

**Files:**
- Delete (filesystem only, not git-tracked): `captain/.git/hooks/pre-push`

- [ ] **Step 1: Delete the hook**

```bash
rm captain/.git/hooks/pre-push
```

- [ ] **Step 2: Verify it is gone**

```bash
ls captain/.git/hooks/pre-push 2>&1
```

Expected: `No such file or directory`

No commit needed — `.git/hooks/` is not tracked by git.

---

## Task 3: Add pattymarket as a git submodule in captain

**Repo:** captain

**Files:**
- Add submodule: `captain/marketplace/` (new directory, tracked as submodule)
- Created by git: `captain/.gitmodules`

- [ ] **Step 1: Add the submodule**

```bash
cd captain
git submodule add https://github.com/patrickdwyer33/pattymarket.git marketplace
```

Expected: git clones pattymarket into `marketplace/` and creates `.gitmodules`.

- [ ] **Step 2: Verify submodule contents**

```bash
cat captain/marketplace/.claude-plugin/marketplace.json
```

Expected: the marketplace.json created in Task 1 — with captain listed at version `1.0.4`.

- [ ] **Step 3: Commit the submodule**

```bash
cd captain
git add .gitmodules marketplace
git commit -m "chore: add pattymarket as marketplace submodule"
git push origin main
```

Expected: push succeeds. (The hook is already removed, so no auto-bump fires.)

---

## Task 4: Write `scripts/release.sh`

**Repo:** captain

**Files:**
- Create: `captain/scripts/release.sh`

- [ ] **Step 1: Write the script**

Create `captain/scripts/release.sh`:

```bash
#!/usr/bin/env bash
# release.sh — bump captain version in both captain and pattymarket, then push both.
#
# Usage:
#   ./scripts/release.sh           # auto-bumps patch version (e.g. 1.0.4 → 1.0.5)
#   ./scripts/release.sh 2.0.0    # explicit version for major/minor releases

set -e

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
CAPTAIN_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"
PLUGIN_JSON="$CAPTAIN_ROOT/.claude-plugin/plugin.json"
PACKAGE_JSON="$CAPTAIN_ROOT/package.json"
MARKETPLACE_JSON="$CAPTAIN_ROOT/marketplace/.claude-plugin/marketplace.json"

# ── Preflight ────────────────────────────────────────────────────────────────

if [ ! -f "$MARKETPLACE_JSON" ]; then
  echo "error: marketplace submodule not initialized. Run: git submodule update --init" >&2
  exit 1
fi

# ── Determine new version ────────────────────────────────────────────────────

current=$(python3 -c "import json; print(json.load(open('$PLUGIN_JSON'))['version'])")

if [ -n "${1:-}" ]; then
  new="$1"
else
  new=$(python3 -c "
parts = '$current'.split('.')
parts[2] = str(int(parts[2]) + 1)
print('.'.join(parts))
")
fi

echo "release: $current → $new"

# ── Update captain version files ─────────────────────────────────────────────

python3 - "$PLUGIN_JSON" "$PACKAGE_JSON" "$new" <<'PYEOF'
import sys, json
plugin_json, package_json, new_version = sys.argv[1], sys.argv[2], sys.argv[3]
for path in [plugin_json, package_json]:
    with open(path) as f:
        d = json.load(f)
    d['version'] = new_version
    with open(path, 'w') as f:
        json.dump(d, f, indent=2)
        f.write('\n')
PYEOF

# ── Update marketplace submodule ─────────────────────────────────────────────

python3 - "$MARKETPLACE_JSON" "$new" <<'PYEOF'
import sys, json
marketplace_json, new_version = sys.argv[1], sys.argv[2]
with open(marketplace_json) as f:
    d = json.load(f)
for plugin in d['plugins']:
    if plugin['name'] == 'captain':
        plugin['version'] = new_version
with open(marketplace_json, 'w') as f:
    json.dump(d, f, indent=2)
    f.write('\n')
PYEOF

# ── Commit and push submodule ────────────────────────────────────────────────

cd "$CAPTAIN_ROOT/marketplace"
git add .claude-plugin/marketplace.json
git commit -m "chore: bump captain to v$new"
git push origin main

# ── Commit and push captain ──────────────────────────────────────────────────

cd "$CAPTAIN_ROOT"
git add .claude-plugin/plugin.json package.json marketplace
git commit -m "chore: release v$new"
git push origin main

echo "release: v$new published ✓"
```

- [ ] **Step 2: Make it executable**

```bash
chmod +x captain/scripts/release.sh
```

- [ ] **Step 3: Dry-run preflight check (without pushing)**

Verify the script parses and the preflight passes:

```bash
cd captain
bash -n scripts/release.sh
echo "syntax ok"
python3 -c "import json; print(json.load(open('.claude-plugin/plugin.json'))['version'])"
```

Expected: `syntax ok`, then the current version (e.g. `1.0.4`).

- [ ] **Step 4: Commit `release.sh`**

```bash
cd captain
git add scripts/release.sh
git commit -m "feat: add release.sh to replace pre-push hook"
git push origin main
```

---

## Task 5: Remove captain's old `marketplace.json`

**Repo:** captain

The submodule is verified working (Task 3). Now remove the stale file.

**Files:**
- Delete: `captain/.claude-plugin/marketplace.json`

- [ ] **Step 1: Delete the file**

```bash
cd captain
git rm .claude-plugin/marketplace.json
```

- [ ] **Step 2: Commit the deletion**

```bash
cd captain
git commit -m "chore: remove marketplace.json (moved to pattymarket submodule)"
git push origin main
```

- [ ] **Step 3: Verify `.claude-plugin/` still has `plugin.json`**

```bash
ls captain/.claude-plugin/
```

Expected: only `plugin.json` remains.

---

## Task 6: Update captain's `README.md`

**Repo:** captain

Update both the **Installation** and **Auto-updates** sections to reference `patrickdwyer33/pattymarket` instead of `patrickdwyer33/captain`.

**Files:**
- Modify: `captain/README.md`

- [ ] **Step 1: Update the Installation section**

Replace:
```
claude plugin marketplace add patrickdwyer33/captain
claude plugin install captain
```
With:
```
claude plugin marketplace add patrickdwyer33/pattymarket
claude plugin install captain
```

- [ ] **Step 2: Update the Auto-updates section**

The Auto-updates section currently says:
> Go to **Marketplaces** → **captain**

Change `captain` to `pattymarket`:
> Go to **Marketplaces** → **pattymarket**

- [ ] **Step 3: Commit**

```bash
cd captain
git add README.md
git commit -m "docs: update install instructions to use pattymarket"
git push origin main
```

---

## Task 7: Run first release via `release.sh`

**Repos:** captain + pattymarket (via submodule)

This is the end-to-end verification that the release flow works correctly.

- [ ] **Step 1: Run release.sh with an explicit version to keep it clean**

Use the current version explicitly (no bump) to verify the flow, then let future releases auto-bump. Since version `1.0.4` is already set correctly in both files, do a patch bump to `1.0.5` as the first real release through the new flow:

```bash
cd captain
./scripts/release.sh 1.0.5
```

Expected output:
```
release: 1.0.4 → 1.0.5
chore: bump captain to v1.0.5
...push to pattymarket...
chore: release v1.0.5
...push to captain...
release: v1.0.5 published ✓
```

- [ ] **Step 2: Verify pattymarket submodule was updated**

```bash
cat captain/marketplace/.claude-plugin/marketplace.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(next(p['version'] for p in d['plugins'] if p['name']=='captain'))"
```

Expected: `1.0.5`

- [ ] **Step 3: Verify captain plugin.json was updated**

```bash
python3 -c "import json; print(json.load(open('captain/.claude-plugin/plugin.json'))['version'])"
```

Expected: `1.0.5`

- [ ] **Step 4: Verify both repos are clean**

```bash
cd captain && git status
cd captain/marketplace && git status
```

Expected: both show `nothing to commit, working tree clean`

- [ ] **Step 5: Invoke `captain:finish-mission`**

All implementation is complete. Invoke the `captain:finish-mission` skill to handle documentation updates, move Mission 1 to Completed, and clean up.
