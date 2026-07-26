---
name: commit-push
description: Automates the complete git workflow: scans project rules for conventions, stages all changes, commits, pushes to the remote, and opens a Pull Request. Ensures strict compliance with documentation rules, including rule 5.4.
license: BSD 3-Clause
compatibility: Requires git CLI configured with remote access and PR creation tools (e.g., gh).
metadata:
  agent: coding
---

# Commit & Push

You are the final step in the craft. Precision matters. Follow these steps in order.

## 1. Read the Rules

Before touching a single file, scan the project root and any specified paths for an `AGENTS.md` rules file and read its contents.

- Identify the required commit message format (e.g., Conventional Commits).
- Identify the required PR title/body format (look for patterns like a PR template reference).
- **Crucially:** Locate and strictly follow **Rule 5.4** (Pull Request Templates section). Extract the rules about how PR title and body should be constructed.
- **If Rule 5.4 is not found:** Log a WARN and proceed without it — do not block on missing documentation.

## 2. Check for Detached HEAD

Before doing anything else, verify git is not in a detached HEAD state:

```bash
git branch --show-current
```

If the output is empty (detached HEAD), create a branch from the current commit:

```bash
git checkout -b "detached-fix-$(date +%s)"
```

## 3. Create & Checkout a Branch

Check if already on a feature branch. If the current branch matches `feat/*`, `fix/*`, `docs/*`, or `chore/*`, skip branch creation and proceed to Step 4:

```bash
CURRENT_BRANCH=$(git branch --show-current)
if echo "$CURRENT_BRANCH" | grep -qE '^(feat|fix|docs|chore)/'; then
  echo "Already on feature branch: $CURRENT_BRANCH. Skipping branch creation."
else
  SYNTHESIZE_BRANCH=true
fi
```

If branch creation is needed (`SYNTHESIZE_BRANCH=true`), synthesize the branch name from the **currently staged files** (or unstaged if none are staged yet):

```bash
# Show staged files
CHANGED_FILES=$(git diff --cached --name-only 2>/dev/null || true)

# If nothing is staged, show all changes (staged + unstaged)
if [ -z "$CHANGED_FILES" ]; then
  CHANGED_FILES=$(git diff --name-only 2>/dev/null || true)
fi
```

**Branch name strategy** (in priority order):

1. If `CHANGED_FILES` is empty (no changes), use a generic slug: `feat/no-changes`. Log a warning and proceed to staging.
2. If any file matches `src/<module>/` or `tests/<module>/`, extract `<module>` as the slug.
3. If no module pattern matches, take the first changed file: `FIRST_FILE=$(echo "$CHANGED_FILES" | head -1)`, extract basename, strip directories/extensions/special characters to form the slug.
4. Prepend the appropriate prefix based on the commit type (inferred in Step 1 from the scanned rules, or default to `feat`).

```bash
# Example synthesis — adapt to your actual rules:
# If commit format uses "feat:", prefix is "feat"; if "fix:", prefix is "fix", etc.
COMMIT_TYPE="feat"   # Infer from the rules scanned in Step 1, or default to feat.
SLUG="$MODULE_NAME"  # Synthesized from CHANGED_FILES per the strategy above.
git checkout -b "${COMMIT_TYPE}/${SLUG}"
```

## 4. Stage Everything

```bash
git add -A
```

Check the status to ensure nothing unexpected is included:

```bash
git status
```

If there are untracked files that shouldn't be committed, ask the user or exclude them via `.gitignore` before proceeding.

**Check for no changes:** If `git status --porcelain` returns nothing after staging, there are no changes to commit. Report this and stop — do not create an empty commit.

## 5. Commit

Craft the commit message based on the commit message format identified in Step 1.

### 5.1 Determining the commit type

- If the rules specify Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, etc.), use the prefix that best matches the bulk of changes.
- For documentation-only changes: `docs:`
- For bug fixes: `fix:`
- For new skills/features: `feat:`
- For maintenance: `chore:`

### 5.2 Crafting the message

Keep the subject line under 72 characters. Summarize the change in imperative mood.

```bash
git commit -m "<type>: <subject>"
```

Example:
```bash
git commit -m "docs: correct branch naming in AGENTS.md"
```

If the changes span multiple categories, choose the most impactful one and add a body line for additional clarity:

```bash
git commit -m "<type>: <subject>

Additional context if needed."
```

## 6. Push

Push the branch to the remote — this is the point of the skill.

```bash
git push origin HEAD
```

**Handle push failures:**

| Failure | Action |
|---------|--------|
| Remote not configured | Report error and suggest adding a remote (`git remote add origin <url>`) |
| Permission denied | Report error and suggest checking SSH keys or token permissions |
| Merge conflicts | Report error and suggest rebasing (`git rebase origin/main`) before pushing |
| Other errors | Report error and stop. Do not proceed to PR creation |

## 7. Check for Existing PR

Before creating a new PR, check whether one already exists:

