# Cross-repository changes

Status: Intended | Owner: TBD

Procedure for a change that touches more than one application repository. The git flow, worktree
rules and Core publishing steps are defined in [.claude/CLAUDE.md](../../.claude/CLAUDE.md) and are
not repeated here.

## 1. Impact statement

Write this before any code changes ([R4](../architecture/repository-rules.md#r4-impact-statement)):

```markdown
Change: <one sentence>
Affected repositories: <Core | BE | FE>
Dependency path: <for example Core -> BE -> FE>
Contracts touched: <Core consumer surface | mobile-agent wire | REST/STOMP | Zod | step patterns | none>
Breaking: <yes/no; if yes, approval reference>
Issues: <one per affected repository>
```

## 2. Implement in each owning repository

One issue, branch and worktree per repository, each through its own git flow
([R2](../architecture/repository-rules.md#r2-change-location)).

## 3. Merge order

Merge in dependency order. For Core changes consumed by BE, follow the Core publishing steps in
`.claude/CLAUDE.md` before the BE merge, and update `coreVersion` and `core.ref` in BE.

## 4. Record

If the change is released, add a record under
[deployment/releases/](../deployment/releases/TEMPLATE.md).
