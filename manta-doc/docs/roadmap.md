# 로드맵

Manta는 AI를 많이 쓰는 1인 빌더를 위한 local-first 작업 운영 도구다.

## 현재 초점 (2026-08): Fyne dogfood 데모

> **Phase 1~7 (CLI-first + Wails) 구현은 일시 정지.**  
> 상세 합의: [demo-fyne-jira-local.md](demo-fyne-jira-local.md)

CLI/Wails를 완벽하게 쌓기 전에, **파일 형식을 실사용으로 발견**한다.

| 항목 | 내용 |
|------|------|
| UI | Fyne, no-design |
| 원본 | 코드 레포 안 `issues/*.md` (Markdown + frontmatter) |
| SQLite | 재빌드 가능한 인덱스만 |
| MVP | 이슈 CRUD, 목록/검색+status 필터, Jira REST로 특정 티켓 일회 import |
| 구현 홈 | 형제 폴더 **`manta-pup/`** (독립 go.mod / git) |
| 태스크 큐 | **`manta-doc/tasks/pup/`** (본선 `tasks/{todo,…}` 와 분리) |
| 동결 | `manta-repo` (`cmd/manta` / Wails) 확장 중지 |
| 종료 | MVP+실사용 후, 형식 고통이 반복될 때 계약 freeze (작성자 판단) |

장기 spine(아래 Phase 1~7)은 폐기한 것이 아니라 **데모 종료 후 재평가**한다.
Manifesto의 파일 원본·local-first·AI-usable 방향은 유지하고, **전달 순서만** 데모로 검증한다.

---

## 장기 spine (데모 후 재평가)

```
Local Linear
→ Root SQLite
→ AI Context
→ Local GUI
→ Lightweight History
→ Import/Export
→ Pro Layer
```

## Stack rewrite (2026-08)

TypeScript monorepo + Electron 구현은 폐기했다.
Go 모듈로 재구현하는 방향은 유지한다.

| 이전 (폐기) | Go rewrite (동결 중) | 현재 데모 |
|---|---|---|
| TypeScript / npm workspaces | Go module | Go module (동일) |
| Electron | Wails (`desktop/`) | **Fyne** (`manta-pup/`) |
| CLI-first Phase 1 | `cmd/manta` + cobra | CLI 동결, GUI-first dogfood |
| better-sqlite3 | `internal/engine` | 레포 로컬 인덱스만 (최소) |

Wails 전환 기록: [stack-go-wails.md](stack-go-wails.md)  
데모 결정: [demo-fyne-jira-local.md](demo-fyne-jira-local.md)

TS 시대 완료 보고는 참고용으로 남긴다:

- [phase-1-dev-report.md](phase-1-dev-report.md) (archived — TS/Electron)
- [phase-3-7-dev-report.md](phase-3-7-dev-report.md) (archived — TS/Electron)

---

## Phase 1: Local Linear

> ⏸ 데모 중 일시 정지. 재개 시 파일 계약은 데모 결과를 반영해 수정할 수 있다.

로컬 파일 기반 작업 관리의 핵심을 구현한다.
목표는 “가벼운 Linear”처럼 매일 쓸 수 있는 CLI다.

### 명령어

- `manta init [path]` — Manta 디렉토리 초기화
- `manta add "제목"` — 작업 생성
- `manta list` — 작업 목록
- `manta show <id>` — 작업 상세
- `manta start <id>` — 작업 시작
- `manta done <id>` — 작업 완료
- `manta edit <id>` — 작업 수정
- `manta search <query>` — 작업 검색
- `manta help [command]` — 도움말

### 완료 기준

- 작업을 파일로 생성/조회/수정/상태 변경할 수 있다.
- 상태는 `tasks/{todo,in-progress,done}/` 폴더 위치로 결정된다.
- `status` frontmatter는 사용하지 않는다.
- title과 Markdown body를 텍스트 검색할 수 있다.
- AI가 `manta help`로 명령어를 학습하고 사용할 수 있다.

