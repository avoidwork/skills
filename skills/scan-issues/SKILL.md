---
name: scan-issues
description: Scans GitHub for open issues labeled 'approved' (excluding those also labeled 'in progress'), creates a git worktree for each, processes them sequentially via fix-issue, and opens one PR at a time with auto-merge. Runs hourly.
license: BSD 3-Clause
compatibility: Requires gh CLI authenticated with the avoidwork/madz repo. Must be run from the project root. Requires git worktree support.
metadata:
  agent: coding
---

# Scan & Fix Issues

An autonomous issue scanner and fixer. Finds approved, unassigned issues and processes them **one at a time** through the `fix-issue` pipeline, each in its own git worktree.

## Workflow

### 1. Determine Repository

Capture the project root and extract the repo from the git remote — never hardcode paths:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel)
GIT_REMOTE=$(git remote get-url origin 2>/dev/null)
if [ -z "$GIT_REMOTE" ]; then
  echo "ERROR: No git remote 'origin' found. Can't determine repository."
  exit 1
fi

if echo "$GIT_REMOTE" | grep -q '^git@'; then
  REPO=$(echo "$GIT_REMOTE" | sed 's/.*@[^:]*:\(.*\).git$/\1/')
elif echo "$GIT_REMOTE" | grep -q 'github\.com'; then
  REPO=$(echo "$GIT_REMOTE" | sed 's/.*github\.com[/:]\(.*\).git$/\1/')
else
  echo "ERROR: Could not parse repository from remote '$GIT_REMOTE'."
  exit 1
fi
```

### 2. Scan for Issues

```bash
gh issue list --state open --label approved --json number,title,url,labels --repo "$REPO"
```

This fetches all open issues that have the `approved` label. **Note:** The `in progress` label check is handled by `fix-issue` in its Step 3, so we don't filter here — let each issue's own validation decide.

If no issues match, report `No approved, unassigned issues found.` and exit cleanly.

### 3. Parse Results

Extract each issue's `number`, `title`, `url`, and `labels` from the JSON array. Sort the remaining issues by `number` ascending (oldest first) to ensure sequential processing.

### 4. Prepare Worktree Directory

Create a dedicated directory for worktrees if it doesn't exist:

```bash
WORKTREES_DIR=".worktrees"
mkdir -p "$WORKTREES_DIR"
```

### 5. Process Issues Sequentially

**Process one issue at a time.** Do NOT spawn subagents or parallelize — each issue must be processed sequentially to avoid label conflicts and race conditions:

For each issue in the sorted list:

1. **Create a worktree** for this issue:
   ```bash
   ISSUE_NUM="<ISSUE_NUMBER>"
   ISSUE_SLUG=$(echo "<ISSUE_TITLE>" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]+/-/g; s/^-//; s/-$//' | head -c 50)
   WORKTREE_BRANCH="scan-issue-${ISSUE_NUM}-${ISSUE_SLUG}"
   WORKTREE_PATH="${WORKTREES_DIR}/issue-${ISSUE_NUM}"

   # Check if branch already exists (from a previous interrupted run)
   if git rev-parse --verify "$WORKTREE_BRANCH" >/dev/null 2>&1; then
     echo "WARNING: Branch '$WORKTREE_BRANCH' already exists. Skipping issue #$ISSUE_NUM."
     continue
   fi

   # Create worktree from main
   git worktree add "$WORKTREE_PATH" "$WORKTREE_BRANCH"

   # Verify worktree creation succeeded
   if [ ! -d "$WORKTREE_PATH" ]; then
     echo "ERROR: Worktree creation failed for issue #$ISSUE_NUM. Skipping."
     continue
   fi
   ```

2. **Change into the worktree:**
   ```bash
   cd "$WORKTREE_PATH"
   ```

3. **Invoke fix-issue** as a chain instruction:
   ```
   fix-issue <ISSUE_NUMBER>
   ```
   This chains through the full `fix-issue` → `create-feature` pipeline inside the worktree.

4. **Wait for completion.** Do not proceed to the next issue until the current one is done.

5. **Extract PR_NUMBER from fix-issue output.** The `fix-issue` chain outputs `PR_NUMBER=<number>`. Parse it:
   ```bash
   PR_NUMBER=$(grep -oP 'PR_NUMBER=\K\d+' <<< "$FIX_ISSUE_OUTPUT" | head -1)
   ```
   If `PR_NUMBER` is empty, the issue was skipped or the pipeline didn't create a PR — log it and continue.

6. **If fix-issue succeeds:** Log the issue number and PR number.

7. **If fix-issue fails:** Log the error, skip the issue, and continue with the next. Never abort the queue over a single failure.

8. **Change back to the project root** before cleanup:
   ```bash
   cd "$PROJECT_ROOT"
   ```

9. **Clean up the worktree** after success or failure:
   ```bash
   git worktree remove "$WORKTREE_PATH" --force 2>/dev/null || true
   rm -rf "$WORKTREE_PATH"
   ```

**Rate limiting:** If GitHub API returns 403 rate limit, wait 60 seconds and retry the current issue once.

**Timeout handling:** Each fix-issue run can take 30-60 minutes (via create-feature). If a run exceeds 60 minutes, report a timeout warning but do not abort — the issue may still be processing.

### 6. PR Auto-Merge

Only **one PR can be open at a time**. The `create-feature` → `commit-push` pipeline within `fix-issue` handles PR creation. After extracting `PR_NUMBER` from the fix-issue output:

- Set it to auto-merge if the repo supports it:
  ```bash
  gh pr merge "$PR_NUMBER" --auto --merge --repo "$REPO"
  ```
- If auto-merge is not available or fails, note it in the report but do not block on it.

### 7. Report

After all issues are processed, produce a structured summary:

```
Scan complete.
✅ Fixed: #42 — "Crash on empty input" (PR #101, auto-merged)
✅ Fixed: #57 — "Missing error boundary" (PR #102, auto-merged)
❌ Skipped: #89 — "Auth timeout race" (fix-issue failed)

Total scanned: 3
Success: 2
Failed: 1
```

## Error Handling

- **No `gh` auth:** Suggest `gh auth login`.
- **API rate limit:** Wait 60 seconds, retry once. If it fails again, report the rate limit and stop.
- **Network error:** Retry once, then report.
- **fix-issue failure on individual issue:** Log it, skip it, continue with the next. Never abort the entire queue over a single failure.
- **Worktree creation failure:** If `git worktree add` fails (e.g., branch already exists), skip the issue and continue.
- **Branch already exists:** If the branch for an issue already exists (from an interrupted run), skip that issue and continue.

## Cron Schedule

This skill is designed to run hourly. When triggered via cron, execute the full workflow above without requiring user input.

---
