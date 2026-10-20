# Deployment

Status: Current-verified (from workflow and config files, 2026-10-20); whether the workflows run is TBD | Owner: TBD

ASpectra is never deployed. Each application repository owns its own build, artifacts and
deployment. This page records what the repository files define. It does not claim that any
workflow is enabled or running: `.claude/CLAUDE.md` says "GitHub Actions is off" (issue C3).

## Ownership (from files)

| Repository | Workflows | Artifacts | Deployment |
|---|---|---|---|
| qa-platformCore | `ci.yml`, `i18n-translate.yml`, `publish.yml` (on `v*` tags) | Maven artifacts `com.qa:*-core` | None to production |
| qa-platformBE | `ci.yml`, `db-backup-restore-scripts.yml`, `deploy.yml` | BE and FE container images in GitHub Container Registry | `deploy.yml` "Deploy - Production": on push to `master` and `v*.*.*` tags, deploys to a VPS |
| qa-platformFE | `ci.yml`, `e2e.yml` | Built by BE's `deploy.yml` at the commit in `fe.ref` | No workflow of its own |

## Independent releases

- **qa-platformBE** can be released on its own, with Core and FE at the pinned refs.
- **qa-platformCore** publishes artifacts on its own. They reach production only after BE updates
  `coreVersion` and `core.ref` and deploys.
- **qa-platformFE** is built and tested on its own. As defined today, it reaches production only
  through BE's `fe.ref`.

## Rollback ownership (Intended)

| What | Owner | Method |
|---|---|---|
| Production deployment | qa-platformBE | Redeploy the previous BE release with its `core.ref` and `fe.ref` |
| Core artifacts | qa-platformCore | Never re-tag; BE pins the previous version |
| FE | qa-platformBE | Revert `fe.ref` |
| Database schema | TBD | Stated per release record |

## To be determined

Answer these before any deployment automation is added to ASpectra:

1. Is GitHub Actions enabled for each repository, and is the VPS deployment live?
2. Which environments exist besides production, and who owns each?
3. Release versioning for BE and FE: tag format, who tags, relation to `coreVersion`.
4. Is deploying FE through BE's `fe.ref` intended long-term?
5. Database migration rollback policy.
6. Who owns the deployment secrets and where they are stored.
7. Image retention and rollback window in the container registry.
8. Who approves a production release and records it.

See [release-coordination.md](release-coordination.md) for coordinated releases.