### 구현 순서 (Go rewrite)

1. `task-1`: Go module scaffold + CLI entry — in progress (scaffold only)
2. `task-2`: 종료·출력 계약 (exit 0/1/2 정석, 사람용 stderr, stdout/stderr 분리)
3. `task-3`: 폴더 상태 모델 + project anchor
4. `task-4`: `manta init` / `manta help`
5. `task-5`: `manta add`
6. `task-6`: `manta list` / `manta show`
7. `task-7`: `manta start` / `manta done`
8. `task-8`: `manta edit`
9. `task-9`: `manta search`

---

## Phase 2: Root SQLite

> ⏸ 데모 중 일시 정지. 데모 구간 SQLite는 레포 로컬 인덱스만.

사용자 홈에 root SQLite를 둔다.
Markdown 파일이 source of truth이고, root SQLite는 빠른 조회와 제품 경험을 위한 로컬 작업 운영 엔진이다.

root DB 위치:

```text
~/.manta/manta.sqlite
```

각 프로젝트에는 root DB와 다시 연결하기 위한 작은 anchor만 둔다.

```text
<project>/.manta/project.json
```

폴더가 이동되어도 Manta는 `projectId`로 root DB의 기존 기록을 찾고 `last_seen_path`를 갱신한다.

### root DB 대상

- project id
- 마지막으로 확인한 프로젝트 경로
- task id
- title
- status
- created
- 파일 경로
- 파일 해시
- 검색용 title/body 텍스트
- 선택적 섹션 위치
- context 후보 추출용 ranking/cache
- 최근 조회/선택 상태 같은 로컬 UI 메타데이터

### 명령어 후보

- `manta index rebuild` — project anchor와 Markdown 파일을 스캔해 root DB를 다시 만든다
- `manta index check` — root DB의 경로, 해시, 파일 상태가 맞는지 확인한다

### 원칙

- 작업 본문, 상태, 결정, 결과처럼 복구 불가능한 사용자 데이터는 SQLite에만 두지 않는다.
- root SQLite는 삭제해도 project anchor와 Markdown에서 다시 만들 수 있어야 한다.
- root DB가 깨져도 작업 파일은 안전해야 한다.
- search, context, GUI는 root DB를 사용해 빨라질 수 있지만, 원본은 항상 파일이다.
- root SQLite 구현은 `internal/engine`에 둔다. `internal/core`에 무거운 의존성을 넣지 않는다.
- root DB는 기본적으로 Git 추적 대상이 아니다.

### 완료 기준

- 프로젝트 anchor와 Markdown task 폴더를 스캔해 root DB를 만들 수 있다.
- 프로젝트 폴더가 이동되면 `projectId`로 같은 프로젝트를 식별하고 `last_seen_path`를 갱신할 수 있다.
- 파일 변경 후 root DB를 재생성할 수 있다.
- root DB가 없어도 사용자가 원본 task 파일을 잃지 않는다.
- Manta를 쓰면 단순 Markdown 폴더보다 검색, GUI, context 조립이 빨라진다.

---

## Phase 3: AI Context

> ⏸ 데모 중 일시 정지.

완료된 작업과 진행 중인 작업을 미래 AI 세션의 입력으로 다시 꺼낼 수 있게 한다.

### 명령어

- `manta context <task-id...>` — 여러 작업을 AI용 Markdown context로 출력
- `manta context <task-id...> --max-chars 6000` — 문자 수 제한
- `manta context <task-id> --for pr-review` — PR 리뷰 목적 출력 (후순위)

### v0 원칙

- 모델을 호출하지 않는다.
- 네트워크를 사용하지 않는다.
- 로컬 task 파일만 읽는다.
- deterministic extractive output을 만든다.
- 하나라도 task 조회에 실패하면 전체 실패한다.
- partial context는 출력하지 않는다.

