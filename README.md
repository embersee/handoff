# claude-handoff

A Claude Code slash command + skill for shipping a task as a PR from a clean, isolated git worktree.

## What it does

You give Claude a task (bug report, tester feedback, feature ask). Claude:

1. Creates a fresh worktree based on the latest `origin/dev` (or repo default).
2. Implements the change in isolation — your main working tree stays untouched.
3. Opens a PR via `gh`.
4. Replies with only the PR URL + worktree path.

No summaries. No co-author tags. No force-push escape hatches.

## Install (local, manual)

```bash
git clone git@github.com:<you>/claude-handoff.git ~/code/claude-handoff
ln -s ~/code/claude-handoff/commands/handoff.md ~/.claude/commands/handoff.md
ln -s ~/code/claude-handoff/skills/handoff ~/.claude/skills/handoff
```

Now `/handoff <task>` is available in any Claude Code session, and the skill auto-triggers when the request matches.

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

- Branch: `philip/<slug>` — change in `commands/handoff.md` to your taste.
- Base: `dev` if it exists, else repo default.
- Worktree: `../<repo>-<slug>` (sibling dir).
- Commits: Conventional Commits.
- PR: ready for review (not draft) by default.

## Roadmap

- Plugin packaging (so it's installable via `/plugin marketplace add`).
- Auto-cleanup of merged worktrees.
- Resume flow when handing off follow-up feedback to an existing PR.
