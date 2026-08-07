# Skills

Agent Skills for software development — issue tracking, feature implementation, git workflows, and release management.

Each skill is a self-contained, spec-compliant Agent Skill (`SKILL.md`) that orchestrates a specific workflow. Skills chain together to form larger pipelines.

## Quick Start

```bash
# Create an issue
create-issue "Fix crash on empty input"

# Fix an approved issue
fix-issue 42

# Scan for all approved issues and fix them
scan-issues
```

## Skill Catalog

### Issue Management

Skills for creating, tracking, and resolving issues.

| Skill | Description |
|-------|-------------|
| **create-issue** | Receives a description, synthesizes a title and description, categorizes as `bug` or `feature`, creates a GitHub issue, audits the codebase, and appends findings. |
| **fix-issue** | Accepts an issue ID, validates approval (`approved` label), marks it `in progress`, and chains to `create-feature` for implementation. |
| **scan-issues** | Finds open issues labeled `approved` (excluding `in progress`), creates a git worktree for each, and processes them sequentially via `fix-issue`. Designed for hourly cron execution. |

### Feature Development

Skills for implementing features from specification to shipped code.

| Skill | Description |
|-------|-------------|
| **create-feature** | Orchestrates the full feature lifecycle: receives goals, synthesizes specs via OpenSpec, commits specs to PR, applies tasks, audits results, updates the PR, and archives the change. |
| **task-queue** | Accepts a list of tasks (JSON or natural language), executes shell commands sequentially with fail-fast logic, and reports a structured summary. |

### Git & PR Workflow

Skills for managing git operations and pull requests.

| Skill | Description |
|-------|-------------|
| **commit-push** | Automates the complete git workflow: scans project rules, stages all changes, commits with a conventional commit message, pushes to the remote, and opens a Pull Request. Handles branch creation, PR template compliance, and existing PR detection. |
| **update-pr** | Updates an existing PR's title and description by scanning project rules and the PR template, synthesizing both from the full delta between branch and target, then applying changes via `gh api`. |
| **purge-branches** | Deletes all local branches except `main`. Pure git — no npm or package.json dependency. |

### Release Management

Skills for versioning, building, and publishing releases.

| Skill | Description |
|-------|-------------|
| **update-semver** | Audits the delta between HEAD and the current version tag, decides major/minor/patch bump, updates `package.json`, runs `npm i` and changelog generation, triggers `commit-push`, and enables auto-merge. Does NOT create a git tag — that is handled by `git-tag` after the PR merges. |
| **git-tag** | Reads version from `package.json` (or other package manager files), synthesizes a change description from git history, creates an annotated git tag (no `v` prefix), and pushes it to origin. |
| **release-madz** | Builds and pushes all Docker images for the madz project via `npm run docker:release:all`. Runs as a foreground process. |

### Code Quality & Auditing

Skills for analyzing code, skills, and system prompts.

| Skill | Description |
|-------|-------------|
| **audit-code** | Audits each directory in `./src` for bugs, security vulnerabilities (OWASP Top 10), and performance issues. Generates one consolidated GitHub issue per directory via `create-issue`. |
| **restructure-code** | Audits each directory in `./src` for opportunities to restructure for better organization (cohesion, coupling, naming, depth, pattern alignment). Generates one issue per directory via `create-issue`. |
| **audit-skill** | Audits Agent Skills `SKILL.md` files against the protocol specification and best practices. Checks 10 categories: schema violations, naming, description quality, structure, progressive disclosure, file references, best practices, calibration, content quality, and optional directories. |
| **audit-sys-prompt** | Expert system prompt evaluator. Analyzes, rates, and provides actionable feedback for system prompts against a 7-criteria framework. Returns structured machine-readable JSON output. |

## Pipeline Architecture

### Standard Feature Pipeline

```
User: "fix-issue 42"
  ↓
fix-issue (validate approval, set in-progress)
  ↓
create-feature (spec → implement → test → PR → archive)
  ↓
commit-push (stage, commit, push, PR)
  ↓
update-pr (finalize title/description from delta)
  ↓
fix-issue comments on original issue with PR link
```

### Release Pipeline

```
User: "run update-semver"
  ↓
update-semver (bump version, build, changelog, PR)
  ↓
commit-push (stage, commit, push, PR)
  ↓
auto-merge enabled (squash)
  ↓
[PR merges to main]
  ↓
User: "run git-tag"
  ↓
git-tag (create annotated tag, push)
  ↓
[Optional] release-madz (build & push Docker images)
```

## License

BSD 3-Clause

### Autonomous Issue Processing

```
cron (hourly)
  ↓
scan-issues (find approved, unassigned issues)
  ↓
for each issue:
  fix-issue (validate, implement via create-feature)
  ↓
structured summary report
```

### Code Audit Pipeline

```
User: "audit-code"
  ↓
audit-code (sequential directory audit)
  ↓
for each directory:
  scan files for bugs, security, performance issues
  create-issue (one issue per directory)
  update issue body with audit table
  ↓
consolidated report
```

## Skill Dependencies

Skills form a dependency graph:

```
create-feature → commit-push
create-feature → update-pr
fix-issue → create-feature
scan-issues → fix-issue
update-semver → commit-push
audit-code → create-issue
restructure-code → create-issue
```

Skills that operate independently:
- `git-tag` — standalone
- `purge-branches` — standalone
- `task-queue` — standalone
- `audit-skill` — standalone (per-skill invocation)
- `audit-sys-prompt` — standalone (per-prompt invocation)
- `release-madz` — standalone (per-release invocation)

## Requirements

All skills share these prerequisites:

| Requirement | Details |
|-------------|---------|
| **Node.js** | 24+ (ECMAScript modules) |
| **npm** | For dependency management and scripts |
| **git CLI** | Configured with remote access |
| **gh CLI** | Authenticated with the target repository |
| **openspec CLI** | Required by `create-feature` for spec generation |

Additional requirements per skill:
- **release-madz**: Docker CLI, remote access
- **scan-issues**: Cron scheduler (hourly execution)
- **audit-sys-prompt**: Capable LLM for evaluation

## Project Conventions

Skills follow these project conventions:

- **Commit messages**: Conventional Commits (`feat:`, `fix:`, `chore:`, etc.)
- **Branch naming**: `feat/<short-desc>`, `fix/<short-desc>`, `chore/<short-desc>`
- **PR targets**: `main` branch (unless otherwise specified)
- **Issue labels**: `bug` and `feature` only (never `enhancement`, `improvement`, etc.)
- **PR template**: `.github/PULL_REQUEST_TEMPLATE.md` must be used when present
- **AGENTS.md**: Project rules scanned via `scanAgents` before git operations

## Creating New Skills

To create a new skill:

```bash
# Use the create-skill tool or run:
npx create-agent-skill <name> <description>
```

Each skill must include:
- `SKILL.md` with YAML frontmatter (name, description, license, compatibility)
- Step-by-step workflow instructions
- Error handling guidance
- Output format specification
- Optional: `scripts/`, `references/`, `assets/` directories

## License

BSD 3-Clause
