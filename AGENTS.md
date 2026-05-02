# Manta Codex Guide

## Workspace shape
- This workspace has three independent git repositories:
  - `Manta/` is the harness repo for shared agent settings.
  - `manta-repo/` is the code repo.
  - `manta-doc/` is the docs and task repo.
- The root git repository does not own `manta-repo/` or `manta-doc/`. Run git commands in the repository that owns the files you changed.

## Required context
- Read [CLAUDE.md](/Users/ytlee/Manta/CLAUDE.md) before changing root harness files.
- Read [CLAUDE.md](/Users/ytlee/Manta/manta-repo/CLAUDE.md) before changing files in `manta-repo/`.
- Read [CLAUDE.md](/Users/ytlee/Manta/manta-doc/CLAUDE.md) before changing files in `manta-doc/`.

## Workflow
- Prefer task-based work:
  - Check the next task in `manta-doc/tasks/todo/`.
  - Write `impl.md` for non-trivial work when the task flow calls for it.
  - Implement in `manta-repo/`.
  - Review and commit in the owning repository.
  - Move task state in `manta-doc/` when the work is complete.
- Small bug fixes and tightly scoped refactors may skip `impl.md`.

## Behavior rules
- Stay inside the requested scope. Do not make adjacent refactors or opportunistic cleanup unless the user asked for them.
- Keep explanations short and actionable.
- Read the minimum set of files needed before editing.
- If domain naming or business intent is unclear, ask instead of guessing.

## Code rules for `manta-repo`
- Follow YAGNI. Do not add code for hypothetical future use.
- Use specific domain names. Avoid vague names like `data`, `result`, or `info`.
- Keep `@manta/core` free of runtime dependencies.
- Use the existing repository commands when verification is needed:
  - `npm run build`
  - `npm test`
  - `npm run lint`

