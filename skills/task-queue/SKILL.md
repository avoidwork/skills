---
name: task-queue
description: Accept a list of tasks (JSON or natural language), execute shell commands sequentially with fail-fast logic, and report a structured summary.
license: BSD 3-Clause
compatibility: Requires a shell (bash or sh) with `timeout` available.
metadata:
  agent: coding
---

# Task Queue

You are an autonomous executor. Your job is to accept a task list, execute shell commands sequentially with fail-fast logic, and report a structured summary.

## 1. Ingest & Parse

Accept the task list. It may be provided as:
- **JSON Array:** `[{ "id": "1", "description": "Run lint", "command": "npm run lint" }]`
- **Natural Language:** A numbered or bulleted list.

**Action:** Parse the input into a standardized structure. Ensure every task has:
- `id` — unique string/number
- `description` — human-readable
- `command` — shell command to execute

**Handle edge cases:**
- **Empty task list:** If the input is empty or contains zero tasks, report "No tasks to execute." and stop.
- **Malformed JSON:** If the input is supposed to be JSON but is malformed, report the parse error and stop.
- **Missing fields:** If a task is missing `id`, `description`, or `command`, skip that task and log a warning. Do not abort.
- **Duplicate IDs:** If two tasks have the same `id`, keep the first occurrence and skip duplicates with a warning.

## 2. Execute (Fail-Fast)

Iterate through tasks in order. **Do not pause for user confirmation.**

**Important:** This skill manages its own task tracking. Do not rely on the system prompt's todo management — create and manage a structured todo list for visibility:

```bash
# Before starting, create todo items for each task
# (Create a todo list with each task from the parsed input)
```

For each task:

1. **Sanitize the command.** Reject commands that contain high-risk patterns (shell metacharacters used for injection) unless they are part of a safe, expected construct. The following patterns are **always rejected**:

    - Semicolons used as command separators after a suspicious context: `"; rm -rf"`, `"; rm -rf /"`
    - `&&` with filesystem-modifying commands in suspicious context: `" && rm -rf /"`
    - `| sh` or `| bash` (pipe into shell execution)
    - Backticks used outside the expected construct

    Safe constructs that should be **allowed**:
    - `"||"` for fallback chains: `"test -f config.sh || echo default"`
    - `"&&"` with logical operations in expected contexts: `npm run lint && echo done"`
    - `"${HOME}"`, `"$(pwd)"`, backticks inside safe contexts
    - Redirections and pipes to non-shell commands: `> /tmp/out`, `| grep foo`

    ```bash
    # Sanitization function — return non-zero if the command is unsafe
    sanitize_command() {
      local cmd="$1"

      # Block pipe to shell execution
      if echo "$cmd" | grep -qP '\\|\\s*(bash|sh|zsh|ksh)\\b'; then
        echo "REJECTED: Command pipes to shell execution."
        return 1
      fi

      # Block obvious destructive patterns
      if echo "$cmd" | grep -qP '(\\brm\\s+-rf\\s+\\\/|\\brm\\s+-rf\\s+\\*)\\b'; then
        echo "REJECTED: Command contains destructive filesystem operations."
        return 1
      fi

      # Block eval/exec of arbitrary strings
      if echo "$cmd" | grep -qP '\\b(eval|exec)\\s+'; then
        echo "REJECTED: Command uses eval/exec to interpret arbitrary strings."
        return 1
      fi

      return 0
    }

    # Use the sanitization function
    if ! sanitize_command "$COMMAND"; then
      echo "Task $TASK_ID skipped: command rejected by security filter."
      TASK_OUTPUT="REJECTED: Command failed security sanitization."
      TASK_EXIT_CODE=1
      TASK_FAILED=true
      break
    fi
    ```

2. **Run a single command with timeout and capture output.** Do **not** run the command twice. Use only the capture form:

    ```bash
    TASK_OUTPUT=$(timeout 300 bash -c "$COMMAND" 2>&1)
    TASK_EXIT_CODE=$?
    ```

    The `timeout 300` limits each command to 5 minutes. The `bash -c "$COMMAND"` ensures directory changes are scoped to the subshell.

3. **Handle working directory isolation.** The `bash -c` wrapper already isolates directory changes, so subsequent commands run in the original directory. No additional handling is needed.

4. **Check Result:**
    - **If Success (exit code 0):** Log the task as completed. If the output is empty, note "No output." If the output exceeds 10,000 characters, truncate to the last 5,000 characters and note "[truncated]".
    - **If Failure (non-zero exit code):**
      - Record the error output (last 50 lines if output is large).
      - **STOP IMMEDIATELY.** Do not process remaining tasks.
      - Report the failure with the error details.

5. **Handle commands requiring user interaction:** If a command appears to hang (no output for > 30 seconds), assume it requires user input. Kill the process and report: "Command appears to require interactive input. Aborting."

## 3. Report

Once the queue finishes (all completed or a failure occurred), generate a clear summary:

```
Task Queue Complete.
✅ Completed: <count> tasks
❌ Failed: <count> tasks (include error details)
⏳ Remaining: <count> tasks (not processed due to failure)
```

Include the error output from the failed task if applicable.

---
