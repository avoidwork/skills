---
name: create-feature
description: Orchestrates a complete feature lifecycle: receive goals, synthesize specs, propose via OpenSpec, commit specs to PR, apply tasks, commit implementation, archive, update PR, and post audit results.
license: BSD 3-Clause
compatibility: Requires Node.js 24+, npm, git with remote access, gh CLI, openspec CLI, and a project root with openspec/ directory. When invoked as a sub-agent, requires a 30–60 minute timeout — the full pipeline (spec → PR → implement → archive → update PR) can take that long.
metadata:
  agent: coding
---

# Create Feature

Orchestrate a complete feature lifecycle from raw goals to shipped code. This is the full pipeline — synthesis, specification, implementation, verification, delivery, and archive.

## Prerequisites

Before starting, ensure:
- You are in the project root directory (contains `package.json`, `openspec/`)
- `gh` CLI is authenticated and configured
- `openspec` CLI is available
- The current branch is `main` (or will be checked out at step 0)

## Input Parsing

The input may include a `BRANCH_TYPE` prefix (e.g., `BRANCH_TYPE=fix create-feature ...`). Extract it if present:

```bash
# Extract BRANCH_TYPE from input if provided (e.g., "BRANCH_TYPE=fix create-feature ...")
if echo "$INPUT" | grep -qP '^BRANCH_TYPE=\w+\s+create-feature'; then
  BRANCH_TYPE=$(echo "$INPUT" | grep -oP 'BRANCH_TYPE=\K\w+')
  # Strip the prefix for goal parsing
  INPUT=$(echo "$INPUT" | sed 's/^BRANCH_TYPE=\w\+\s\+create-feature\s\+//')
fi
```

## Input

The user provides a list of goals/features in any format (natural language, JSON, markdown list, etc.). Parse and normalize them into a clean array of goal strings.

**Capture the raw input** — store it in a variable for later use:

```bash
# The agent must capture whatever the user provided as input.
# If invoked via chain instruction, the input is the text after "create-feature".
# Store it for the BRANCH_TYPE extraction block below.
INPUT="<USER_PROVIDED_INPUT>"
```

**Example input:**
> Add a new tool that can summarize web pages, and improve the TUI memory panel to show retention stats.

**Normalized goals:**
1. Add a new tool that can summarize web pages
2. Improve the TUI memory panel to show retention stats

---

## Step 0: Ensure Clean State

**Create the memory directory if it does not exist:**

```bash
mkdir -p memory
```

**Determine the target repository dynamically from the git remote (needed for later gh commands):**

```bash
GIT_REMOTE=$(git remote get-url origin 2>/dev/null)
if echo "$GIT_REMOTE" | grep -q '^git@'; then
  GH_REPO=$(echo "$GIT_REMOTE" | sed 's/.*@[^:]*:\(.*\).git$/\1/')
elif echo "$GIT_REMOTE" | grep -q 'github\.com'; then
  GH_REPO=$(echo "$GIT_REMOTE" | sed 's/.*github\.com[/:]\(.*\).git$/\1/')
fi
```

```bash
# Only checkout main if not already on it
if [ "$(git branch --show-current)" != "main" ]; then
  # Verify main exists on remote before pulling
  REMOTE_MAIN=$(git ls-remote --heads origin main 2>/dev/null)
  if [ -n "$REMOTE_MAIN" ]; then
    git fetch origin main
    git checkout main
    git pull origin main
  else
    echo "WARNING: No remote branch 'main' found. Current branch '$(git branch --show-current)' will be used as-is."
  fi
fi
```

Verify the working tree is clean:
```bash
git status --porcelain
```

If there are uncommitted changes, report them and stop. Do not proceed with a dirty tree.

---

## Step 0.5: Capture Session ID

Capture a unique session identifier. All memory files are prefixed with this ID so that multiple instances (e.g., subagents) can run in parallel without file collisions.

```bash
# Portable: works in Alpine, minimal images, and standard Linux
# Generates 8 random hex chars from /dev/urandom, falls back to $RANDOM if unavailable
SESSION_ID=$(head -c 8 /dev/urandom 2>/dev/null | od -An -tx1 | tr -d ' \n' || echo $RANDOM$RANDOM$RANDOM)
echo "SESSION_ID=$SESSION_ID"
```

