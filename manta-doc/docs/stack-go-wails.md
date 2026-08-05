# Stack: Go + Wails

> 2026-08 스택 전환 기록. TypeScript monorepo + Electron을 폐기하고
> Go + Wails로 Phase 1부터 재구현한다.

## 왜 바꿨는가

TS 스택으로 Phase 1~6 v0를 한 바퀴 돌렸다. 제품 계약(파일 상태 모델,
CLI 표면, root SQLite 역할, context all-or-nothing)은 검증됐다.
남은 문제는 계약이 아니라 **구현 스택**이다.

- Electron은 배포 크기·네이티브 의존성·유지 비용이 크다.
- better-sqlite3 같은 native Node 모듈은 패키징 경계를 복잡하게 만든다.
- CLI + 로컬 GUI + 단일 바이너리 배포에는 Go가 더 단순하다.
- Wails는 Go backend에 web frontend를 얹되, Electron보다 가벼운 셸을 준다.

철학(파일 원본, AI-usable CLI, local-first)은 그대로다.
스택만 교체한다.

## 매핑

| 이전 | 이후 |
|---|---|
| `packages/core` (`@manta/core`) | `internal/core` |
| `packages/engine` (`@manta/engine`, better-sqlite3) | `internal/engine` (pure Go SQLite 우선) |
| `packages/cli` (`@manta/cli`, commander) | `cmd/manta` + `internal/cli` (cobra) |
| `packages/desktop` (Electron + Vite + React) | `desktop/` (Wails v2) |
| npm workspaces monorepo | 단일 `go.mod` |

## 유지하는 것

- 디렉토리/파일 계약 (`tasks/{todo,in-progress,done}/`, frontmatter 3필드)
- CLI 명령 표면과 오류 정책 (exit 0/1/2, `[CODE] message`)
- root SQLite 위치와 역할 (`~/.manta/manta.sqlite`, rebuildable index)
- context all-or-nothing, import/export 번들 v1 계약
- CLI와 GUI가 같은 core를 쓰는 adapter 구조

## 폐기하는 것

- TypeScript 패키지, Jest, ESLint monorepo 설정
- Electron Forge / preload / IPC 패턴
- `@manta/*` npm 패키지 이름

## 구현 원칙

1. **파일 계약은 `internal/core` 소유** — CLI/GUI가 갈라지지 않게
2. **GUI는 Wails binding** — CLI shell 호출 금지
3. **SQLite는 `internal/engine`** — core에 CGo/무거운 의존 금지 우선
4. **YAGNI** — TS 시절 기능을 한 번에 포팅하지 않고 Phase 순서대로 재구축
