# AGENTS.md

Rules and principles for agents working on **this** project.

---

## 1. Core Rules

### 1.0 Document Conventions

When updating this document, append new information or sections. Do NOT delete or overwrite existing content unless explicitly directed. Always ask before making structural changes. When in doubt, keep it.

### 1.1 Forbidden Patterns

The following are **strictly prohibited**:

- Hardcoded secrets, API keys, or credentials.
- Bypassing the `gh` CLI authentication when creating issues, PRs, or repo operations.
- Creating issues or PRs without a clear description and actionable details.
- Overwriting SKILL.md files without preserving the author's original intent.
- Assuming a label exists — always verify label presence before acting on it.

### 1.2 Security Rules

- Never store or log plaintext tokens, keys, or credentials.
- Validate all GitHub API interactions through `gh` CLI — never craft raw HTTP requests with secrets.
- Do not expose repository secrets in logs, issue bodies, or commit messages.
- When auditing code, report OWASP Top 10-style issues but never expose sensitive findings in public issues unless explicitly authorized.

### 1.3 Git Operations

- **Never rebase under any circumstance without explicit agreement from the user.** Never assume your decision is correct.
- **Never push to any branch without explicit user approval.** Git changes (checkout, reset, revert, amend) are local operations — do not auto-push. Always ask "Push to remote?" before running `git push`.
- Never force push.

### 1.4 Core Principles

- **DRY**: Extract repeated logic (e.g., GitHub API calls, issue/PR creation) into shared patterns. Centralize conventions in this document.
- **KISS**: Keep skills focused and instructions clear. If a SKILL.md requires more than a few levels of nesting, break it into separate skills.
- **YAGNI**: Do NOT build skills or abstractions not required by the current project scope. Ad-hoc solutions are acceptable as long as they serve a present requirement.
- **Single Responsibility**: Each skill must do one thing well — issue management, git workflow, release, or audit.
- **Open/Closed**: Extend pipelines by chaining existing skills — not by modifying their internals.

---

## 2. Project Context

A collection of Agent Skills (`skills/<name>/SKILL.md`) for building, maintaining, and releasing AI harness applications and OSS projects. These skills power an autonomous development pipeline — from issue creation to tagged releases — via the opencode Agent Skill protocol.

### 2.0 Expected Project Layout

```
AGENTS.md           # Documented project rules (this file)
README.md           # Project overview and skill catalog
LICENSE             # BSD 3-Clause
.github/
  PULL_REQUEST_TEMPLATE.md   # PR template (must be used when present)
skills/
  audit-code/
    SKILL.md
  audit-skill/
    SKILL.md
  audit-sys-prompt/
    SKILL.md
  commit-push/
    SKILL.md
  create-feature/
    SKILL.md
  create-issue/
    SKILL.md
  fix-issue/
    SKILL.md
  git-tag/
    SKILL.md
  purge-branches/
    SKILL.md
  release-madz/
    SKILL.md
  restructure-code/
    SKILL.md
  scan-issues/
    SKILL.md
  task-queue/
    SKILL.md
  update-pr/
    SKILL.md
  update-semver/
    SKILL.md
```

Misc details here.

### 2.1 Quick Commands

| Command        | Purpose |
|----------------|---------|
| `npm run docker:release:all` | Build and push all Docker images for the madz project (used by `release-madz`) |
| `npm run purge-branches`     | Delete all local branches except `main` (used by `purge-branches`) |

---

## 3. Skill Conventions

### 3.1 Skill Structure

Each skill is a self-contained directory under `skills/<name>/` containing:

- **SKILL.md** — YAML frontmatter with `name`, `description`, `license`, and `compatibility`; followed by step-by-step workflow instructions.
- Optional: `scripts/`, `references/`, `assets/` directories with supporting content.

### 3.2 YAML Frontmatter

```yaml
---
name: <skill-name>
description: <one-line description>
license: BSD 3-Clause
compatibility: <opencode version or agent framework>
---
```

### 3.3 Skill Naming

- Use kebab-case for skill directory names and frontmatter names (e.g., `create-issue`, `scan-issues`).
- Names must be verb-noun or adjective-noun patterns describing the skill's primary action.

### 3.4 Content Quality

- Every skill must have a clear **Purpose** or **Overview** section.
- Steps must be numbered and actionable (imperative verb first).
- Include error handling guidance and output format specification.
- Skills that modify GitHub state (issues, PRs, labels) must reference the relevant conventions from this document (Section 5).

