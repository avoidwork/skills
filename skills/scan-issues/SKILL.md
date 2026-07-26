---
name: scan-issues
description: Scans GitHub for open issues labeled 'approved' (excluding those also labeled 'in progress'), then processes each sequentially via fix-issue. Runs hourly.
license: BSD 3-Clause
compatibility: Requires gh CLI authenticated with the avoidwork/madz repo. Must be run from the project root.
metadata:
  agent: coding
---

# Scan & Fix Issues

An autonomous issue scanner and fixer. Finds approved, unassigned issues and processes them **one at a time** through the `fix-issue` pipeline.

## Workflow

### 1. Scan for Issues

```bash
gh issue list --state open --label approved --json number,title,url,labels --repo avoidwork/madz
```

This fetches all open issues that have the `approved` label. **Note:** The `in progress` label check is handled by `fix-issue` in its Step 3, so we don't filter here — let each issue's own validation decide.

If no issues match, report `No approved, unassigned issues found.` and exit cleanly.

### 2. Parse Results

Extract each issue's `number`, `title`, `url`, and `labels` from the JSON array. Sort the remaining issues by `number` ascending (oldest first) to ensure sequential processing.

### 3. Process Issues Sequentially

**Process one issue at a time.** Do NOT spawn subagents or parallelize — each issue must be processed sequentially to avoid label conflicts and race conditions:

For each issue in the sorted list:
1. **Invoke fix-issue** as a chain instruction:
   ```
   fix-issue <ISSUE_NUMBER>
   ```
2. **Wait for completion.** Do not proceed to the next issue until the current one is done.
3. **If fix-issue succeeds:** Log the issue number and PR number (if available).
4. **If fix-issue fails:** Log the error, skip the issue, and continue with the next. Never abort the queue over a single failure.

**Rate limiting:** If GitHub API returns 403 rate limit, wait 60 seconds and retry the current issue once.

**Timeout handling:** Each fix-issue run can take 30-60 minutes (via create-feature). If a run exceeds 60 minutes, report a timeout warning but do not abort — the issue may still be processing.

### 4. Report

After all issues are processed, produce a structured summary:

```
Scan complete.
✅ Fixed: #42 — "Crash on empty input"
✅ Fixed: #57 — "Missing error boundary"
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

## Cron Schedule

This skill is designed to run hourly. When triggered via cron, execute the full workflow above without requiring user input.

---
