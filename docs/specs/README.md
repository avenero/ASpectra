# ASpectra — feature specifications

Each feature of the implementation baseline (`constitution.md`) is specified in its own folder `specs/<nnn>-<name>/` holding three documents:

| Document | Role | Contains |
|---|---|---|
| `spec<Name>.md` | WHAT and WHY | Context, user stories, functional requirements RF-x.y in EARS notation, non-functional requirements RNF-n, limit cases LC-n, out of scope, completion criteria, the decisions taken during specification, and the register items the feature depends on. No technology. |
| `plan<Name>.md` | HOW | Constitution compliance map, module structure, behaviour-to-module trace, technical decisions with discarded alternatives, test strategy, register dependencies, coverage check. |
| `tasks<Name>.md` | DO | Ordered tasks of 20–30 minutes with the requirements each covers and a verifiable "Done when" line, plus a coverage index. |

Questions a specification cannot answer from the design are never left open in these documents. A question that depends on an architecture or design decision is recorded in decision register `122:2` §14 (A-nn) or in the Screen decisions ledger of D03 `148:4` (S-nn) together with the assumption the specification builds until it is resolved; a question that depends on neither is decided with the product owner and recorded in the specification's decisions table (§1.4). While a specification still has open items it lists them in a §8; once every item is recorded or decided, §8 is removed and the register items the feature relies on are named in §1.4 and in its last completion criterion, and cited by identifier beside the criteria they affect. Specifications 001 and 002 keep their §8; 003 … 017 have closed theirs. Since 2 November 2026 every S-item is drawn and every register item is decided; 001 and 002 keep their §8 tables as the record of the assumptions they were built on, now pointing at the drawn nodes.

## Identity area (constitution Part II · IDN)

| # | Specification | Constitution features | Status |
|---|---|---|---|
| 001 | Login — sign-in, TOTP challenge, landing | F-IDN-03, F-IDN-04 | spec, plan and tasks written |
| 002 | Signup and email verification | F-IDN-01, F-IDN-02 | spec, plan and tasks written |
| 003 | TOTP enrollment, recovery codes and factor replacement | F-IDN-04 (enrollment), F-IDN-05, F-SET-03 | spec, plan and tasks written |
| 004 | Password reset and forced password change | F-IDN-06 | spec, plan and tasks written |
| 005 | Controlled account recovery | F-IDN-07 | spec, plan and tasks written |
| 006 | Sessions, refresh and expiry | F-IDN-08 | spec, plan and tasks written |
| 007 | Logout and session management | F-IDN-09 | spec, plan and tasks written |
| 008 | Invitation acceptance | F-IDN-10 | spec, plan and tasks written |
| 009 | Roles, grants and the authority matrix | F-IDN-11 | spec, plan and tasks written |
| 010 | Tenant members, invitations and ownership transfer | F-IDN-12, F-SET-04 | spec, plan and tasks written |
| 011 | Project grants editor | F-IDN-13 | spec, plan and tasks written |
| 012 | Workspace creation and bootstrap | F-IDN-14 | spec, plan and tasks written |
| 013 | Tenant chooser | F-IDN-15 | spec, plan and tasks written |
| 014 | Tenant suspension state | F-IDN-16 | spec, plan and tasks written |
| 015 | Leave tenant and identity erasure request | F-IDN-17 | spec, plan and tasks written |
| 016 | Guarded tenant and project switch | F-IDN-18 | spec, plan and tasks written |
| 017 | Public landing | F-IDN-19 | spec, plan and tasks written |

Specifications for the other areas (PRJ, CAT, AUT, APR, SET, OPS, RUN) continue the numbering from 018 when they are written.

## Conventions

- Requirement identifiers inside a specification are local (RF-1.1, RNF-3, LC-7); the FR identifiers of `requirements.md` are cited beside the criterion they support.
- A plan names only technologies fixed by constitution P10; test tools are flagged test-only; anything else is a register item.
- A task's "Done when" names the test from the plan's §5.2 that must pass, or a directly observable condition.
- Elements that one feature builds and another reuses are tagged **[001]**-style with the owning feature's number.
