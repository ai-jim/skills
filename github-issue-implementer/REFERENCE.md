# Reference: GitHub Issue Implementer

## Branch Naming

### Convention

```
feat/<issue-number>
```

Example: issue #42 → `feat/42`.

### Conflict Resolution

If `feat/<issue-number>` already exists **locally** or **remotely**, append a uniqueness suffix (`-1`, `-2`, etc.):

```
feat/42     # taken →
feat/42-1   # taken →
feat/42-2   # use this
```

### Checking for Existing Branches

```bash
# Check locally
git branch --list feat/42

# Check remotely
git branch -r --list origin/feat/42
```

| Condition | Result |
|-----------|--------|
| `feat/<N>` does not exist locally or remotely | Use `feat/<N>` |
| `feat/<N>` exists locally or remotely | Append `-1`, `-2`, etc. |

## PR Format

### Title

```
Implement <original GitHub Issue title> - closes #<issue-number>
```

### Body

Document any changes made during implementation that deviated from the original plan. For example:

```
Deviations from the original plan:
- Used sqlite3 instead of PostgreSQL for local dev simplicity
- Skipped the email notification step (no SMTP credentials available in this repo)
- Added an extra validation step for duplicate entries not mentioned in the issue
```

If no deviations were necessary:

```
No deviations from the original plan.
```

### Command

```bash
gh pr create \
  --title "Implement <title> - closes #<number>" \
  --body "<deviations>"
```

### Deviation Tracking

Before opening the PR, review what was actually implemented vs. what the issue asked for:

```bash
git log origin/main..HEAD --oneline
git diff origin/main...HEAD --stat
```

Compare the changes against the issue body and list any intentional or unintentional differences.

## Notes

- The `gh` CLI must be authenticated (`gh auth status` to verify)
- If the repo uses a different default branch (e.g., `master`, `develop`), adjust commands accordingly
- If the issue body references files or APIs that don't exist, adapt to what's available and note the deviation
