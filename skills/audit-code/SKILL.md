---
name: audit-code
description: Audits each directory in ./src for bugs, security vulnerabilities, and performance issues. Generates one consolidated issue per directory via /create-issue. Supports parallel scanning of independent directories.
license: BSD 3-Clause
compatibility: Node.js project with ./src directory structure
metadata:
  version: 2.3.0
  phase-mode: parallel
  target: ./src
  agent: code-review
---

# Audit Code

> **⚠️ EXECUTION RULE:** Every code block in this skill is a shell command to **execute**. Do not print them as text, explain them, or treat them as examples — run them directly.

Audit the `./src` directory tree for bugs, security vulnerabilities, and performance issues. Directories are **independent** — each gets its own issue, so they can be scanned in parallel. Fan out one subagent per directory, collect the findings, then create the issues in a single pass. This is the default and preferred mode.

## State Management

Use a state file to persist progress across responses. The state file is cleaned at the start of a new audit and at the end when all phases are complete.

### State File Location

The state file path is provided in the chain context (the text after `/audit-code`). If no path is provided, default to an example name like `audit-state.md` — the agent decides where to place it.

To prevent collisions when multiple instances run, include a session identifier in the path:

```bash
SESSION_ID=$(head -c 8 /dev/urandom 2>/dev/null | od -An -tx1 | tr -d ' \n' || echo $RANDOM$RANDOM$RANDOM)
STATE_FILE="${STATE_FILE_PATH:-audit-state-${SESSION_ID}.md}"
```

### State File Format

```markdown
# Audit State

## Phase Queue
- [x] ./src
- [ ] ./src/agent
- [ ] ./src/utils

## Current Phase
./src/agent

## Completed
- ./src

## Findings
[Current phase findings stored here as markdown table]
```

### State Lifecycle

1. **Start of audit** — Delete any existing state file at the configured path. Begin fresh.
2. **After each directory** — Save the current state (completed directories, current phase, findings).
3. **On resume** — Read the state file to determine where to pick up.
4. **End of audit** — Delete the state file once all phases are complete.

**Cleanup:** The state file should exist only while the audit is actively in progress. Delete it once all phases are complete. If the audit fails or is interrupted, delete the state file as part of cleanup. Do not rely on a shell `trap` — the agent must explicitly remove its work files when the task finishes or when cleaning up after a failure.

---

## Phase Protocol

### Phase 0: Discovery

1. **Derive repository info** once at the start (used for all issue operations):
   ```bash
   GIT_REMOTE=$(git remote get-url origin 2>/dev/null)
   if echo "$GIT_REMOTE" | grep -q '^git@'; then
     REPO=$(echo "$GIT_REMOTE" | sed 's/.*@[^:]*:\(.*\).git$/\1/')
   elif echo "$GIT_REMOTE" | grep -q 'github\.com'; then
     REPO=$(echo "$GIT_REMOTE" | sed 's/.*github\.com[/:]\(.*\).git$/\1/')
   else
     echo "ERROR: Could not parse repository from remote '$GIT_REMOTE'."
     exit 1
   fi
   echo "REPO=$REPO"
   ```

2. **Clean state.** Delete any existing state file: `rm -f "$STATE_FILE"`

3. **Enumerate directories.** List all directories to audit:
   ```bash
   # List ./src and all immediate subdirectories, excluding common patterns
   find ./src -maxdepth 1 -type d | sort
   ```

4. Sort them alphabetically (the `find | sort` above handles this).

5. Build the phase queue: `[./src, subdir1, subdir2, ...]` in alphabetical order.

6. **Save the phase queue** to the state file:
   ```bash
   cat > "$STATE_FILE" << EOF
   # Audit State

   ## Phase Queue
   - [x] ./src
   EOF
   # Append subdirectories
   find ./src -mindepth 1 -maxdepth 1 -type d | sort | while read dir; do
     echo "- [ ] $dir" >> "$STATE_FILE"
   done
   echo "" >> "$STATE_FILE"
   echo "## Current Phase" >> "$STATE_FILE"
   echo "./src" >> "$STATE_FILE"
   echo "" >> "$STATE_FILE"
   echo "## Completed" >> "$STATE_FILE"
   echo "- ./src" >> "$STATE_FILE"
   echo "" >> "$STATE_FILE"
   echo "## Findings" >> "$STATE_FILE"
   ```
6. **Proceed** to parallel processing immediately.

### Parallel Directory Processing

Process directories concurrently. Each directory is independent and gets its own issue, so there is no ordering dependency.

#### Phase A: Fan out scans

For each directory in the queue, dispatch a subagent to scan it. The subagent reads the files and returns a structured findings list. Use the `task` tool with the `code-review` subagent type (or `general-purpose` if `code-review` is unavailable). Batch the dispatches with `Promise.all`, bounded to a reasonable concurrency (for example, ten at a time).

Each subagent prompt must include:
- The directory path to audit (absolute or relative to the project root).
- The concrete pattern checks (silent catch, unhandled promises, console.log, eval, SQL injection, sync I/O, wildcard exports, while-true, unbounded Sets/Maps, module-level cwd capture, duplicated logic).
- The severity criteria (critical/high/medium/low).
- The instruction to return findings as a structured list of `{ file, type, severity, summary }`.
- The instruction to NEVER include raw secret values in the report.

