---
name: purge-branches
description: Deletes all local git branches except 'main'. Works in any git repository — no npm or package.json dependency.
license: BSD 3-Clause
compatibility: Requires git CLI. Must be run from a git repository root.
metadata:
  agent: coding
---

# Purge Branches

Delete all local branches except `main`. Pure git — no npm, no package.json required.

## Pre-flight Check

Check current state and confirm we're not on a branch that will be deleted:

```bash
# Check current branch
CURRENT_BRANCH=$(git branch --show-current)
echo "Current branch: $CURRENT_BRANCH"

# Show branches that will be deleted
echo ""
echo "Branches that will be deleted:"
git branch --format='%(refname:short)' | grep -v '^main$'
BRANCH_COUNT=$(git branch --format='%(refname:short)' | grep -cv '^main$')
echo ""
echo "Total: $BRANCH_COUNT branches to delete"

if [ "$BRANCH_COUNT" -eq 0 ]; then
  echo "No branches to purge."
  exit 0
fi
```

## Execute

Delete all branches except `main`:

```bash
# Delete all branches except main (safely, one at a time)
git branch --format='%(refname:short)' | grep -v '^main$' | while read -r branch; do
  git branch -D "$branch" 2>/dev/null && echo "Deleted: $branch" || echo "Skipped (checkout error): $branch"
done
```

**Handle failure:** If any branch fails to delete (e.g., it's checked out), the loop continues — only successfully deleted branches are counted. Report any failures at the end.

## Report

```bash
# Count remaining branches (excluding main)
REMAINING=$(git branch --format='%(refname:short)' | grep -cv '^main$')
echo ""
echo "Purge complete."
echo "Remaining branches: $REMAINING"
```

---