### 완료 기준

- 과거 완료 작업을 10초 안에 AI context로 재사용할 수 있다.
- context 출력은 Codex/Claude에 바로 붙여넣기 좋다.
- `--max-chars`로 길이를 제한할 수 있다.

---

## Phase 4: Local GUI (Wails)

> ⏸ 데모 중 일시 정지. 1차 dogfood UI는 Fyne ([demo-fyne-jira-local.md](demo-fyne-jira-local.md)).
> 데모 후 Wails 재개·폐기·병행은 미정.

GUI는 무료 제품의 일부다.
CLI-only 제품이 아니라 CLI-first 제품으로 간다.
데스크톱 셸은 **Wails**다 (Electron 아님).

### 방향

- 왼쪽: todo / in-progress / done 작업 목록
- 가운데: 선택한 task Markdown editor
- 전역: command palette
- AI context는 상시 패널이 아니라 on-demand action

### 원칙

- GUI는 `internal/core` 위의 adapter다 (Wails Go binding).
- GUI는 root SQLite를 표시/검색용 운영 계층으로 사용할 수 있다.
- GUI는 CLI를 shell로 호출하지 않는다.
- GUI는 CLI stdout/stderr를 파싱하지 않는다.
- GUI는 별도 source of truth를 만들지 않는다.
- preview는 read-only다.
- 명시적 save action 없이는 파일을 변경하지 않는다.

---

## Phase 5: Lightweight History

> ⏸ 데모 중 일시 정지.

작업 파일 안의 선택적 섹션을 점진적으로 표준화한다.
처음부터 무거운 handoff packet을 강제하지 않는다.

선택적 섹션:

```markdown
## Intent
## Notes
## Decisions
## Result
```

### 원칙

- 섹션은 있으면 활용하고, 없어도 동작해야 한다.
- 일반 문서 관리용 `manta doc` 명령은 보류한다.
- source/reference 문서가 task 밖의 독립 제품 표면을 필요로 한다는 증거가 생기기 전까지 Docs phase는 만들지 않는다.

---

## Phase 6: Import / Export

> 데모 중에는 **Jira REST 단방향·특정 키·최소 필드 import만** 한다.
> 아래 번들/멀티 커넥터 설계는 일시 정지.

Manta가 local source of truth라는 점이 분명해진 뒤 외부 시스템과 연결한다.

후보:

- Jira export
- Notion import/reference
- GitHub PR summary export

원칙:

- Jira/Notion/GitHub는 원본이 아니라 bridge다.
- import/export는 Manta의 로컬 작업 기록을 흐리면 안 된다.

v0 목표: 커넥터들이 공통으로 딛고 설 `manta-tasks` JSON 번들 포맷(v1)과
`manta export`/`manta import <file>` round-trip을 먼저 고정한다.
외부 커넥터는 이 번들의 변환기로 후속 구현한다.

---

## Phase 7: Pro Layer — 방향 기획 ([pro-layer-plan.md](pro-layer-plan.md))

> ⏸ 데모 중 일시 정지.

무료 제품은 한 대의 로컬 머신에서 완결된 느낌이어야 한다.
Pro는 여러 기기, 깊은 검색, 고급 context intelligence, 외부 시스템 bridge가 필요해지는 순간부터 시작한다.

무료:

- CLI
- local file contract
- local task CRUD/search
- basic `manta context`
- Local Workspace GUI (Wails)
- Markdown task editor
- local context preview
- Git-friendly backup

Pro 후보:

- encrypted continuity sync
- optional user-owned backup target
- semantic search
- advanced context compiler
- import/export connectors
- team/shared workspace later

Sync 원칙:

- 로컬 파일이 source of truth다.
- 클라우드는 원본 DB가 아니라 연속성 계층이다.
- 가능하면 암호화된 sync와 사용자 소유 백업 대상을 우선한다.
