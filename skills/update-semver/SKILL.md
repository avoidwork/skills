---
name: update-semver
description: Audits the delta between HEAD and the tag matching the current package.json version, decides if it's a major/minor/patch bump, updates package.json, runs npm i and npm run changelog, then triggers commit-push, enables auto-merge on the PR, and announces the new version. Does NOT create a git tag — that is handled by the separate git-tag skill after the PR merges.
license: BSD 3-Clause
compatibility: Requires Node.js 24+, npm, git CLI with remote access, and gh CLI for PR management. Must be run from the project root directory containing package.json.
metadata:
  agent: coding
---

# Update Semver

You are the release conductor. The version number is the promise you make to the world. Treat it with precision. Follow these steps in order.

## Step 1: Create a Release Branch

Before touching any files, create a deterministic branch for this release. This keeps `main` pristine and gives the PR a clear home.

```bash
TIMESTAMP=$(date -u +"%Y-%m-%dT%H-%M-%SZ")
BRANCH="chore/update-semver-${TIMESTAMP}"
git checkout -b "$BRANCH"
echo "Branch: $BRANCH"
```

The timestamp uses ISO 8601 format in UTC. Colons are replaced with dashes (`%H-%M-%S` instead of `%H:%M:%S`) to ensure filesystem compatibility.

## Step 2: Read the Current Version

Extract the version from `package.json` using `jq` for precision:

```bash
CURRENT_VERSION=$(jq -r '.version' package.json)
echo "CURRENT_VERSION=$CURRENT_VERSION"
```

If `jq` is unavailable, fall back to `grep`:

```bash
CURRENT_VERSION=$(grep '"version"' package.json | head -1 | sed 's/.*: *"\([^"]*\)".*/\1/')
echo "CURRENT_VERSION=$CURRENT_VERSION"
```

## Step 3: Find the Last Tag

The tag to compare against is the most recent tag on the current branch — the actual last release.

```bash
TARGET_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
echo "TARGET_TAG=$TARGET_TAG"
```

If no tag exists, set `TARGET_TAG=""` and use the entire history as the delta.

## Step 4: Audit the Delta

Collect commit messages between the target tag and HEAD:

```bash
if [ -n "$TARGET_TAG" ]; then
  git log --pretty=format:"%s" "${TARGET_TAG}..HEAD"
else
  git log --pretty=format:"%s" --max-count=50
fi
```

## Step 5: Decide the Bump

Analyze the commit messages using **Conventional Commits** semantics. Apply the **highest** bump found:

| Commit prefix | Bump |
|---|---|
| `feat:` | **minor** |
| `fix:` | **patch** |
| `BREAKING CHANGE` or `!` after type/scope | **major** |
| `perf:` | **minor** (performance improvements are treated as features) |
| `refactor:`, `chore:`, `docs:`, `style:`, `test:` | **no bump** |

**Decision rules:**
- If **any** commit is a breaking change → **major**
- If **any** commit is a `feat:` or `perf:` → **minor**
- If only `fix:` (and no feat/breaking) → **patch**
- If no version-relevant commits but commits exist → **patch** (maintenance release)
- If no commits at all → **abort** (nothing to release)

Compute the new version:
- **major**: increment major, reset minor and patch to 0
- **minor**: increment minor, reset patch to 0
- **patch**: increment patch

Print the decision:
```
Bump: <type> (<reason>)
Version: <CURRENT_VERSION> → <NEW_VERSION>
```

## Step 6: Update package.json

Replace the version in `package.json` with the new semver value. Use `jq` for precision:

```bash
jq --arg v "<NEW_VERSION>" '.version = $v' package.json > package.json.tmp && mv package.json.tmp package.json
```

If `jq` is unavailable, fall back to `sed` — but verify there's only one version field:

```bash
VERSION_COUNT=$(grep -c '"version"' package.json)
if [ "$VERSION_COUNT" -ne 1 ]; then
  echo "ERROR: Found $VERSION_COUNT version fields in package.json. Cannot proceed safely."
  exit 1
fi
sed -i 's/"version": "[^"]*"/"version": "<NEW_VERSION>"/' package.json
```

Verify the change:
```bash
jq -r '.version' package.json
```

## Step 7: Install, Build, and Generate Changelog

```bash
npm i --ignore-scripts
if [ $? -ne 0 ]; then
  echo "ERROR: npm install failed. Aborting."
  exit 1
fi

# Run build if the script exists — MUST succeed if present
if jq -e '.scripts.build' package.json > /dev/null 2>&1; then
  echo "Build script detected. Running build..."
  npm run build
  if [ $? -ne 0 ]; then
    echo "ERROR: Build script failed. Aborting — cannot release a broken build."
    exit 1
  fi
  echo "Build succeeded."
fi

npm run changelog
```

This installs dependencies (ensuring lockfile is current), runs `build` if the project has one (**must succeed** if present — a broken build is a broken release), and generates an updated `CHANGELOG.md` using `auto-changelog`. The `--ignore-scripts` flag prevents postinstall scripts from running during the version bump. If `npm i` fails, abort immediately — do not proceed to commit a broken state.

## Step 8: Trigger commit-push

Delegating version release to `commit-push`, which will:
1. Scan for AGENTS.md to read project rules
2. Stage **all** files (`git add -A` — nothing left behind)
3. Commit with a conventional commit message (e.g., `chore: release v1.3.8`)
4. Push to the remote — **asks user for explicit approval first** (AGENTS.md §1.3)
5. Open a PR targeting `main`

Invoke the `commit-push` skill.

**After `commit-push` completes, continue to Step 9.** Do not stop or wait for further input — the pipeline proceeds automatically.

## Step 9: Enable Auto-Merge

After the PR is created, enable auto-merge on it with **squash** merge. Extract the PR number from the `commit-push` output (look for `PR_NUMBER=<number>` printed as structured output), then enable auto-merge:

```bash
gh pr merge "$PR_NUMBER" --auto --squash
echo "Auto-merge: ENABLED (squash)"
```

If auto-merge fails (e.g., repo settings don't support it), report the error and advise manual merge.

## Step 10: Announce the New Version

Print a final announcement:

```
Version <NEW_VERSION> is live.

Branch: <branch-name>
PR: <PR_URL>
Bump: <type> (<count> commits analyzed)
From: <TARGET_TAG or "initial">
Changelog: updated

The world gets a new version.
```

## Examples

```
User: update-semver

Agent: Creating release branch: chore/update-semver-2026-08-15T10-30-00Z
Current version: 1.34.0
Last tag: 1.34.0
Delta: 12 commits

Bump: minor (3 feat: commits found)
Version: 1.34.0 → 1.35.0

Updated package.json to 1.35.0
npm install: success
Build: success
Changelog: updated

commit-push: PR #456 created
Auto-merge: ENABLED (squash)

Version 1.35.0 is live.

Branch: chore/update-semver-2026-08-15T10-30-00Z
PR: https://github.com/avoidwork/madz/pull/456
Bump: minor (3 commits analyzed)
From: 1.34.0
Changelog: updated

The world gets a new version.
```

## Gotchas

---
