# Workspace setup

Status: Intended | Owner: TBD

How to recreate the A·Spectra workspace on a new machine. Toolchains and build steps for each
application are in that repository's own README.

## 1. Clone the governance repository

```bash
git clone https://github.com/avenero/ASpectra.git ASpectra
cd ASpectra
```

## 2. Clone the application repositories inside it

The directory names must match the ignored paths in `.gitignore` exactly.

```bash
git clone https://github.com/avenero/qa-platformCore.git qa-platformCore
git clone https://github.com/avenero/qa-platformBE.git   qa-platformBE
git clone https://github.com/avenero/qa-platformFE.git   qa-platformFE
```

## 3. Verify the boundary

```bash
git status --porcelain --ignored | grep qa-platform   # each must show as "!!" (ignored)
for r in qa-platformCore qa-platformBE qa-platformFE; do git -C "$r" rev-parse --show-toplevel; done
```

Each `rev-parse` prints the child path, not the ASpectra root.

## 4. Create local-only files

Create these by hand. They are ignored and must never be committed. Ask the project owner for
the values; they are not documented anywhere in Git.

| File | Purpose |
|---|---|
| `.env` | Workspace-level environment values |
| `.claude/settings.local.json` | Personal Claude Code permissions and settings |
| Each repository's `.env` | See that repository's `.env.example` and README |

## 5. Per-repository setup

Follow the README of each repository:
[qa-platformCore](https://github.com/avenero/qa-platformCore),
[qa-platformBE](https://github.com/avenero/qa-platformBE),
[qa-platformFE](https://github.com/avenero/qa-platformFE).
