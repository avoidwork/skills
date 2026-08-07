---
name: create-issue
description: Receives a user description, synthesizes it into a title and description, categorizes as 'fix' or 'feat', creates a GitHub issue, audits the codebase for actionable details, and updates the issue with findings.
license: BSD 3-Clause
compatibility: Requires gh CLI authenticated with the repo. Must be run from the project root.
metadata:
  agent: coding
---

# Create Issue

Receives a user description, synthesizes it into a polished title and description, categorizes the work as a **fix** or **feat**, creates a GitHub issue, audits the codebase for actionable details, and updates the issue with findings.

## Workflow

### 1. Parse Input

The user's description is provided in the command context (the text after `/create-issue`). **Use it directly.** Do not ask the user for it.

If no description was provided, ask the user for one.

### 2. Categorize

**CRITICAL LABEL RULE:** The only valid GitHub labels are `bug` and `feature`. **Never** use `enhancement`, `improvement`, `refactor`, `docs`, or any other label. This is non-negotiable.

Analyze the user's description and determine whether this is a **fix** or a **feat**:

- **fix** — A bug, regression, unexpected behavior, crash, or something that is broken and needs repair.
- **feat** — A new capability, enhancement, improvement, or something that doesn't exist yet.

**Default to `feat` (→ `feature` label) unless the description is clearly a bug.** If you are even slightly uncertain, it is a feature. There is no third option.

**Capture the category and label as variables:**

```bash
# Set CATEGORY and LABEL based on the analysis above
CATEGORY="<fix|feat>"
LABEL="<bug|feature>"
```

### 3. Synthesize Title

Create a concise, conventional-commit-style title:

- **Fix:** `fix: <short description>` — e.g., `fix: crash on empty input`
- **Feat:** `feat: <short description>` — e.g., `feat: add file upload endpoint`

Keep it under 70 characters. Be specific but terse. No trailing punctuation.

**Capture the title in a variable:**

```bash
SYNTHESIZED_TITLE="<fix|feat>: <short description>"
```

### 4. Synthesize Description

Read the appropriate template from disk, then populate every section with synthesized content derived from the user's description.

**This is purely a grammatical task.** Do not search the codebase, read source files, or look up implementation details. You are rephrasing and structuring what the user has already told you — not investigating the code. The audit (step 6) is where code investigation happens. If you don't know a detail, write `Unknown — user to confirm`.

**Read the template first** — the skill must load the actual file from `.github/ISSUE_TEMPLATE/` so it stays in sync with whatever is on disk:

- **fix** → first try `.github/ISSUE_TEMPLATE/bug_report.md`, fall back to `.github/ISSUE_TEMPLATE.md`
- **feat** → first try `.github/ISSUE_TEMPLATE/feature_request.md`, fall back to `.github/ISSUE_TEMPLATE.md`

**Verify template exists before reading**; fall back gracefully:

```bash
# Determine which template to use
if [ "$CATEGORY" = "fix" ]; then
  TEMPLATE_PATH=".github/ISSUE_TEMPLATE/bug_report.md"
else
  TEMPLATE_PATH=".github/ISSUE_TEMPLATE/feature_request.md"
fi

# Try the specific template first, then fall back to the general one
if [ ! -f "$TEMPLATE_PATH" ]; then
  TEMPLATE_PATH=".github/ISSUE_TEMPLATE.md"
fi

# If nothing exists, report error and stop
if [ ! -f "$TEMPLATE_PATH" ]; then
  echo "ERROR: No issue template found. Expected one of:"
  echo "  .github/ISSUE_TEMPLATE/bug_report.md"
  echo "  .github/ISSUE_TEMPLATE/feature_request.md"
  echo "  .github/ISSUE_TEMPLATE.md"
  exit 1
fi
```

The template may contain YAML frontmatter — strip it before populating. Use `sed` to remove lines between the first `---` and second `---` portably (with `-e` for multi-command):

