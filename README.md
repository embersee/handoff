# handoff

A Claude Code skill (plus a `/handoff-clean` companion command) for shipping a task as a PR from a clean, isolated git worktree.

## What it does

You give Claude a task (bug report, tester feedback, feature ask). Claude:

1. Creates a fresh worktree based on the latest `origin/dev` (or repo default).
2. Implements the change in isolation — your main working tree stays untouched.
3. Opens a PR via `gh`.
4. Replies with only the PR URL + worktree path.

No summaries. No co-author tags. No force-push escape hatches.

## Install (local, manual)

```bash
git clone git@github.com:<you>/handoff.git ~/code/handoff
ln -s ~/code/handoff/skills/handoff ~/.claude/skills/handoff
ln -s ~/code/handoff/commands/handoff-clean.md ~/.claude/commands/handoff-clean.md
```

The skill auto-triggers when you ask Claude to ship something as a PR; you can also invoke it explicitly as `/handoff <task>`. The companion `/handoff-clean` command runs the aggressive worktree sweep on demand.

## Usage

```
/handoff fix the login button not being clickable on mobile, repro on iOS Safari
```

Claude returns:
```
https://github.com/you/repo/pull/123
worktree: /Users/you/code/repo-login-button-mobile
```

## Conventions (current)

- Branch: `handoff/<slug>` — change in `skills/handoff/SKILL.md` to your taste.
- Base: the currently checked-out branch by default. Override by saying "handoff from main" / "off of release/v2" / etc. in the task.
- Worktree: `../<repo>-<slug>` (sibling dir).
- Commits: Conventional Commits.
- PR: ready for review (not draft) by default.

## Cleanup

Worktrees stick around until you remove them. Two mechanisms:

- **Auto-sweep on `/handoff`** — at the start of every handoff, the agent removes `handoff/*` worktrees whose PR is **merged or closed**. Conservative: open PRs, no-PR branches, dirty trees, and unpushed commits are left alone.
- **`/handoff-clean`** — aggressive. Removes worktrees whose PR is merged/closed **and** worktrees whose `handoff/*` branch has no PR at all. Still skips dirty / unpushed / open-PR worktrees.

Run from the main worktree, not from inside a handoff worktree.

## Roadmap

- Plugin packaging (so it's installable via `/plugin marketplace add`).
- Resume flow when handing off follow-up feedback to an existing PR.
- Optional `/schedule`'d weekly background sweep.
