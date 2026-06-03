---
name: autonomous-coding-agent
description: >-
  Autonomously discover prioritized issues in the current repository from
  GitHub or GitLab, implement them strictly, push a feature branch, and open a
  pull/merge request with a list of any deviations from the original plan. Use
  when asked to autonomously work on issues in the current repository.
---

# Autonomous Coding Agent

Automatically discover, implement, and open pull/merge requests for prioritized issues in the current repository.

## Quick Start

1. Detect the repo from `git remote get-url origin` and determine platform (GitHub vs GitLab)
2. List open prioritized issues, pick one, and check for conflicts with open PRs/MRs
3. Fetch issue details with `gh issue view` or `glab issue view`
4. Create branch `feat/<number>`, implement, push, open PR/MR

See [REFERENCE.md](REFERENCE.md) for detailed branch naming and PR/MR format.

## Platform detection

Detect the platform from the `origin` remote and derive the repo slug:

```bash
REMOTE_URL=$(git remote get-url origin 2>/dev/null) || {
  echo "No 'origin' remote found"; exit 1
}

if echo "$REMOTE_URL" | grep -q gitlab; then
  CLI="glab"
  ISSUE_CMD="issue"
  MR_CMD="mr"
elif echo "$REMOTE_URL" | grep -q github; then
  CLI="gh"
  ISSUE_CMD="issue"
  MR_CMD="pr"
else
  echo "Unknown platform — remote must be from github.com or gitlab.com"; exit 1
fi

# Derive repo slug (e.g. "owner/repo")
if echo "$REMOTE_URL" | grep -q "github.com"; then
  REPO=$(echo "$REMOTE_URL" | sed -n 's|.*github.com[/:]\(.*\)\.git|\1|p')
elif echo "$REMOTE_URL" | grep -q "gitlab"; then
  REPO=$(echo "$REMOTE_URL" | sed -n 's|.*gitlab.com[/:]\(.*\)\.git|\1|p')
fi
```

Verify the CLI is authenticated:

```bash
$CLI auth status
```

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

### 1. Discover and Select an Issue

List open prioritized issues on the repo:

```bash
$CLI $ISSUE_CMD list --repo "$REPO" --label prioritized --json number,title --limit 20 2>/dev/null || \
  $CLI $ISSUE_CMD list --repo "$REPO" --json number,title --limit 20
```

If there are no prioritized issues, fall back to listing all open issues:

```bash
$CLI $ISSUE_CMD list --repo "$REPO" --json number,title --limit 20
```

Pick the highest-priority issue from the list (typically the first listed).

Before proceeding, check that the issue doesn't already have an open PR/MR:

```bash
ISSUE_NUMBER=<selected number>
OPEN_BRANCHES=$($CLI $MR_CMD list --repo "$REPO" --state open --json headRefName 2>/dev/null | jq -r '.[].headRefName // empty')
for BRANCH in $OPEN_BRANCHES; do
  if echo "$BRANCH" | grep -q "feat/$ISSUE_NUMBER"; then
    echo "Issue #$ISSUE_NUMBER already has an open PR/MR — skipping. Try the next issue."
    exit 1
  fi
done
```

If safe, proceed to fetch the full issue details.

### 2. Fetch the Issue Details

**GitHub:**
```bash
gh issue view "$ISSUE_NUMBER" --repo "$REPO" --json title,body
```

**GitLab:**
```bash
glab issue view "$ISSUE_NUMBER" --repo "$REPO" --output json
```

Extract `title` and `body`. The body contains implementation instructions.

### 3. Create a Branch

Use `feat/<issue-number>` (e.g., issue #42 → `feat/42`). If that branch exists locally or remotely, append `-1`, `-2`, etc.

```bash
git checkout -b feat/42
```

See [REFERENCE.md](REFERENCE.md#branch-naming) for conflict resolution.

### 4. Implement the Task

Follow the issue body strictly:

- Make real, working changes — no placeholders or TODOs
- Adhere to existing code conventions
- Run tests, lint, and verification steps the repo expects
- Commit with descriptive messages

### 5. Push the Branch

```bash
git push -u origin feat/42
```

### 6. Open a Pull Request / Merge Request

**PR title / MR title:**

```
Implement <original title> - closes #<number>
```

**Body:** List any changes that deviated from the original plan, or state "No deviations from the original plan."

**GitHub:**
```bash
gh pr create \
  --title "Implement <title> - closes #<number>" \
  --body "<deviations>"
```

**GitLab:**
```bash
glab mr create \
  --title "Implement <title> - closes #<number>" \
  --description "<deviations>"
```

### 7. Confirm

**GitHub:**
```bash
gh pr view --web
```

**GitLab:**
```bash
glab mr view --web
```

## Notes

- `gh` (GitHub) and/or `glab` (GitLab) CLI must be installed and authenticated
- Default branch assumed to be `main`; adjust if different
- Always push before creating the PR/MR
