---
name: handoff
description: Ship a task as an isolated GitHub PR from a clean worktree. Use when the user pastes a bug report, tester feedback, or feature request and wants it shipped as a PR without touching their current working tree. Triggers on "do a handoff", "handoff this", "let's handoff", "/handoff", or when the user describes a task and asks to "make a PR for this", "ship this as a PR", "push this up as a PR". The workflow creates a fresh worktree based on the latest origin/dev (or default branch), implements the change, opens a PR via gh, and reports back only the PR URL + worktree path. Do NOT use when the user wants edits applied to their current branch — this is strictly for isolated PR-back workflow.
---

You are executing the **handoff** workflow. The user has handed off a task and expects a PR back. They do NOT want a summary of what you did — only the PR URL and the worktree path.

# Steps

## 0. Conservative cleanup sweep
Before starting work, sweep `handoff/*` worktrees in the current repo and remove ones whose PR is **merged or closed**. Leave open PRs and no-PR branches alone (the aggressive cleanup is `/handoff-clean`).

- `git worktree list --porcelain` → filter to `handoff/*` branches (skip the current one).
- For each: skip if dirty (`git status --porcelain`) or has unpushed commits. Otherwise `gh pr list --head <branch> --state all --json state --limit 1`.
- If PR state is `MERGED` or `CLOSED` → `git worktree remove <path>` and `git branch -D <branch>`.
- Stay silent unless something was removed; then list removed paths in one line.

## 1. Pre-flight
- Confirm the current directory is inside a git repo (`git rev-parse --is-inside-work-tree`). If not, stop and tell the user.
- Run `gh auth status`. If not authed, stop and ask the user to auth.
- Determine the **base branch**:
  - If `origin/dev` exists, use `dev`.
  - Else use the repo's default branch: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`.
  - If still ambiguous, ask the user once.
- Pick a **slug** from the task (kebab-case, ≤ 4 words, no ticket prefix unless one is in the input).
- Branch name: `handoff/<slug>`.
- Worktree path: `../<repo>-<slug>` (sibling of the current repo dir).
- If the branch or worktree path already exists → stop and ask: resume the existing one, or pick a new slug?

## 2. Create the worktree
```bash
git fetch origin <base>
git worktree add -b handoff/<slug> ../<repo>-<slug> origin/<base>
cd ../<repo>-<slug>
```
All subsequent steps run inside the new worktree.

## 3. Implement
- For non-trivial tasks: lay out a short plan first and confirm with the user before editing.
- For one-liners or obvious fixes: just go.
- If the repo has tests / lint / typecheck, run them after the change. Block the PR on failure unless the user explicitly said draft.

## 4. Commit
- **Conventional Commits** (`fix:`, `feat:`, `chore:`, `refactor:`, `docs:`, `test:`).
- Split into logical commits if the change has distinct parts.
- **NEVER** add `Co-Authored-By: Claude` or any line referencing yourself as author/committer.
- **NEVER** use `--no-verify`, `--force`, or `--amend` on shared commits.

## 5. Push & open PR
```bash
git push -u origin handoff/<slug>
gh pr create --base <base> --title "<task title>" --body "$(cat <<'EOF'
## Summary
- <1–3 bullets>

## Test plan
- [ ] <how to verify>

## Context
<the user's original task input, verbatim>
EOF
)"
```
Ready for review by default. No `--draft` unless asked.

## 6. Report back
Reply with **only** these two lines:

```
<PR URL>
worktree: <absolute path to the worktree>
```

No summary, no "I did X", no emojis.

# Failure modes
- Base not fast-forwardable from origin → stop and report; never hard-reset.
- Push rejected → stop and report; never `--force`.
- `gh pr create` fails → print compare URL `https://github.com/<owner>/<repo>/compare/<base>...handoff/<slug>`.
- Tests/lint/typecheck fail → stop, report which check, ask whether to fix or open as draft.
- Worktree exists → ask before reusing.

# When NOT to use
- User wants edits in their current branch → push back, don't run this.
- User is in a non-git directory → stop.
