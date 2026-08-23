---
name: release-madz
description: Builds and pushes all Docker images for the madz project by running npm run docker:release:all. Runs the build as a foreground process — it takes a few minutes. Reports completion status to the user.
license: BSD 3-Clause
compatibility: Requires Node.js 24+, npm, Docker CLI, and git with remote access. Must be run from the project root directory.
metadata:
  agent: coding
---

# Release Madz

> **⚠️ EXECUTION RULE:** Every code block in this skill is a shell command to **execute**. Do not print them as text, explain them, or treat them as examples — run them directly.

> **⚠️ CRITICAL: This skill is idempotent but NOT retryable.**
> Docker image tags are immutable — once `1.23.0` is pushed, it cannot be overwritten.
> A second run with the same version will fail. You cannot re-run this skill to check
> status or retry a failed run. The first execution is the only execution.
> **If the build fails, you must fix the underlying issue, bump the version, and run
> again — not re-run this skill against the same tag.**

Build and push all Docker images for the madz project. This is the final step — the moment of truth.

## 0. Pre-flight Check

Before doing anything, verify the Docker image has not already been pushed. Docker image tags are immutable — a second run with the same version will fail.

```bash
# Extract version reliably using node (avoids matching nested "version" keys)
TAG_VERSION=$(node -e "console.log(require('./package.json').version)")
DOCKER_USER=${DOCKER_USER:-avoidwork}

# Verify Docker is running
docker info >/dev/null 2>&1
if [ $? -ne 0 ]; then
  echo "ERROR: Docker daemon is not running. Start Docker and try again."
  exit 1
fi

# Verify Docker registry authentication
docker info 2>&1 | grep -q "Login Status"
if [ $? -ne 0 ]; then
  echo "WARNING: Docker registry authentication status unclear. Ensure you're logged in."
fi

# Check disk space (Docker builds need ~2GB free)
AVAILABLE_SPACE=$(df -m . | tail -1 | awk '{print $4}')
if [ "$AVAILABLE_SPACE" -lt 2048 ]; then
  echo "WARNING: Less than 2GB disk space available. Build may fail."
fi

# Check if the image already exists in the registry
# Note: docker manifest inspect requires registry access; may fail if registry is unreachable
docker manifest inspect "${DOCKER_USER}/madz:${TAG_VERSION}" >/dev/null 2>&1 && echo "EXISTS" || echo "NOT_FOUND"
```

If the image already exists in the registry, **stop and ask**. Do not proceed — the release was already pushed.

Also verify the git tag exists locally (for context, not as the primary check):

```bash
git tag -l "${TAG_VERSION}"
```

If the git tag is missing, warn the user but do not block — the tag may have been pushed separately.

## 1. Run the Build

Run `npm run docker:release:all` as a **foreground process**. This takes a few minutes. The skill description says "foreground" — do not background it.

```bash
npm run docker:release:all
```

## 2. Wait for Completion

Wait for the process to exit, then check the exit code:

- **exit code 0** — Success. All images built and pushed.
- **non-zero exit code** — Failure. Capture the last 10 lines of output for context.

**Note on retryability:** The warning at the top of this skill says the skill is "NOT retryable" — this means you cannot re-run this skill against the **same tag** (Docker image tags are immutable). If the build fails, you can report the failure and stop. The user must fix the underlying issue, bump the version, and run the skill again with the new version. This step is about **handling the failure of the current run**, not about retrying the same tag.

## 3. Report

Print a final summary using actual variable values:

```
Release complete.
Status: SUCCESS / FAILED
Images built and pushed.
Version: $TAG_VERSION
Docker user: $DOCKER_USER
```

If the build fails, include the last 10 lines of output for context.

## Examples

```
User: release-madz

Agent: Pre-flight check...
  Version: 1.45.1
  Docker daemon: running
  Disk space: 15GB available
  Image exists: NOT_FOUND

Running build...
  npm run docker:release:all
  [build output...]

Release complete.
Status: SUCCESS
Images built and pushed.
Version: 1.45.1
Docker user: avoidwork
```

## Gotchas

- **This skill is NOT retryable.** Docker image tags are immutable. Once `1.23.0` is pushed, it cannot be overwritten. A second run with the same version will fail. If the build fails, fix the underlying issue, bump the version, and run again — do not re-run this skill against the same tag.
- **Always check disk space.** Docker builds need ~2GB free. If disk space is insufficient, the build will fail partway through.
- **Verify Docker is running.** The pre-flight check verifies the Docker daemon is accessible. If it's not, start Docker before proceeding.
- **Registry auth must be configured.** If `docker info` shows unclear login status, the push will fail. Run `docker login` first.

---