```bash
# Strip YAML frontmatter (lines between first and second ---)
sed -e '1,/^---$/d' -e '/^---$/,$d' "$TEMPLATE_PATH"
```

**Rules for synthesis:**
- Fill **every** section. Never leave a section blank.
- If you cannot infer a detail (e.g., OS, Node version), write `Unknown — user to confirm` rather than guessing.
- The **Summary** should be 1-2 sentences, clear and direct.
- The **Motivation** (for feats) should explain the "why" — the problem being solved.
- The **Proposed Solution** (for feats) should be actionable — what the implementation would look like.
- For fixes, the **Reproduction** steps should be concrete and sequential.
- For fixes, the **Expected** vs **Actual** behavior should be clearly contrasted.

**Capture the populated template body** — store it in `$FULL_TEMPLATE_BODY` for use in Step 5:

```bash
# Read the template, strip frontmatter, and store in variable
FULL_TEMPLATE_BODY=$(sed -e '1,/^---$/d' -e '/^---$/,$d' "$TEMPLATE_PATH")
```

### 4.5. Populate Environment Section

**If the template contains an "Environment" section, populate it with live machine details before creating the issue.** Do this after step 4 (Synthesize Description) and before step 5 (Create the Issue).

1. **Check for an Environment section** — Scan the populated template body for a section titled `## Environment` (or similar, case-insensitive). If none exists, skip this step.

2. **Gather machine details:**
   - **OS:** Run `uname -s` and `uname -r` to get the OS name and kernel version. Format: `OS_NAME KERNEL_VERSION` (e.g., `Linux 7.0.2-7-pve`).
   - **Node.js:** Run `node --version` to get the Node version (e.g., `v25.8.1`).
   - **madz version:** Read `package.json` from the project root and extract the `version` field. Use `node -e "console.log(require('./package.json').version)"`.
   - **LLM provider:** If the user's description mentions a specific provider (e.g., OpenAI, Anthropic), use that. Otherwise write `Unknown — user to confirm`.

3. **Replace placeholder values** — In the template body, replace the Environment section's placeholder lines with the actual values. Typical placeholders look like:
   ```
   - **OS**: (e.g., macOS 14.5, Ubuntu 24.04, Windows 11)
   - **Node.js**: (e.g., 24.2.0)
   - **madz version**: (e.g., 1.7.3 — check `npm list @avoidwork/madz` or Docker tag)
   - **LLM provider**: (e.g., OpenAI gpt-4o, Anthropic claude-3.5-sonnet)
   ```
   Replace each line's placeholder with the actual value. Preserve the `**KEY**:` format.

4. **Handle `#` characters carefully** — GitHub interprets `#` as a link trigger in certain contexts. When writing values, **always omit `#` in inline content**:

    - ❌ `issue #290` in prose → ✅ `issue 290` in prose
    - ❌ `line #42` → ✅ `line 42`
    - ❌ `v25#8` → ✅ `v25.8`

    **Exception:** Issue references in the form `#<number>` (e.g., `resolves #42`, `see #17`) are **preserved** and SHOULD NOT be stripped — GitHub uses them to create clickable links to those issues. This rule applies only to stray `#` characters in content, not to valid GitHub issue references.

5. **Write the updated body** to the temp file before proceeding to step 5.

**Example transformation:**
```
Before:
## Environment
- **OS**: (e.g., macOS 14.5, Ubuntu 24.04, Windows 11)
- **Node.js**: (e.g., 24.2.0)
- **madz version**: (e.g., 1.7.3)
- **LLM provider**: (e.g., OpenAI gpt-4o)

After:
## Environment
- **OS**: Linux 7.0.2-7-pve
- **Node.js**: v25.8.1
- **madz version**: 1.7.5
- **LLM provider**: OpenAI gpt-4o
```

### 5. Create the Issue — **EXECUTE THIS BEFORE ANY CODE SEARCH**

