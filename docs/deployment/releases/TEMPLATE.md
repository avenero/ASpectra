# Release record template

Status: Intended | Owner: TBD

Copy this file to `YYYY-MM-DD-<short-name>.md` in this folder for each coordinated release.

```markdown
# Release YYYY-MM-DD <short-name>

Status: Current-verified
Release owner: <name>
Approved by: <name, reference>

## Components
| Repository | Version or tag | Commit |
|---|---|---|
| qa-platformCore | <coreVersion> | <sha> |
| qa-platformBE | <tag> | <sha> |
| qa-platformFE | - | <sha, equals BE fe.ref> |

BE `core.ref`: <sha>
BE `fe.ref`: <sha>

## Contents
<issues and pull requests per repository>

## Compatibility
<contract changes and how consumers were validated>

## Database
<migrations included; reversible yes/no>

## Rollback
Owner: <name>
Procedure: <steps or link>
Constraints: <for example irreversible migrations>
```