All subsequent memory file paths use the pattern `memory/${SESSION_ID}-<name>`.

---

## Step 1: Synthesize Detailed Goals

Take the raw goals from the input and expand each into a more detailed, actionable specification. For each goal, produce:

- **Goal:** The original goal string
- **Scope:** What is included and explicitly excluded
- **Key Requirements:** 3-5 concrete requirements
- **Acceptance Criteria:** How success is measured
- **Dependencies:** Any existing code, configs, or specs that need to change
- **Risks / Edge Cases:** Potential pitfalls to consider

Write the detailed goals to `memory/${SESSION_ID}-feature-goals.md` as a markdown document. This file serves as the source of truth for all subsequent audits.

---

## Step 2: Synthesize Proposal Prompt

Using the detailed goals from Step 1, synthesize a multi-paragraph proposal prompt suitable for feeding into `openspec-propose`. The prompt should:

- Start with a parseable change name on the first line: `CHANGE_NAME: <kebab-case-name>` (e.g., `CHANGE_NAME: web-page-summarizer-tui-memory-stats`)
- Include a concise summary of what is being built and why
- Describe the technical approach in 2-4 paragraphs
- Reference specific files, modules, and patterns in the codebase
- Note any architectural decisions or trade-offs
- Be specific enough that `openspec new change` + artifact generation will produce useful specs

Write the proposal prompt to `memory/${SESSION_ID}-feature-prompt.md`.

---

## Step 3: Extract Change Name

Parse the change name from the proposal prompt in Step 2. This is the kebab-case identifier that will be used for the OpenSpec change directory.

```bash
CHANGE_NAME=$(grep -oP '(?<=CHANGE_NAME: )\S+' memory/${SESSION_ID}-feature-prompt.md | head -1)
if [ -z "$CHANGE_NAME" ]; then
  CHANGE_NAME=$(basename "$(pwd)" | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
fi
echo "CHANGE_NAME=$CHANGE_NAME"
```

---

## Step 3.5: Create Feature Branch

Determine the branch type from the input context. If the input contains a reference to an issue with a known type (e.g., from `fix-issue` chain), extract it. Otherwise, default to `feat`.

Valid types: `feat`, `fix`, `chore`, `docs`, `test`.

Strip any existing type prefix from `$CHANGE_NAME` to avoid double-prefixing:

```bash
# Determine branch type — default to feat if not specified
BRANCH_TYPE="${BRANCH_TYPE:-feat}"

# Validate branch type
case "$BRANCH_TYPE" in
  feat|fix|chore|docs|test) ;;
  *) BRANCH_TYPE="feat" ;;
esac

# Strip any existing type prefix from CHANGE_NAME to avoid double-prefixing
BRANCH_NAME=$(echo "$CHANGE_NAME" | sed 's/^\(feat\|fix\|chore\|docs\|test\)\///')
git checkout -b "${BRANCH_TYPE}/${BRANCH_NAME}"
```

Verify the branch was created:
```bash
git branch --show-current
```

---

## Step 4: Create OpenSpec Change

Run the OpenSpec propose workflow for the synthesized change:

```bash
openspec new change "$CHANGE_NAME"
```

Then generate all artifacts using this procedure (optimized for the automated pipeline):

1. Get the artifact build order:
   ```bash
   openspec status --change "$CHANGE_NAME" --json
   ```

2. For each artifact that is `ready` (dependencies satisfied):
   - Get instructions: `openspec instructions <artifact-id> --change "$CHANGE_NAME" --json`
   - Read any completed dependency files for context
   - Create the artifact file using the template and following the instructions
   - Re-run `openspec status --change "$CHANGE_NAME" --json` after each artifact

3. Continue until all `applyRequires` artifacts are complete.

After completion, verify all expected files exist:
```bash
ls -la openspec/changes/$CHANGE_NAME/
```

Expected files: `.openspec.yaml`, `proposal.md`, `design.md`, `tasks.md`, and any spec deltas in `specs/`.

If any expected file is missing, stop and report the error.

---

## Step 5: Audit Specs Against Goals (Up to 3 Iterations)

Read the detailed goals from `memory/${SESSION_ID}-feature-goals.md` and the generated spec documents (proposal.md, design.md, tasks.md, and any spec deltas).

Perform a thorough audit:

