---
name: fix-issue
description: Accepts a GitHub issue ID, validates approval via labels, categorizes the work, chains to /create-feature for full implementation, and comments on the issue linking to the PR.
license: BSD 3-Clause
compatibility: Requires gh CLI authenticated with the repo. Must be run from the project root.
metadata:
  agent: coding
---

# Fix Issue

> **⚠️ EXECUTION RULE:** Every code block in this skill is a shell command to **execute**. Do not print them as text, explain them, or treat them as examples — run them directly.

Accepts a GitHub issue ID, validates it's approved for work, categorizes the effort, and chains to `/create-feature` for full implementation.

## Pipeline

This is an automatic pipeline. Execute each step in order without stopping for confirmation or narration. When you reach step 6, chain to `/create-feature` and wait for it to complete. Do not describe the chain — execute it.

### Step 1: Parse Input

The issue ID is provided in the command context (the text after `/fix-issue`). **Use it directly.** Do not ask the user for it. It may be provided as:
- A bare number: `42`
- With a prefix: `#42` or `issue-42`

Strip any non-numeric characters and extract the numeric ID. If no ID was provided in the command, *then* ask the user for it.

```bash
# Extract issue number from chain context (text after /fix-issue)
# The chain context is passed via CHAIN_CONTEXT env var or as $1
ISSUE_NUM="${CHAIN_CONTEXT:-$1}"
# Strip non-numeric characters (handles #42, issue-42, etc.)
ISSUE_NUM=$(echo "$ISSUE_NUM" | grep -oE '[0-9]+' | head -1)
if [ -z "$ISSUE_NUM" ]; then
  echo "ERROR: No issue number provided. Usage: fix-issue <number>"
  exit 1
fi
echo "ISSUE_NUM=$ISSUE_NUM"
```

### Step 2: Fetch Issue Details

Determine the target repository dynamically from the git remote. Never hardcode a repo name:

```bash
# Extract owner/repo from git remote URL — handles both HTTPS and SSH
GIT_REMOTE=$(git remote get-url origin 2>/dev/null)
if [ -z "$GIT_REMOTE" ]; then
  echo "ERROR: No git remote 'origin' found. Can't determine repository."
  exit 1
fi

# SSH format: git@github.com:owner/repo.git
if echo "$GIT_REMOTE" | grep -q '^git@'; then
  REPO=$(echo "$GIT_REMOTE" | sed 's/.*@[^:]*:\(.*\).git$/\1/')
# HTTPS format: https://github.com/owner/repo.git
elif echo "$GIT_REMOTE" | grep -q 'github\.com'; then
  REPO=$(echo "$GIT_REMOTE" | sed 's/.*github\.com[/:]\(.*\).git$/\1/')
else
  echo "ERROR: Could not parse repository from remote '$GIT_REMOTE'."
  exit 1
fi
```

```bash
ISSUE_DETAILS=$(gh issue view "$ISSUE_NUM" --json title,body,labels,state,url --repo "$REPO")
echo "$ISSUE_DETAILS"
```

**Check if issue is closed:** After fetching, inspect the `state` field. If `state` is `"CLOSED"`, stop and report: "Issue #$ISSUE_NUM is closed. No action taken."

If the issue does not exist or is inaccessible, report the error and stop.

### Step 3: Validate Approval

Inspect the `labels` array from the issue JSON.

- **If `in progress` label is present:** Stop. Inform the user that the issue is already being worked on. Provide the issue URL.
- **If `approved` label is present:** Proceed.
- **If neither `in progress` nor `approved` is present:** Stop. Inform the user that the issue is not approved for work. Provide the issue URL and suggest they add the label before proceeding.

```
Issue #<ID> is not approved for work.
Label "approved" not found.
URL: <url>
Please add the "approved" label before requesting implementation.
```

```
Issue #<ID> is already being worked on.
Label "in progress" found.
URL: <url>
No action taken.
```

### Step 4: Set In-Progress Label

Before any work begins, mark the issue as being worked on:

