# Manta Codex Guide

## Workspace shape
- This workspace has multiple independent trees:
  - `Manta/` is the harness repo for shared agent settings (and may track docs).
  - `manta-fyne/` is the **active experiment** code repo (Fyne + file-based issues).
  - `manta-repo/` is the frozen mainline code repo (Go + Wails/CLI) — do not expand during the experiment.
  - `manta-doc/` is the docs and task tree.
- Run git commands in the repository that owns the files you changed (`manta-fyne`, `manta-repo`, or harness root).

## Required context
- Read [CLAUDE.md](/Users/ytlee/Manta/CLAUDE.md) before changing root harness files.
- Read [CLAUDE.md](/Users/ytlee/Manta/manta-fyne/CLAUDE.md) before changing files in `manta-fyne/`.
- Read [CLAUDE.md](/Users/ytlee/Manta/manta-repo/CLAUDE.md) before changing files in `manta-repo/` (frozen).
- Read [CLAUDE.md](/Users/ytlee/Manta/manta-doc/CLAUDE.md) before changing files in `manta-doc/`.

## Stack
- **Active:** Go module in `manta-fyne/` (Fyne experiment).
  See `manta-doc/docs/experiment-fyne-jira-local.md`.
- **Frozen:** `manta-repo/` (Wails, CLI) — re-evaluate after the experiment.
- Do not reintroduce TypeScript packages, npm workspaces, or Electron
- SQLite is rebuildable index only during the experiment (files remain source of truth)

## Workflow
- Prefer task-based work when `manta-doc/tasks/` has an active task.
- During the Fyne experiment, **implement in `manta-fyne/`**, not `manta-repo/`.
- Review and commit in the owning repository (`manta-fyne` for experiment code).
- Small bug fixes and tightly scoped refactors may skip `impl.md`.

## Behavior rules
- Stay inside the requested scope. Do not make adjacent refactors or opportunistic cleanup unless the user asked for them.
- Keep explanations short and actionable.
- Read the minimum set of files needed before editing.
- If domain naming or business intent is unclear, ask instead of guessing.

## Code rules for `manta-fyne`
- Follow YAGNI. Do not add code for hypothetical future use.
- Use specific domain names. Avoid vague names like `data`, `result`, or `info`.
- Files under the opened code repo's `issues/*.md` are source of truth; SQLite is rebuildable index only.
- Prefer:
  - `go build -o bin/manta-fyne .` (or `./cmd/...` once laid out)
  - `go test ./...`
  - `go vet ./...`

## Code rules for `manta-repo` (frozen)
- Do not expand unless the user explicitly unfreezes mainline work.
- Keep `internal/core` free of heavy runtime dependencies if touching it.
