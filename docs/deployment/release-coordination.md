# Release coordination

Status: Intended | Owner: TBD

A generic workflow for releases that span repositories. It is a proposal, not a description of
current practice. ASpectra provides only this procedure and the release records; it runs no
automation.

1. **Impact**: list affected repositories and the dependency path
   ([cross-repository-changes.md](../development/cross-repository-changes.md)).
2. **Contracts**: agree the contract changes and compatibility expectations with every consumer.
3. **Implement and test** in each owning repository, through its own git flow.
4. **Review** the resulting commits and release identifiers: Core version, BE tag, FE commit.
5. **Validate** compatibility together: the BE gate runs against the pinned Core and FE.
6. **Deploy in dependency order**: Core artifacts, then BE updates `coreVersion`, `core.ref` and
   `fe.ref`, then the BE release. Confirm this order against real practice before marking this
   page Current-verified.
7. **Record** the release in [releases/](releases/TEMPLATE.md).
8. **Rollback**: name the rollback owner and constraints in the record.

A machine-readable release manifest is not added until releases are frequent enough to need it
and the questions in [README.md](README.md#to-be-determined) are answered.