---

## 4. Opencode Protocol Conventions

### 4.1 Agent Skill Format

Skills must be compliant with the opencode Agent Skill protocol:

- SKILL.md must start with YAML frontmatter.
- Content is structured with Markdown headings (`##`, `###`) for progressive disclosure.
- Skills that require multi-step pipelines should reference other skills by name (e.g., chains `create-issue` → `audit-code`).

### 4.2 Skill Dependency Chains

Skills may chain:

```
create-issue ← audit-code, restructure-code
fix-issue → create-feature
create-feature → commit-push, update-pr
fix-issue → scan-issues
update-semver → commit-push
```

When chaining, the orchestrating skill must handle all error cases and provide a structured summary.

### 4.3 State Persistence

Skills that require multi-turn interaction (e.g., `audit-code`, `restructure-code`) must use state persistence (file-based or environment) to resume across responses.

---

## 5. Git Conventions

### 5.1 Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add audit-code skill for security scanning
fix: correct label validation in fix-issue
docs: update skill catalog in README
test: add unit tests for create-feature pipeline
chore: update dependencies in package.json
```

### 5.2 Branching

- Main branch is `main`.
- Feature branches: `feat/<short-desc>`, `fix/<short-desc>`, `docs/<short-desc>`, `chore/<short-desc>`.
- Never commit directly to `main`. Always create a feature branch first, then open a PR targeting `main`.

### 5.2.1 Agent Workflow

When auditing or modifying AGENTS.md (or any file):

1. Create a feature branch: `git checkout -b docs/<short-desc>`.
2. Make changes and commit on the feature branch.
3. Push the feature branch and open a PR with `gh pr create --base main`.
4. Never commit or push directly to `main` or `master`.

### 5.3 Code Review

- All changes require at least one passing check (lint, format, or opencode skill validation).
- No merging without passing CI checks.
- PR descriptions should reference relevant skills, issues, or project conventions.

### 5.4 Pull Request Templates

If a `.github/PULL_REQUEST_TEMPLATE.md` file exists, it MUST be used when creating PRs. Fill out every section — do not leave any section blank. If a section does not apply, write `N/A` rather than skipping it.

---

## 6. Operational Rules

Skills require the following operational context and constraints.

### 6.1 GitHub Conventions

#### 6.1.1 Issue Labels

- Use only `bug` and `feature` — never `enhancement`, `improvement`, `question`, or similar.
- `approved` — issue has been reviewed and approved for work.
- `in progress` — work is underway (set by `fix-issue`).

#### 6.1.2 Issue Creation

When creating issues (`create-issue`):

1. Synthesize a clear title and description from the user input.
2. Categorize as `bug` or `feature`; apply the correct label.
3. Include actionable details: codebase context, reproduction steps (if bug), and impact.

#### 6.1.3 Approval Gate

`fix-issue` validates that an issue has the `approved` label before chaining to `create-feature`. If the label is missing, the skill reports the issue as unapproved and stops.

### 6.2 Requirements

All skills share these prerequisites:

| Requirement | Details |
|-------------|---------|
| **Node.js** | 24+ (ECMAScript modules) |
| **npm** | For dependency management and scripts |
| **git CLI** | Configured with remote access |
| **gh CLI** | Authenticated with the target repository |
| **openspec CLI** | Required by `create-feature` for spec generation |

Additional per-skill requirements:

- **release-madz**: Docker CLI with remote access.
- **scan-issues**: Cron scheduler for hourly execution.
- **audit-sys-prompt**: Capable LLM for evaluation.

### 6.3 Session Timeouts

- `create-feature` requires 30–60 minute timeout for the full pipeline (spec → implement → test → PR → archive).
- `audit-code` and `restructure-code` run sequentially per directory and use state persistence to resume.

---

## 7. Session Learnings

Discovery notes about the codebase and opencode skill mechanics.

### 7.1 AGENTS.md is scanned via `scanAgents`

Skills that perform git operations (e.g., `commit-push`, `update-pr`) scan AGENTS.md for project rules before acting. The file must be kept up to date with all conventions.

### 7.2 README.md is the source of truth for skill catalog

The README.md contains the full skill catalog with descriptions, categorization (Issue Management, Feature Development, Git & PR Workflow, Release Management, Code Quality & Auditing), and pipeline diagrams. When adding a new skill, update both README.md and AGENTS.md.
