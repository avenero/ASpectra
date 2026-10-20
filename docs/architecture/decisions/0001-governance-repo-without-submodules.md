# 0001 Governance repository with nested, ignored application repositories

Status: Accepted
Date: 2026-10-20
Deciders: Abel (project owner)

## Context
The workspace root held cross-repository documentation and agent configuration but was not a
Git repository. qa-platformFE, qa-platformBE and qa-platformCore are independent repositories
on GitHub, cloned inside the root. They must keep their histories, remotes and release flows.

A parent Git repository sees each nested repository as an untracked directory. Adding one
records a gitlink (mode 160000), which behaves like a submodule without `.gitmodules`.

## Decision
The workspace root becomes the ASpectra Git repository, branch `main`. The three application
repositories stay where they are and are ignored by explicit path in the root `.gitignore`.
ASpectra references them by URL in [REPOSITORIES.md](../../../REPOSITORIES.md) and
[config/repository-map.yaml](../../../config/repository-map.yaml). Files are staged by explicit
path only.

## Alternatives
- Git submodules: pins commits in the parent and adds a second update workflow for every change.
  Rejected; the repositories release independently.
- Monorepo: merges histories and release flows. Rejected; out of scope and irreversible.
- Moving the application repositories next to ASpectra instead of inside it: works, but breaks
  existing relative paths (`../.claude`, worktree locations) and gives no extra safety once the
  ignores are in place. Not needed now.

## Consequences
- The first and every later ASpectra commit contains only ASpectra files.
- A new machine clones ASpectra first, then clones the three repositories into it by name
  ([workspace-setup.md](../../development/workspace-setup.md)).
- The statement in `.claude/CLAUDE.md` that the root is not a Git repository is now outdated
  (issue C2 in [docs/README.md](../../README.md#known-open-documentation-issues)).