**This is the hard stop.** You must create the GitHub issue *now*, with nothing more than the synthesized title and description. No codebase search. No audit. No looking at files. The issue must exist on GitHub before you touch a single source file.

**Determine the target repository dynamically from the git remote.** Never hardcode a repo name:

```bash
# Extract owner/repo from git remote URL — handles both HTTPS and SSH
GIT_REMOTE=$(git remote get-url origin 2>/dev/null)
if [ -z "$GIT_REMOTE" ]; then
  echo "ERROR: No git remote 'origin' found. Cannot determine repository."
  exit 1
fi

GH_REPO_FLAG=""
# SSH format: git@github.com:owner/repo.git
if echo "$GIT_REMOTE" | grep -q '^git@'; then
  GH_REPO=$(echo "$GIT_REMOTE" | sed 's/.*@[^:]*:\(.*\).git$/\1/')
  GH_REPO_FLAG="--repo $GH_REPO"
# HTTPS format: https://github.com/owner/repo.git or https://TOKEN@github.com/owner/repo.git
elif echo "$GIT_REMOTE" | grep -q 'github\.com'; then
  GH_REPO=$(echo "$GIT_REMOTE" | sed 's/.*github\.com[/:]\(.*\).git$/\1/')
  GH_REPO_FLAG="--repo $GH_REPO"
fi

if [ -z "$GH_REPO" ]; then
  echo "ERROR: Could not parse repository from remote '$GIT_REMOTE'. Cannot create issue."
  exit 1
fi
```

**Execution order is non-negotiable:**
1. Synthesize title and description (steps 2–4)
2. Populate Environment section if present (step 4.5)
3. **Create the issue on GitHub** (this step)
4. *Then and only then* proceed to the audit (step 6)

```bash
BODY_FILE=$(mktemp)

cat > "$BODY_FILE" << BODYEOF
$(echo "$FULL_TEMPLATE_BODY" | sed "s|<SYNTHESIZED_TITLE>|$SYNTHESIZED_TITLE|g")
BODYEOF

ISSUE_URL=$(gh issue create \
  --title "$SYNTHESIZED_TITLE" \
  --body-file "$BODY_FILE" \
  --label "$LABEL" \
  $GH_REPO_FLAG)

ISSUE_NUMBER=$(echo "$ISSUE_URL" | grep -oP '/issues/\K\d+')

rm -f "$BODY_FILE"
```

**Important:** Replace all `<PLACEHOLDER>` tokens with actual variable values before executing. Never pass literal placeholder strings to `gh` commands.

**STOP here.** The issue is created. Report the number and URL. Then proceed to step 6.

### 6. Light Audit — Gather actionable context

*Now* that the issue exists, do a **targeted, lightweight** audit. **You must use the `ISSUE_NUMBER` returned in Step 5.** Anchor every finding to this specific issue. The goal is to surface useful context — not to solve the problem or write a treatise.

#### For **fix** issues — quick root-ause hunt:

1. **Search for relevant code** — Use `searchFiles` to find files, functions, or patterns related to the bug.
2. **Read the most relevant file** — Open one or two key files. Look for the code that handles the reported scenario.
3. **Note what you find** — File paths, line numbers, function names, the likely culprit. Keep it brief. Explicitly reference `#<ISSUE_NUMBER>` when documenting findings.

#### For **feat** issues — quick landscape scan:

1. **Search for existing related functionality** — What's already there?
2. **Identify integration points** — Where would this plug in?
3. **Note relevant files** — Which files would need changes? Explicitly reference `#<ISSUE_NUMBER>` when documenting findings.

**Time budget:** Keep this to a few focused searches and reads. If the audit yields nothing obvious, move on — don't spin your wheels.

#### Audit output format:

```markdown
## Audit Findings (for Issue #<ISSUE_NUMBER>)

- **File**: `path/to/file.ts` — <brief, actionable observation>
- `path/to/related.ts` — <why this is relevant>
- <any concrete guidance for the implementer>
```

### 6.5 Update Issue with Audit Findings

