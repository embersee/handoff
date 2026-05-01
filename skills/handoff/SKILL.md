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
  - If the user explicitly named a base in the task input (e.g. "handoff from main", "off of release/v2", "based on staging") → use that branch.
  - Otherwise → use the **currently checked-out branch** (`git branch --show-current`). Default is "branch off whatever I'm on right now."
  - If HEAD is detached or there's no current branch → stop and ask which base to use.
  - Verify the base exists on origin (`git ls-remote --exit-code --heads origin <base>`). If not, stop and ask.
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

### Allowed vs forbidden commands during verification
- **Allowed (one-shot, must terminate):** `build`, `test`, `lint`, `typecheck`, `tsc`, `format --check`, single-run scripts.
- **FORBIDDEN — never run:** any long-running / watch / server command. This includes but is not limited to:
  - `npm run dev`, `pnpm dev`, `yarn dev`, `bun dev`
  - `next dev`, `vite`, `vite dev`, `astro dev`, `nuxt dev`, `remix dev`
  - `*--watch`, `*-w` flag variants, `tsc --watch`, `jest --watch`
  - `npm start`, `pnpm start`, `yarn start` (when they boot a server)
  - `cargo watch`, `cargo run` against a server target
  - Anything that does not exit on its own
- If verifying behavior **requires** a running server, stop and ask the user — do not start one in the background.
- Before running any script you're unsure about, inspect `package.json` (or equivalent) to confirm it terminates. If in doubt, skip it and note this in the PR's Test plan section.

### Process hygiene
- If you start ANY background process during the workflow (even accidentally — e.g. a build step that spawns a watcher), you MUST terminate it before reporting back. Track every PID you spawn and `kill` it at the end.
- Before the final reply, run `jobs -l` (or equivalent) and confirm there are no leftover processes from this session.

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
<3–4 sentence rewrite of the user's task input — what the problem is and what this PR does about it. Neutral PR-prose, not a quote of the prompt. Strip filler ("hey claude", "can you"), instructions to yourself, and conversational tone. Do NOT paste the original prompt.>
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