1. **Coverage audit:** Does every detailed goal have corresponding requirements in the specs?
2. **Fidelity audit:** Do the specs faithfully represent the original intent of each goal?
3. **Completeness audit:** Are there missing requirements, edge cases, or acceptance criteria not captured?
4. **Consistency audit:** Do the tasks in tasks.md map to the requirements in the specs?

Write audit findings to `memory/${SESSION_ID}-audit-results.md`.

**If errors are found:**
- Fix the spec documents to address each finding
- Update tasks.md if task-to-spec mapping is broken
- Re-audit (increment iteration count)
- Repeat until no errors remain or 3 iterations are exhausted

**If no errors are found:**
- Proceed to Step 6

---

## Step 6: Commit & Push OpenSpec Files (via commit-push)

**Purpose:** Lock the spec documents into a pull request before implementation begins.

**Do not perform git operations inline.** Invoke the `commit-push` skill to handle staging, committing, pushing, and PR creation.

```
commit-push
```

The `commit-push` skill will:
- Stage all OpenSpec files and spec deltas
- Commit using conventional commit format from the scanned project rules §5.1
- Push to the remote
- Create a PR using the template from the scanned project rules §5.4
- Output `PR_NUMBER=<number>` (whether newly created or pre-existing)

After `/commit-push` completes, extract the PR number from its output text (the skill outputs `PR_NUMBER=<number>`):

```bash
# The agent must capture the output from the commit-push chain instruction
# and extract the PR_NUMBER from it.
PR_NUMBER=$(echo "$CHAIN_OUTPUT" | grep -oP 'PR_NUMBER=\K\d+' | head -1)
if [ -z "$PR_NUMBER" ]; then
  echo "ERROR: Could not extract PR_NUMBER from commit-push output. Stopping."
  exit 1
fi
echo "$PR_NUMBER" > memory/${SESSION_ID}-pr-number.txt
```

If `/commit-push` fails, report the error and stop. Do not attempt to recover with manual git commands.

---

## Step 7: Apply Tasks (via /opsx-apply)

Execute tasks using this procedure (simplified for the automated pipeline context):

1. Read `openspec/changes/$CHANGE_NAME/tasks.md` for all unchecked tasks.
2. Create a todo queue mapping each unchecked task to a todo item.
3. Execute tasks sequentially in creation order.
4. When a task completes, mark it `[x]` in `tasks.md`.
5. If a task fails, report the error and stop. Do not continue with incomplete work.
6. Continue until all tasks are complete or the queue is exhausted.

After completion, verify:
- All tasks in `tasks.md` are marked `[x]`

**Check which npm scripts are available before running them** — not all projects define the same scripts:

```bash
AVAILABLE_SCRIPTS=$(node -e "console.log(Object.keys(require('./package.json').scripts || {}).join('\n'))" 2>/dev/null)
```

- If `test` script exists, run: `npm run test`
- If `lint` script exists, run: `npm run lint`
- If `coverage` script exists, run: `npm run coverage`
- If no script exists in package.json, skip the step and note "No script defined — skipped."

If any verification fails, fix the issues and re-verify.

---

## Step 7.5: Commit & Push Implementation Code (via commit-push)

**Purpose:** Push the code produced by `/opsx-apply` to the open PR.

Invoke `commit-push` to stage, commit, push, and update the existing PR:

```
commit-push
```

This will:
- Stage all implementation files (not openspec files — they are already committed)
- Commit using conventional commit format from the scanned project rules §5.1
- Push to the remote
- Update the existing PR (created in Step 6) if `commit-push` detects a pre-existing PR

If `/commit-push` fails, report the error and stop. Do not attempt to recover with manual git commands.

---

## Step 8: Verify Application Starts (`npm start`)

After tasks are applied, verify the application actually starts without crashing.

**Run with a timeout** — this is the preferred method:

```bash
timeout 10 npm start 2>&1 || true
```

This ensures the process terminates after 10 seconds even if it hangs.

If the application fails to start, fix the issue before proceeding. Do not skip this step — a crashing application means the implementation is flawed.

---

## Step 9: Audit Results Against Specs & Goals (Up to 3 Iterations)

Read the original goals from `memory/${SESSION_ID}-feature-goals.md` and the spec documents. Audit the implemented results:

