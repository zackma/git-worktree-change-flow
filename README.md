# git-worktree-change-flow

A reusable [Claude Code](https://claude.com/claude-code) skill that standardizes
making an isolated code change and landing it on the default branch.

## What it does

Encodes a repeatable git lifecycle for changes that should be developed off the
main checkout (background-session isolation, risky edits, parallel work):

1. **Create** a git worktree with a descriptive name
2. **Edit** in the worktree path
3. **Validate** with the cheapest correctness check (`bash -n`, `docker-compose config`, `go build`, `yarn build`…)
4. **Commit** with a clear message and a `Co-Authored-By` trailer
5. **Merge** to the default branch with `--no-ff`
6. **Push** — with deliberate retry on network failure (timeouts are environmental, not code bugs)
7. **Clean up** the worktree once the push succeeds

## When it triggers

When making code changes that need git isolation and a commit→merge→push
lifecycle — or when you ask to "make this change in a worktree", "commit and
merge to main", or "push it".

## Install

Copy `SKILL.md` into a skill directory Claude Code reads:

```bash
mkdir -p ~/.claude/skills/git-worktree-change-flow
cp SKILL.md ~/.claude/skills/git-worktree-change-flow/
```

The skill is model-invoked: Claude activates it automatically when the task
matches the description above.
