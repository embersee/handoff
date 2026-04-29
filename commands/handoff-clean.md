---
description: Sweep handoff worktrees — remove ones whose PR is merged/closed or whose branch has no PR
---

Aggressive cleanup of `handoff/*` worktrees in the **current repo**. Run this from the main worktree (not inside a handoff worktree).

# Steps

## 1. Collect candidates
- `git worktree list --porcelain` to enumerate worktrees.
- Filter to ones whose branch matches `handoff/*`.
- Skip the current worktree if it happens to be a `handoff/*` one (don't saw off the branch you're on).

## 2. For each candidate worktree
- **Uncommitted-changes check:** `git -C <path> status --porcelain`. If non-empty → **SKIP**, log the path. Never clean a worktree with uncommitted work.
- **Unpushed-commits check:** `git -C <path> log @{u}.. --oneline 2>/dev/null`. If non-empty → **SKIP**, log it. The branch has commits the remote doesn't.
- **PR lookup:** `gh pr list --head <branch> --state all --json state,number,url --limit 1`.
- **Decide:**
  | PR state           | Action  |
  |--------------------|---------|
  | `MERGED`, `CLOSED` | remove  |
  | no PR found        | remove  |
  | `OPEN`             | skip    |

## 3. Remove
For each worktree marked for removal:
```bash
git worktree remove <path>
git branch -D <branch>
```
If `git worktree remove` fails (locked, etc.), report the path and continue with the rest. Never use `--force` to override locks without asking.

## 4. Report
Reply with a compact summary, nothing else:

```
Removed:
  - handoff/<slug>  (PR #123 merged)
  - handoff/<slug>  (no PR)

Skipped (uncommitted): <paths or "none">
Skipped (unpushed):    <paths or "none">
Skipped (open PR):     <branch — PR URL>
```

# Rules
- NEVER touch worktrees not matching `handoff/*`.
- NEVER force-remove a worktree with uncommitted or unpushed changes.
- NEVER `--force` past a worktree lock without explicit user confirmation.
- If not inside a git repo, stop and tell the user.
