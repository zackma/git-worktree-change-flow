---
name: git-worktree-change-flow
description: This skill should be used when making code changes that need git isolation and a commit→merge→push lifecycle — especially in background/isolated sessions, or when the user asks to "make this change in a worktree", "commit and merge to main", "push it", or work on an isolated branch. Covers the full loop: create worktree, edit, validate, commit with co-author trailer, merge --no-ff to the default branch, push with retry on network failure, then clean up the worktree.
version: 1.0.0
---

# Git Worktree Change Flow

A repeatable lifecycle for making an isolated code change and landing it on the
default branch. Use this whenever a change should be developed off the main
checkout (background-session isolation, risky edits, parallel work) and then
merged + pushed cleanly.

## When to use

- The harness requires background sessions to isolate changes in a worktree.
- The user asks to work "in a worktree", or to "commit, merge to main, and push".
- You want edits staged on a branch before touching the default branch.

## The loop

### 1. Create the worktree
Use `EnterWorktree` with a short, descriptive name (e.g. `hide-ports`,
`add-caddy`, `fix-volume-prune`). All subsequent edits MUST use the worktree
path, not the original checkout — editing the shared checkout in a background
session is blocked.

### 2. Edit in the worktree path
Read each file from the worktree path before editing (the harness tracks file
state per path). Mirror existing conventions in the repo.

### 3. Validate before committing
Run the cheapest correctness check available for what changed:
- Shell scripts: `bash -n script.sh`
- Docker Compose: `docker-compose -f ./docker-compose.yml config` (parses +
  shows the merged result; grep for `published:` to confirm port exposure)
- Go: `go build ./...`; Node: `yarn build` / `yarn lint`
Never commit without at least a syntax/parse check.

### 4. Commit with a co-author trailer
Use a clear subject + body explaining *why*. End the message with:
```
Co-Authored-By: <model> <noreply@anthropic.com>
```
Only `git add` the files you intentionally changed — check `git status --short`
first so stray files (e.g. `.claude/`) don't get committed.

### 5. Merge to the default branch with --no-ff
Run the merge from the original checkout path via `git -C <repo-root>`:
```bash
git -C <repo-root> merge --no-ff <worktree-branch> -m "Merge branch '<branch>': <summary>

Co-Authored-By: <model> <noreply@anthropic.com>"
```
`--no-ff` keeps the change as a reviewable merge commit.

### 6. Push — with retry on network failure
```bash
git -C <repo-root> push origin <default-branch>
```
Network timeouts (`Failed to connect to github.com port 443`, `Recv failure`)
are environmental, NOT code problems. On failure:
- Run the retry as a **background** task (`run_in_background: true`) and wait on
  it — pushes can take 60–90s before timing out.
- Distinguish clearly for the user: the change is safe on local `master`; only
  the push to remote failed. Offer to retry or let them push behind a proxy/VPN.
- Do NOT keep hammering synchronously; report state and retry deliberately.

### 7. Clean up the worktree
Only after push succeeds. `ExitWorktree` with `action: remove`. The guard will
warn that the worktree has 1 commit — that commit is already merged + pushed, so
`discard_changes: true` is safe (you're discarding only the redundant branch
pointer, not the work). Confirm this reasoning to the user.

## Reporting

End with a compact status table: change committed / merged to master / pushed to
origin (with the ref range, e.g. `9fd6a6c..19617c5`) / worktree cleaned up. If
the change affects a deployed server, remind the user it's on GitHub but the
server still needs `git pull` to receive it.

## Pitfalls

- **Editing the shared checkout in a background session is blocked** — always
  enter the worktree first.
- **Empty `git add` / wrong path** — run edits and reads against the worktree
  path, run merge/push against the original repo root via `git -C`.
- **Push failure ≠ merge failure** — a failed push leaves local master ahead of
  origin; the work is not lost.