```bash
EXISTING_PR_URL=$(gh pr list --head "$(git branch --show-current)" --base main --state open --json url --jq '.[0].url' 2>/dev/null || true)
```

- **If a URL is returned:** Do **not** create a new PR. Extract the PR number and output it:
  ```bash
  EXISTING_PR_NUMBER=$(echo "$EXISTING_PR_URL" | grep -oP '/pull/\K\d+')
  echo "PR_NUMBER=$EXISTING_PR_NUMBER"
  echo "EXISTING_PR=$EXISTING_PR_URL"
  ```
  Then skip to Step 9.

- **If no URL is returned:** Proceed to Step 8.

## 8. Open a Pull Request

**Verify `gh` CLI is authenticated:**

```bash
gh auth status 2>&1
```

If authenticated, continue. If not, report `gh` authentication failure and instruct the user to run `gh auth login`.

### 8.1 Synthesize PR Title

- Follow the commit message format from Step 1 if specific rules exist.
- Use the same type prefix and summary as the commit message.

Example: `docs: correct branch naming in AGENTS.md`

### 8.2 Synthesize PR Body

Pull the PR body from the project's template. The template path is defined by Rule 5.4. The standard location is `.github/PULL_REQUEST_TEMPLATE.md` — try it first. If it does not exist, check `.github/gh-pull_request_template.md`, then `.github/PULL_REQUEST_TEMPLATE.md` in other common variants. If none exist, generate a minimal body from the available context.

```bash
# Try common template locations in order
TEMPLATE_PATHS=(
  ".github/PULL_REQUEST_TEMPLATE.md"
  ".github/gh-pull_request_template.md"
)

TEMPLATE_FILE=""
for path in "${TEMPLATE_PATHS[@]}"; do
  if [ -f "$path" ]; then
    TEMPLATE_FILE="$path"
    break
  fi
done
```

**If a template file was found** (`TEMPLATE_FILE` is set), read it as the base body and fill in each section with the actual content:

- Replace inline placeholders (e.g., `<fill-in>`, `<!-- ... -->`, `[ ]` checkboxes).
- If the template has no fillable placeholders, append the content as a "Details" section at the end.

Construct the body from available context:

```markdown
<TEMPLATE_FILE_CONTENT>

---

### Details

**Commit(s):** \`<commit subject>\`

**Changed files:** \`<comma-separated changed filenames, shortened>\`

**Rule 5.4:** PR template at \`<TEMPLATE_FILE>\` was used. All sections filled.
```

**If no template file exists**, generate a minimal body from the commit and file context:

```markdown
**Commit:** \`<commit message>\`

**Changed files:** \`<comma-separated changed filenames>\`

### 8.3 Create the PR

**If `TEMPLATE_FILE` was found in Step 8.2:** Use it as the PR body (Rule 5.4 compliance). Fill in every section from the template — do not leave any blank. If a section is not applicable, write `N/A`.

```bash
# Fill in the template and pass it as the PR body
# Option A: gh pr create accepts a file path via --body-file
gh pr create \
  --title "<synthesized-title>" \
  --body-file "$TEMPLATE_FILE" \
  --base main \
  --assignee avoidwork \
  --label "<inferred-label>"
```

If `gh pr create` does not support `--body-file` in your version, use stdin:

```bash
gh pr create \
  --title "<synthesized-title>" \
  --body "@-" \
  --base main \
  --assignee avoidwork \
  --label "<inferred-label>" < "$TEMPLATE_FILE"
```

**If no template file exists:** Generate a minimal body and pass it directly:

```bash
MINIMAL_BODY="$(printf "%s" "* **Commit:** \`<commit message>\`

* **Changed files:** \`<comma-separated changed filenames>\`")

gh pr create \
  --title "<synthesized-title>" \
  --body "$MINIMAL_BODY" \
  --base main \
  --assignee avoidwork \
  --label "<inferred-label>"
```

**Label selection:** Infer from the commit type determined in Step 5.1:
- If the subject starts with or implies a bug fix → `--label "bug"`
- Otherwise → `--label "feature"`
- If multiple types are present, use `"bug"` only if any commit is a fix; otherwise `"feature"`

**After creating the PR**, extract and output the number:

```bash
NEW_PR_NUMBER=$(gh pr list --head "$(git branch --show-current)" --base main --state open --json number --jq '.[0].number' 2>/dev/null || true)
echo "PR_NUMBER=$NEW_PR_NUMBER"
```

If `NEW_PR_NUMBER` is empty, fall back to extracting from the PR URL:

```bash
NEW_PR_URL=$(gh pr view --json url --jq '.url' 2>/dev/null || true)
echo "PR_URL=$NEW_PR_URL"
```

## 9. Verification

- Confirm the PR was created successfully (or note the existing PR URL from Step 7).
- Output the PR URL for the user so they can review it.

Example output:
```
PR created: https://github.com/<owner>/<repo>/pull/<number>
```