**After the audit (step 6), append the findings to the issue body.** Do not skip this step — the audit notes must be visible on the issue.

1. **Build the audit section** using the format from step 6.

2. **Append to the existing issue body** using `gh issue edit`. Read the current body, append the audit section, and write it back:

```bash
# Read current body
CURRENT_BODY=$(gh issue view <ISSUE_NUMBER> --json body --jq '.body')

# Build the audit section
AUDIT_SECTION="
## Audit Findings (for Issue #<ISSUE_NUMBER>)

- **<file-path>** — <brief, actionable observation>
- <more findings...>
"

# Append and update — write to temp file to avoid stdin escaping issues
AUDIT_FILE=$(mktemp)
cat > "$AUDIT_FILE" << AUDITEOF
## Audit Findings (for Issue #${ISSUE_NUMBER})

- **<file-path>** — <brief, actionable observation>
- <more findings...>
AUDITEOF

gh issue edit "$ISSUE_NUMBER" --body-file "$AUDIT_FILE" $GH_REPO_FLAG
rm -f "$AUDIT_FILE"
```

3. **Verify the audit notes are present.** Read the issue back and confirm the audit section exists:

```bash
VERIFY=$(gh issue view "$ISSUE_NUMBER" --json body --jq '.body' $GH_REPO_FLAG)
if echo "$VERIFY" | grep -q "Audit Findings"; then
  echo "Audit notes verified on issue."
else
  echo "WARNING: Audit notes not found on issue. Retrying..."
  # Retry once with the same append logic
  echo -e "${CURRENT_BODY}${AUDIT_SECTION}" | gh issue edit "$ISSUE_NUMBER" --body-file - $GH_REPO_FLAG
fi
```

4. **If verification fails after retry**, report the failure in the final summary but do not block — the issue still exists.

### 6.5.1 Handle Large Audit Findings

GitHub issue bodies have a 65,536 character limit. If your audit findings are large:
- Keep findings concise — one line per file, no verbose explanations
- If findings exceed 10,000 characters, summarize: list files with issues and note "Full details in attached log" (then attach if possible)
- Never truncate mid-sentence — always end with a complete thought

### 7. Report

Print a final summary:

```
Created issue #<ID>: <TITLE>
URL: <URL>
Category: <fix|feat>
Audit: <brief note on what was found — e.g., "root cause identified in callHandler.ts:142" or "mapped 3 integration points for proposed feature">
```

## Error Handling

- **Not authenticated:** Suggest running `gh auth login`.
- **Network error:** Retry once, then report the failure.
- **Template not found:** Report the error and stop — do not proceed without the template.
- **GitHub API error:** Report the specific error message and stop.
- **Audit yields nothing:** If the codebase search returns no relevant results, note "No relevant code paths found — issue may be environment-specific or in external dependencies." and still report the issue creation.

## Example

```
User: create-issue the TUI hangs when the user types a very long command and presses enter

Agent: Categorizing... This is a fix — the TUI should handle long input gracefully.

Synthesizing title: fix: TUI hangs on long command input

Filling bug_report template...
  Summary: The TUI freezes when a user enters a command exceeding a certain length and presses Enter.
  Reproduction: 1. Start madz TUI 2. Type a very long command (e.g., 500+ characters) 3. Press Enter
  Expected: The command is processed or truncated gracefully.
  Actual: The TUI becomes unresponsive and requires a force quit.

Creating issue...
Created issue #256: fix: TUI hangs on long command input
URL: https://github.com/avoidwork/madz/issues/256

Auditing codebase...
  Found input handler in src/tui/input.ts:87 — no length validation before dispatch.
  Related: src/tui/provider.ts handles command dispatch but assumes valid input.

Updating issue with audit findings...

Created issue #256: fix: TUI hangs on long command input
URL: https://github.com/avoidwork/madz/issues/256
Category: fix
Audit: Root cause in src/tui/input.ts:87 — no length validation before dispatch
```

---