#### Phase B: Create one issue per directory

After all scans return, create one issue per directory that has findings. For each directory:

1. **Classify severity.** Use these concrete criteria:
   - **Critical:** Data breach risk, production crash, data loss, authentication bypass
   - **High:** Functional bug affecting users, XSS, SQL injection, memory leak
   - **Medium:** Performance degradation, missing error handling, code smell
   - **Low:** Minor optimization, style inconsistency, TODO items

2. **Create Issue.** Invoke the `create-issue` skill with a synthesized description:
   ```
   create-issue Audit findings for $DIR: Found N critical, N high, N medium, N low issues. See audit results below for details.
   ```

   **IMPORTANT — category:** Audit findings are **fixes** (defects to repair), not features. Pass `fix` as the category so the issue is labeled `bug`, not `feature`. Do not rely on create-issue's default-to-feat rule.

   **Capture `ISSUE_NUMBER` from the create-issue output** (the chained skill prints `ISSUE_NUMBER=<number>`).

3. **Update Issue Body.** Append a structured audit table to the issue. **Read the current body first, then append — do NOT overwrite.** The synthesized description must be preserved:

   ```bash
   CURRENT_BODY=$(gh issue view "$ISSUE_NUMBER" --json body --jq '.body' --repo "$REPO")
   AUDIT_FILE=$(mktemp)
   printf '%s\n' "$CURRENT_BODY" > "$AUDIT_FILE"
   printf '\n## Audit Findings\n\n| File | Type | Severity | Summary |\n|------|------|----------|---------|\n' >> "$AUDIT_FILE"
   # Append each finding as a table row
   printf '| %s | %s | %s | %s |\n' "$file" "$type" "$severity" "$summary" >> "$AUDIT_FILE"
   gh issue edit "$ISSUE_NUMBER" --body-file "$AUDIT_FILE" --repo "$REPO"
   rm -f "$AUDIT_FILE"
   ```

   **Verify** the body contains both the description and the audit table:
   ```bash
   VERIFY_BODY=$(gh issue view "$ISSUE_NUMBER" --json body --jq '.body' --repo "$REPO" 2>/dev/null || true)
   if echo "$VERIFY_BODY" | grep -q "File.*Type.*Severity.*Summary"; then
     echo "Audit table verified on issue #$ISSUE_NUMBER."
   else
     echo "WARNING: Audit table not found on issue #$ISSUE_NUMBER."
   fi
   ```

4. **Report Output.** Use the standard Output Format below.

#### Phase C: Cleanup

Delete the state file (`rm -f "$STATE_FILE"`) and report completion.

---

## Execution Rules

1. **PARALLEL PROCESSING.** Scan directories concurrently via subagents. Directories are independent — each gets its own issue, so there is no ordering dependency.
2. **ONE ISSUE PER DIRECTORY.** Do not combine multiple directories into a single issue.
3. **FIX CATEGORY.** Audit findings are fixes, not features. Create each issue with the `fix` category so it is labeled `bug`, never `feature`.
4. **PRESERVE ISSUE BODY.** When appending the audit table, read the current body first and append — never overwrite the synthesized description.
5. **STOP CONDITION.** Terminate when all directories in the phase queue have been processed and their issues created.

## Error Handling

- **State file not found:** If the state file doesn't exist on resume, begin fresh from Phase 0.
- **Directory not found:** If a directory in the phase queue doesn't exist, skip it and log a warning.
- **gh CLI not authenticated:** Suggest running `gh auth login`.
- **Network error:** Retry once, then report the failure and skip the current directory.
- **create-issue failure:** Log the error, skip issue creation for this directory, and continue. Do not abort the audit.
- **Partial audit:** If some directories succeed and others fail, report what was accomplished. Never leave the audit half-done.
- **Rate limit:** If GitHub API returns 403 rate limit, wait 60 seconds and retry once.

## Gotchas

- **Security findings must never include raw secret values.** The audit table only reports pattern type and count — never the matched line content. Writing secrets to a GitHub issue exposes them to anyone with repo access.
- **State file cleanup is critical.** Always delete the state file at the start of a new audit and at the end when complete. Stale state files cause incorrect resume behavior.
- **`gh` CLI must be authenticated before starting.** The audit will fail silently at Step 6 if `gh` is not logged in. Run `gh auth login` first.
- **Never overwrite the issue body.** `gh issue edit --body-file` replaces the entire body. When appending the audit table, read the current body first and append — otherwise you clobber the synthesized description. This is a real, observed failure mode.
- **Audit findings are fixes, not features.** Defaulting to `feat`/`feature` mislabels every defect as an enhancement. Pass `fix`/`bug` explicitly when chaining to create-issue.

## Output Format

Each phase response must include:

```
## Audit: [directory path]
- **Status:** completed | blocked | no-issues
- **Files Audited:** [count]
- **Issues Found:** [count] (critical: N, high: N, medium: N, low: N)
- **Details:**
  - [key-finding-1]
  - [key-finding-2]
- **Action Items:** [issue created with ID or "No issues detected"]
- **Next Steps:** [next directory name or "All phases complete"]
```