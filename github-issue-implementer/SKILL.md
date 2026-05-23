---
name: github-issue-implementer
description: Fetch a GitHub issue by URL, create a feature branch from it, implement the described task strictly, push the branch, and open a pull request with a list of any deviations from the original plan. Use when the user provides a GitHub issue URL to implement, or asks to work on / close / implement a GitHub issue from a URL.
---

# GitHub Issue Implementer

Use this skill to turn a GitHub issue URL into a completed implementation with a pull request.

## Quick Start

```
<user provides a GitHub issue URL>
```

1. Parse the issue number from the URL
2. Fetch: `gh issue view <url> --json title,body`
3. Create branch `feat/<number>`, implement, push, open PR

See [REFERENCE.md](REFERENCE.md) for detailed branch naming and PR format.

## Prerequisites

Before starting, verify the environment:

```bash
# 1. Must be inside a Git repository
if ! git rev-parse --git-dir > /dev/null 2>&1; then
  echo "Not in a Git repository"; exit 1
fi

# 2. No uncommitted changes
if ! git diff --quiet --exit-code; then
  echo "Uncommitted changes detected"; exit 1
fi
if ! git diff --cached --quiet --exit-code; then
  echo "Unstaged changes detected"; exit 1
fi

# 3. On the default branch (usually main, but detect it)
DEFAULT_BRANCH=$(git remote show origin 2>/dev/null | grep 'HEAD branch' | cut -d' ' -f5)
: "${DEFAULT_BRANCH:=main}"            # fallback if detection fails
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
if [ "$CURRENT_BRANCH" != "$DEFAULT_BRANCH" ]; then
  echo "Not on default branch ($DEFAULT_BRANCH) — currently on $CURRENT_BRANCH"; exit 1
fi

# 4. Up to date with remote
git fetch origin "$DEFAULT_BRANCH"
BEHIND=$(git rev-list --count "HEAD..origin/$DEFAULT_BRANCH")
if [ "$BEHIND" -gt 0 ]; then
  echo "Branch is $BEHIND commit(s) behind origin/$DEFAULT_BRANCH — pull first"; exit 1
fi

echo "All prerequisites met."
```

## Workflow

### 1. Fetch the Issue

```bash
gh issue view <issue-url> --json title,body
```

Extract `title` and `body`. The body contains implementation instructions.

### 2. Create a Branch

Use `feat/<issue-number>` (e.g., issue #42 → `feat/42`). If that branch exists locally or remotely, append `-1`, `-2`, etc.

```bash
git checkout -b feat/42
```

See [REFERENCE.md](REFERENCE.md#branch-naming) for conflict resolution.

### 3. Implement the Task

Follow the issue body strictly:

- Make real, working changes — no placeholders or TODOs
- Adhere to existing code conventions
- Run tests, lint, and verification steps the repo expects
- Commit with descriptive messages

### 4. Push the Branch

```bash
git push -u origin feat/42
```

### 5. Open a Pull Request

**PR title:**

```
Implement <original title> - closes #<number>
```

**PR body:** List any changes that deviated from the original plan, or state "No deviations from the original plan."

```bash
gh pr create \
  --title "Implement <title> - closes #<number>" \
  --body "<deviations>"
```

### 6. Confirm

```bash
gh pr view --web
```

## Notes

- `gh` CLI must be installed and authenticated
- Default branch assumed to be `main`; adjust if different
- Always push before creating the PR
