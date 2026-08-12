# Manta — Agent Guide

## What we build now

**Manta Pup demo only.**

| Path | Role |
|------|------|
| `manta-pup/` | Implementation home (Fyne + `issues/*.md`) |
| `manta-doc/` | Docs + tasks (`tasks/pup/`) |

Do not implement elsewhere. When the product moves past demo, the user will edit `CLAUDE.md` / this file.

## Workflow

1. Tasks: `manta-doc/tasks/pup/{todo,in-progress,done}/`
2. Code: only under `manta-pup/`
3. Commit in the owning git repo (`manta-pup` or `manta-doc`)

## Commands

```bash
cd manta-pup
go test ./...
go vet ./...
go build -o bin/manta-pup .
```

## Rules

- YAGNI; no-design Fyne widgets
- Files under the opened code repo's `issues/*.md` are source of truth
- SQLite is rebuildable index only
- Read `manta-pup/CLAUDE.md` before changing demo code
- Read `manta-doc/CLAUDE.md` before changing docs/tasks
