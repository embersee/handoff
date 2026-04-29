---
description: Ship a task as a PR from a clean worktree based on the latest origin
argument-hint: <task description, paste from tester, bug report, etc.>
---

You are running the **handoff** workflow. The user is handing off a task and expects a PR back. They do NOT want a summary of what you did — only the PR URL and the worktree path.

# Task input

$ARGUMENTS

# Steps

## 1. Pre-flight
- Confirm the current directory is inside a git repo (`git rev-parse --is-inside-work-tree`). If not, stop and tell the user.
- Run `gh auth status`. If not authed, stop and ask the user to auth.
- Determine the **base branch**:
  - If `origin/dev` exists, use `dev`.
  - Else use the repo's default branch: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`.
  - If still ambiguous, ask the user once.
- Pick a **slug** from the task (kebab-case, ≤ 4 words, no ticket prefix unless one is in the input).
- Branch name: `philip/<slug>`.
- Worktree path: `../<repo>-<slug>` (sibling of the current repo dir).
- If the branch or worktree path already exists → stop and ask: resume the existing one, or pick a new slug?

## 2. Create the worktree
```bash
git fetch origin <base>
git worktree add -b philip/<slug> ../<repo>-<slug> origin/<base>
cd ../<repo>-<slug>
```
All subsequent steps run inside the new worktree.

## 3. Implement
- For non-trivial tasks: lay out a short plan first and confirm with the user before editing.
- For one-liners or obvious fixes: just go.
- If the repo has tests / lint / typecheck (`package.json` scripts, `Cargo` test, `pytest`, etc.), run them after the change. Block the PR on failure unless the user explicitly said draft.

## 4. Commit
- Use **Conventional Commits** (`fix:`, `feat:`, `chore:`, `refactor:`, `docs:`, `test:`).
- Split into logical commits if the change has distinct parts; otherwise one commit is fine.
- **NEVER** add `Co-Authored-By: Claude` or any author/committer line referencing yourself. The commit is the user's.
- **NEVER** use `--no-verify`, `--force`, or `--amend` on shared commits. If a hook fails, fix the underlying issue and create a new commit.

## 5. Push & open PR
```bash
git push -u origin philip/<slug>
gh pr create --base <base> --title "<task title>" --body "$(cat <<'EOF'
## Summary
- <1–3 bullets describing the change>

## Test plan
- [ ] <how to verify>

## Context
<the user's original task input from $ARGUMENTS, verbatim>
EOF
)"
```
Ready for review by default — no `--draft` unless the user asked.

## 6. Report back
Reply with **only** these two lines, nothing else:

```
<PR URL>
worktree: <absolute path to the worktree>
```

No summary. No "I did X." No emojis. The user reads the diff.

# Failure modes
- Base branch not fast-forwardable from origin → stop and report; never hard-reset.
- Push rejected → stop and report; never `--force`.
- `gh pr create` fails → fall back to printing the compare URL: `https://github.com/<owner>/<repo>/compare/<base>...philip/<slug>`.
- Tests/lint/typecheck fail → stop, report which check failed, ask whether to fix or open as draft.
- Worktree already exists → ask before reusing.

# Notes
- This workflow is for *isolated* work. If the user wants edits in their current branch, do NOT run this — push back and ask.
- The lifecycle of the worktree (cleanup after merge) is the user's responsibility for now.
