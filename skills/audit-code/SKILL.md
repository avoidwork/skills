---
name: audit-code
description: Audits each directory in ./src for bugs, security vulnerabilities, and performance issues sequentially. Generates one consolidated issue per directory via /create-issue.
license: BSD 3-Clause
compatibility: Node.js project with ./src directory structure
metadata:
  version: 2.2.0
  phase-mode: sequential
  target: ./src
  agent: code-review
---

# Audit Code

Audit the `./src` directory tree for bugs, security vulnerabilities, and performance issues. Execute **sequentially** — process one directory at a time, saving state between steps.

## State Management

Use a state file to persist progress across responses. The state file is cleaned at the start of a new audit and at the end when all phases are complete.

### State File Location

The state file path is provided in the chain context (the text after `/audit-code`). If no path is provided, default to an example name like `audit-state.md` — the agent decides where to place it.

Store the path in a variable:
```bash
STATE_FILE="${STATE_FILE_PATH:-audit-state.md}"
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

---

## Phase Protocol

### Phase 0: Discovery

1. **Clean state.** Delete any existing state file: `rm -f "$STATE_FILE"`
2. **Enumerate directories.** List all directories to audit:
   ```bash
   # List ./src and all immediate subdirectories, excluding common patterns
   find ./src -maxdepth 1 -type d | sort
   ```
3. Sort them alphabetically (the `find | sort` above handles this).
4. Build the phase queue: `[./src, subdir1, subdir2, ...]` in alphabetical order.
5. **Save the phase queue** to the state file:
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
6. **Proceed** to sequential processing immediately.

### Sequential Directory Processing

Process directories one at a time in alphabetical order.

#### Directory Scan Execution

For each directory in the queue:

1. **Read state.** Read the state file to get the current phase:
   ```bash
   CURRENT_PHASE=$(grep -A1 "^## Current Phase" "$STATE_FILE" | tail -1 | tr -d ' \n')
   echo "Processing: $CURRENT_PHASE"
   ```

2. **Enumerate files.** List all files in the target directory:
   ```bash
   # Recurse into subdirectories, excluding common patterns
   find "$CURRENT_PHASE" -type f \
     ! -path '*/node_modules/*' \
     ! -path '*/.git/*' \
     ! -path '*/__tests__/*' \
     ! -path '*/.next/*' \
     ! -path '*/dist/*' \
     ! -path '*/build/*' \
     ! -name '*.test.*' \
     ! -name '*.spec.*' \
     ! -name '*.d.ts' \
     ! -name '*.js.map' \
     \( -name '*.js' -o -name '*.ts' -o -name '*.jsx' -o -name '*.tsx' \)
   ```

3. **Audit each file.** For each file, check for issues using concrete patterns:

   **Bugs** — grep for common anti-patterns:
   ```bash
   # Silent catch blocks
   grep -rn 'catch\s*(' "$file" | grep -v 'err' | grep -v 'error'
   # Unhandled promises (no .catch())
   grep -rn '\.then(' "$file" | grep -v '\.catch('
   # console.log in production code
   grep -rn 'console\.log' "$file"
   ```

    **Security** — grep for OWASP Top 10 violations (per AGENTS.md §1.2):
    ```bash
    # Hardcoded secrets — capture count only, NEVER print values to issue body
    SECRET_COUNT=$(grep -rnc 'password\s*=\s*["\x27]' "$file" 2>/dev/null | tail -1 | cut -d: -f2)
    SECRET_COUNT=${SECRET_COUNT:-0}
    # eval() usage (forbidden per AGENTS.md §1.1)
    grep -rn 'eval(' "$file"
    # SQL injection vectors (string concatenation in queries)
    grep -rn "query\s*.*\+\s*req" "$file"
    # Missing auth middleware on routes
    # (check route definitions for missing auth guards)
    ```

    **CRITICAL:** When writing security findings to the issue body or state file, NEVER include the matched line content. Only report the pattern type and count:

    ```
    | [file] | security | critical | Potential hardcoded secret: password/api_key pattern detected (N occurrences) |
    ```

    The actual secret value must never appear in any output — it will be written to a GitHub issue via `gh issue edit`, exposed to anyone with repo access.

   **Performance** — grep for common issues:
   ```bash
   # Synchronous file I/O in async functions
   grep -rn 'readFileSync\|writeFileSync' "$file"
   # Unbounded loops
   grep -rn 'while\s*(true)' "$file"
   # Large bundle imports
   grep -rn 'import.*\*' "$file"
   ```

4. **Classify severity.** Use these concrete criteria:
   - **Critical:** Data breach risk, production crash, data loss, authentication bypass
   - **High:** Functional bug affecting users, XSS, SQL injection, memory leak
   - **Medium:** Performance degradation, missing error handling, code smell
   - **Low:** Minor optimization, style inconsistency, TODO items

5. **Create Issue.** Invoke the `create-issue` skill with a synthesized description:
   ```
   create-issue Audit findings for $CURRENT_PHASE: Found N critical, N high, N medium, N low issues. See audit results below for details.
   ```
   Capture the issue number from the output:
   ```bash
   ISSUE_NUMBER=$(echo "$CREATE_ISSUE_OUTPUT" | grep -oP '#\d+' | head -1 | tr -d '#')
   ```
   If the issue number cannot be extracted, note it but continue.

   **After creating the issue (or failing to), continue to Step 6.** Do not stop or wait for further input — the pipeline proceeds automatically.

6. **Update Issue Body.** Append a structured audit table to the issue:
   ```bash
   AUDIT_TABLE="| File | Type | Severity | Summary |\n|------|------|----------|---------|\n"
   # Build table from findings
   # ... (populate from audit results)
   echo -e "$AUDIT_TABLE" > /tmp/audit-table.md
   gh issue edit "$ISSUE_NUMBER" --body-file /tmp/audit-table.md --repo avoidwork/madz
   ```

7. **Update State.** Mark the directory as completed in the state file:
    ```bash
    # Use | as sed delimiter to safely handle directory paths containing /
    ESCAPED_PHASE=$(echo "$CURRENT_PHASE" | sed 's/[|&]/\\&/g')
    sed -i "s|- \[ \] $ESCAPED_PHASE|- [x] $ESCAPED_PHASE|" "$STATE_FILE"
    # Append findings
    echo "" >> "$STATE_FILE"
    echo "## Findings: $CURRENT_PHASE" >> "$STATE_FILE"
    echo "[Findings stored here]" >> "$STATE_FILE"
    ```

8. **Report Output.** Use the standard Output Format below.

9. **Advance.** Move to the next directory in the queue. If none remain, delete the state file (`rm -f "$STATE_FILE"`) and report completion.

---

## Execution Rules

1. **SEQUENTIAL PROCESSING.** Process directories one at a time in alphabetical order.
2. **NO PARALLELISM.** Do not spawn subagents or process multiple directories concurrently.
3. **STATE PERSISTENCE.** Always save state after processing each directory. Always read state before starting a new directory.
4. **ONE ISSUE PER DIRECTORY.** Do not combine multiple directories into a single issue.
5. **STOP CONDITION.** Terminate when all directories in the phase queue have been processed.

## Error Handling

- **State file not found:** If the state file doesn't exist on resume, begin fresh from Phase 0.
- **Directory not found:** If a directory in the phase queue doesn't exist, skip it and log a warning.
- **gh CLI not authenticated:** Suggest running `gh auth login`.
- **Network error:** Retry once, then report the failure and skip the current directory.
- **create-issue failure:** Log the error, skip issue creation for this directory, and continue. Do not abort the audit.
- **Partial audit:** If some directories succeed and others fail, report what was accomplished. Never leave the audit half-done.
- **Rate limit:** If GitHub API returns 403 rate limit, wait 60 seconds and retry once.

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