1. **Goal fulfillment:** Does the implementation satisfy every detailed goal?
2. **Spec compliance:** Does the code match the requirements in the spec documents?
3. **Task completion:** Were all tasks in tasks.md actually implemented correctly?
4. **Quality check:** Are there any obvious issues, missing edge cases, or inconsistencies?

Write audit findings to `memory/${SESSION_ID}-audit-results.md` (overwrite previous results).

**If errors are found:**
- Fix the code to address each finding
- Re-verify tests and lint pass
- Re-audit (increment iteration count)
- Repeat until no errors remain or 3 iterations are exhausted

**If no errors are found:**
- Proceed to Step 10

---

## Step 10: Archive and Push (opsx-archive → commit-push)

**Purpose:** Archive the OpenSpec change and push the archive (and any remaining implementation fixes from Step 9) to the open PR.

1. **Archive the OpenSpec change with automatic sync:**

    1. Run the archive:
        ```bash
        openspec archive "$CHANGE_NAME" --yes
        ```

    2. **When presented with a sync prompt**, automatically choose **sync** (the recommended option). Do not require user input — just select sync:
        - If the prompt shows "Sync now (recommended)" / "Archive without syncing" → choose "Sync now (recommended)"
        - If the prompt shows "Archive now" / "Sync anyway" / "Cancel" → choose "Archive now"
        - If no delta specs exist → proceed without sync prompt

    3. **Verify** the archive was created:
        ```bash
        ls openspec/changes/archive/
        ```

    The change directory should now be under `openspec/changes/archive/YYYY-MM-DD-<name>/`.

2. **Push the archive (and all remaining changes) to the open PR** by invoking `commit-push`:

    ```
    commit-push
    ```

    This will:
    - Stage the archived change directory and any delta specs synced from the archive
    - Stage any remaining implementation fixes from Step 9
    - Commit using conventional commit format from the scanned project rules §5.1
    - Push to the remote
    - Update the existing PR (created in Step 6) if `commit-push` detects a pre-existing PR

    If `/commit-push` fails, report the error and stop. Do not attempt to recover with manual git commands.

---

## Step 11: Update PR Title & Description (via update-pr)

**Purpose:** Set the final, accurate PR title and description reflecting what was actually implemented.

Invoke the `update-pr` skill to update the PR. This skill scans project rules for conventions and the PR template, synthesizes both from the delta between the branch and target, and applies changes via `gh api`:

```
update-pr
```

The `update-pr` skill will:
- Gather the actual changes made (`git diff main...HEAD --stat`, `git diff main...HEAD --name-only`)
- Synthesize a final PR description that reflects what was actually implemented, references specific files and modules changed, notes any deviations from the original plan, and includes a testing coverage summary
- Follow the PR template format from the scanned project rules §5.4
- Update the PR using `gh api`

After `update-pr` completes, verify the PR was updated correctly by checking the PR on GitHub.

---

## Step 12: Post Audit Results as PR Comment

Post the final audit results from Step 9 as a comment on the PR:

```bash
PR_NUMBER=$(cat memory/${SESSION_ID}-pr-number.txt)
gh pr comment "$PR_NUMBER" --body "$(cat memory/${SESSION_ID}-audit-results.md)" --repo "$GH_REPO"
```

---

## Step 13: Final Report

Print a summary:

```
Feature lifecycle complete.
Change: $CHANGE_NAME
PR: <URL>
Goals addressed: <count>/<total>
Iterations: <spec-audit-x>, <impl-audit-y>
Tests: passing
Lint: passing
Coverage: maintained
```

---

## Step 14: Cleanup

Remove the intermediate memory files — they served their purpose and won't be needed again:

```bash
rm -f memory/${SESSION_ID}-feature-goals.md memory/${SESSION_ID}-feature-prompt.md memory/${SESSION_ID}-audit-results.md memory/${SESSION_ID}-pr-number.txt
```

Verify cleanup:
```bash
ls memory/${SESSION_ID}-feature-*.md memory/${SESSION_ID}-audit-results.md memory/${SESSION_ID}-pr-number.txt 2>&1
```

If any files remain, report them and remove manually.

---

## Error Handling

- If any step fails, report the error clearly and stop. Do not continue with incomplete work.
- If 3 audit iterations are exhausted and errors remain, report the remaining issues and stop.
- If `gh` API calls fail, report the error and provide the manual commands needed.
- Always leave the repository in a clean state (working tree clean, on the feature branch).
