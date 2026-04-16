# CLAUDE.md

This file is a guide for Claude Code when working on this repository.

## Philosophy
> 상세: [manta-doc/Manifesto.md](manta-doc/Manifesto.md)

- **작업은 파일이다**: DB가 아닌 Markdown 파일로 존재
- **AI는 사용자다**: AI가 직접 조작할 수 있는 구조화된 인터페이스 제공
- **로컬-first**: 서버/클라우드 종속 없이 사용자가 데이터를 완전히 소유
- **단순함 우선**: 복잡한 워크플로우, 과도한 설정을 거부

## Commands

```bash
# Build all packages
npm run --workspace packages/core build && npm run --workspace packages/cli build

# Run CLI
node packages/cli/dist/index.js

# Dev mode (watch)
npm run --workspace packages/core dev
npm run --workspace packages/cli dev
```

## Architecture

### Project Structure
```
manta-repo/
├── packages/
│   ├── core/          # @manta/core — task CRUD, file I/O, state management
│   │   └── src/
│   └── cli/           # @manta/cli — CLI interface (commander)
│       └── src/
├── tsconfig.base.json # shared TS config
└── package.json       # npm workspaces root
```

### Key Decisions
- **TypeScript + CommonJS**: `module: node16`, source uses `import`, compiled output is CJS
- **Monorepo**: npm workspaces, `@manta/core` is shared by CLI and future Electron app
- **File-based tasks**: tasks are Markdown files with YAML frontmatter, not DB records
- **CLI-first**: CLI is the primary interface, GUI is a layer on top

---

## Code Design Principles

### YAGNI (You Aren't Gonna Need It)
- Do not write unused code. Remove it when found.
- Do not add code "just in case it's needed later."
- Unused return values, parameters, fields → remove.
- Uncalled functions, unused imports → delete.

### Naming (Critical)

> **If you don't know the exact domain context or business logic, ASK the user. Do not guess.**

#### Forbidden Patterns
```typescript
// BAD — too vague
const data = getData()
const result = process(items)
const info = fetchInfo()

// BAD — excessive abbreviation
const usr = getUser()
const calcAmt = calculateAmount()
```

#### Correct Patterns
```typescript
// GOOD — specific and clear
const taskFileContent = readTaskFile(taskId)
const filteredTasksByStatus = filterTasksByStatus(tasks, 'done')
const parsedFrontmatter = parseYamlFrontmatter(rawContent)
```

#### Naming Checklist
1. Can you understand what it is just by reading the name?
2. Does it use domain terminology? (`data` → `taskFileContent`)
3. Does it describe the action specifically? (`process()` → `parseAndValidateTaskFile()`)
4. Is singular/plural clear? (list → `tasks`, single → `task`)

---

## Testing

### Principles
- Test naming: `describe('[Feature]')` + `it('should [expected behavior]')`
- Follow Given-When-Then flow
- Mock external APIs and third-party services only
- Time-dependent tests: use fake timers

### Test Naming
```typescript
describe('TaskFileParser', () => {
  it('should parse valid frontmatter and return task object', () => {})
  it('should throw error when frontmatter is missing', () => {})
})
```