```bash
gh issue edit "$ISSUE_NUM" --add-label "in progress" --repo "$REPO"

# Verify the label was added successfully
LABEL_CHECK=$(gh issue view "$ISSUE_NUM" --json labels --jq '.[].name' --repo "$REPO" 2>/dev/null || true)
if ! echo "$LABEL_CHECK" | grep -q '"in progress"'; then
  echo "ERROR: Failed to add 'in progress' label to issue #$ISSUE_NUM."
  exit 1
fi
echo "Label 'in progress' verified on issue #$ISSUE_NUM."
```

This ensures the issue won't be picked up again by `scan-issues` and clearly marks it as being worked on. (Note: `--add-label` is idempotent — adding an already-present label is a no-op.)

### Step 5: Categorize the Work

Inspect the labels for a categorization tag. Priority order:

1. `bug` — defect or unexpected behavior
2. `feature` — new capability
3. `chore` — maintenance, cleanup, refactoring
4. `docs` — documentation changes
5. `test` — test coverage improvements

If no categorization label is found, **default to `bug`**.

### Step 6: Run create-feature

**This is not a description. Execute it.**

Map the categorized type to a conventional commit prefix for the branch name:

| Category | Branch Type |
|----------|-------------|
| `bug`    | `fix`       |
| `feature`| `feat`      |
| `chore`  | `chore`     |
| `docs`   | `docs`      |
| `test`   | `test`      |

Invoke the `create-feature` skill as a chain instruction (text delegation), passing the branch type via `BRANCH_TYPE`. Keep the instruction under 300 characters.

**Wait for the invocation to complete.** Do not proceed to Step 7 until the feature implementation, testing, PR creation, and archival are all done. The `create-feature` skill handles:

- Proposal generation
- Spec design
- Task breakdown
- Todo item creation
- Implementation
- Testing
- PR creation
- Archival
- Prints structured output: `PR_NUMBER=<number>` and `PR_URL=<url>`

**Read the `PR_NUMBER` from the structured output printed by `create-feature`.** Do not attempt to grep a shell variable — the value is in the conversation history.

**Do NOT create todo items in this skill.** Todo management is the sole responsibility of `create-feature`.

**If create-feature fails:** Log the error, skip Step 7 (commenting), and report the failure in the final summary. Do not attempt to recover.

### Step 7: Comment on Issue

**Skip this step if create-feature failed.** If create-feature failed (per Step 6 error handling), do not attempt to comment — the PR was never created.

Use the `PR_NUMBER` from the structured output printed by `create-feature` in Step 6. Do not attempt to extract it from a shell variable — it is in the conversation history.

```bash
gh issue comment "$ISSUE_NUM" --repo "$REPO" --body "Fixed in #${PR_NUMBER}."

# Verify the comment was posted successfully
COMMENT_CHECK=$(gh issue view "$ISSUE_NUM" --json comments --jq '.comments[-1].body' --repo "$REPO" 2>/dev/null || true)
if ! echo "$COMMENT_CHECK" | grep -q "Fixed in #${PR_NUMBER}"; then
  echo "WARNING: Failed to verify comment on issue #$ISSUE_NUM."
fi
```

If the comment fails, note it in the final report but do not stop.

## Error Handling

- **Issue not found:** Report the error clearly with the provided ID.
- **Not authenticated:** Suggest running `gh auth login`.
- **Network error:** Retry once, then report the failure.
- **`/create-feature` failure:** Report the specific failure and continue with what was accomplished. Never leave the queue half-done.

## Gotchas

- **Approval is mandatory.** The skill stops immediately if the issue lacks the `approved` label. Do not attempt to bypass this check.
- **`in progress` label prevents re-processing.** If the label is present, the skill reports and exits — the issue is already being worked on.
- **PR number comes from create-feature output.** The `create-feature` skill prints `PR_NUMBER=<number>` as structured output. Read it from the conversation history.
- **Todo management is delegated.** The `create-feature` skill owns todo items. Do not create todos in this skill.

## Example

```
User: fix-issue 123

Agent: Fetching issue #123...
  Title: "Crash on empty input"
  Labels: approved, bug
  State: open

  Issue is approved. Category: bug.

  Invoking create-feature...

  [waits for create-feature to complete]

  Commenting on issue #123 → Fixed in #126.
```