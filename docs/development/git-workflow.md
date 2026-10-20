# Git workflow

Status: Intended (moved from `.claude/CLAUDE.md` on 2026-10-20) | Owner: TBD

Git rules for the three application repositories. ASpectra itself uses a single `main` branch with explicit-path
staging ([R3](../architecture/repository-rules.md#r3-root-isolation)).

## Shared workspace rules (several agents work here at once)

**One session, one worktree, one branch.** Concurrent sessions never share a checkout. A session that
is about to work on a feature creates its own git worktree for that branch and works only there:

```
cd <repo> && git fetch origin && git worktree list
git worktree add ../<repo>.worktrees/<issue>-<kebab-slug> -b feature/<issue>-<kebab-slug> origin/develop
```

- The worktree directory is a sibling of the three checkouts, at workspace root:
  `<repo>.worktrees/<issue>-<kebab-slug>`. The slug matches the branch name, so the mapping from
  branch to directory is mechanical and any session can find it.
- Branch from `origin/develop` after a `git fetch`, never from a stale local `develop`.
- Run `git worktree list` first. If a worktree for that issue already exists, reuse it or stop and ask
  the user - never create a second one for the same branch (git refuses the same branch in two
  worktrees anyway).
- Every agent that edits files works **only inside its own worktree path**. Never edit the main
  checkout, and never edit another session's worktree, even when `git worktree list` reveals it.
- The main checkout stays on `develop` and is left alone. If it is on someone else's branch, has
  uncommitted work, or holds `develop` with unpushed merges, stop and ask the user. Never route around it.
- Stage by explicit path, never `git add -A` or `git add .`: another agent's files may be in the tree.
- Never use a bare `git stash` or `git stash pop`; **the stash is shared across every worktree of the
  same repository**, so a worktree does not isolate it. Use a WIP commit on your own branch instead.
- Cleanup belongs to `gitFlow` and happens only after the merge is confirmed on the remote: it removes
  the worktree with `git worktree remove <path>` (never `rm -rf`, which leaves stale metadata in
  `.git/worktrees/`) and deletes the feature branch. It then removes the `<repo>.worktrees/` directory
  itself with `rmdir`, which succeeds only once that directory is empty. That directory is shared by
  every session of the repository, so a refusal means another session still has a worktree in it: it is
  reported and left alone, never emptied by force.

**What a worktree does not isolate.** These are single shared resources that live outside any
worktree, so they serialize across all sessions no matter how many worktrees exist:

- `~/.m2` and `qa-platformBE/lib/core-m2` - only one Core publish may run at a time, serialized by the
  `.core-publish.lock` lock at workspace root. When several sessions changed Core, the last one to finish
  publishes once for all of them instead of each publishing its own.
- `master` - one promotion at a time, and only when the user asks for it literally.
- Fixed ports (FE dev server 3000, Compose services, Testcontainers): two sessions running a stack at
  once need different ports.
- In the three application repositories, `.claude/CLAUDE.md` and `.claude/settings.json` are committed;
  `.claude/settings.local.json` and everything else under `.claude/` are not. ASpectra commits its own
  `.claude/` except local settings ([ADR 0002](../architecture/decisions/0002-single-source-for-agent-instructions.md)).
- GitHub Actions runs CI; every gate also runs locally before a push. Testcontainers on Rancher Desktop needs
  `TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock`. Run the BE gate as
  `env -u SPRING_PROFILES_ACTIVE ./gradlew clean check -Pstrict`, and never run a targeted Gradle test in
  the same checkout while it runs.
- Agent instructions are loaded as described in [agent-workflow.md](agent-workflow.md). Rules live once, in
  the canonical documents; tool files only link to them.

## Git flow rules and Classification Criteria 

This epigraph defines how Claude Code must handle GitHub work depending on whether the task is a new feature or a simple task/change.

***
Branch strategy (always applies for local and remote branches)

A feature/task branch is never merged directly into master.
The flow is always: feature/<branch> → develop → master.
develop is the integration branch: all feature branches are created from develop and merged back into develop.
**A task ends at develop.** Promotion of develop to master is a separate step that runs only when the
user asks for it literally. It is never part of a feature flow, never implied by a green gate, and never
triggered because develop looks stable.

Each session works in its own worktree on its own branch (see "Shared workspace rules" above), so
several sessions can run this flow at the same time without touching each other's state.

***
Flow for "New Feature"

1. Create an issue (if it doesn't exist yet)
gh issue create --title "Feature: <name>" --body "<description, acceptance criteria>"
This keeps traceability and links the PR to the issue.

2. Create a dedicated worktree and branch from origin/develop
Naming convention, used for both the branch and its worktree directory:
feature/<issue-number>-<short-kebab-case-description>
Example: feature/142-user-authentication -> qa-platformBE.worktrees/142-user-authentication

cd <repo>
git fetch origin
git worktree list                      # never create a second worktree for the same branch
git worktree add ../<repo>.worktrees/142-user-authentication -b feature/142-user-authentication origin/develop
cd ../<repo>.worktrees/142-user-authentication

Every later step in this flow runs inside that worktree. The main checkout is left on develop, untouched.

3. Iterative development with atomic commits
Commits must follow Conventional Commits:
feat: add OAuth login
test: add authentication tests
docs: update README with new feature
Every commit must build and pass tests locally (never commit broken code)

4. Tests are mandatory before moving on
Before considering the feature done, ensure that:
Write unit tests for the new functionality
Run the full test suite (not just the new tests)
Confirm nothing existing was broken
Compile locally to verify the feature works as expected and without error or warnings
If any issues or warnings are found, fix them before proceeding to the next step.

5. Run core-publisher - only when Core changed
`core-publisher` is called when `qa-platformCore` changed in this task, or to repair an empty or stale
`lib/core-m2`. A task that did not touch Core has nothing to publish and does not call it at all.
When it is called, it runs **before** the merge: its purpose is to prove BE compiles against the Core
artifacts on disk before anything reaches develop.
Only one publish runs at a time across all sessions, because `~/.m2` and `lib/core-m2` live outside every
worktree: it takes the `.core-publish.lock` lock at workspace root.
**Several sessions working on Core publish once, not once each.** The session that finishes last takes the
batch: gitFlow merges every finished Core branch into local `develop`, core-publisher publishes that merged
state once, the BE gate runs against it, and only then is `develop` pushed. A branch joins the batch only if
it is pushed, its gate passed, its session reported finished, and no live worktree still holds it; anything
else is left out and named in the report. A session that finds another still working on Core releases the
lock and lets the last one publish - it never waits holding the lock.

6. Merge the feature branch into develop (gitFlow does this, not you by hand)
Re-fetch first: another session may have merged into develop while this task ran.
git fetch origin
If develop moved, merge origin/develop into the feature branch inside the worktree and re-run the full
gate before continuing. A gate that was green against an older develop proves nothing.
Then merge the feature branch into develop in the main checkout and push:
git -C <main-checkout> checkout develop
git -C <main-checkout> pull origin develop
git -C <main-checkout> merge --no-ff feature/142-user-authentication
git -C <main-checkout> push origin develop

7. Delete the worktree and the branch after the merge is confirmed on the remote
git fetch origin && git log origin/develop --oneline -1      # confirm the merge landed
git -C <main-checkout> worktree remove ../<repo>.worktrees/142-user-authentication
git -C <main-checkout> branch -d feature/142-user-authentication
Never `rm -rf` a worktree directory: that leaves stale metadata in `.git/worktrees/`. Never delete
either one before the push is confirmed.
For a batch merge, each auxiliary branch is deleted only after proving `develop` contains all of its
commits - `git log --oneline <branch> --not develop` must print nothing - and `git branch -d` (never `-D`)
is the second safety net. Remove only your own worktrees: another session's worktree is its own to clean up.

8. Stop here
Do not promote develop to master. That is its own task, on an explicit request from the user, and it
runs the mandatory `security-guardian` gate first.