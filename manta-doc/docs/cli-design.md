# CLI 설계

## 개요

Manta CLI는 파일 기반 작업 시스템의 표준 인터페이스다.
사람, 스크립트, AI 에이전트가 모두 같은 명령어와 같은 파일 계약으로 작업을 조작한다.

CLI는 유일한 제품 표면이 아니라 `@manta/core` 위에 올라가는 adapter다.
향후 GUI도 같은 core와 같은 로컬 파일 계약을 사용해야 한다.

```
          ┌─────────────┐
CLI   ──▶ │             │
          │ @manta/core │ ──▶ local files
GUI   ──▶ │             │
          └─────────────┘
                  │
                  ▼
          root SQLite engine
```

원칙:

- CLI와 GUI는 `@manta/core` 위의 동등한 adapter다.
- GUI는 CLI를 shell로 호출하거나 stdout/stderr를 파싱하지 않는다.
- 로컬 파일 계약은 CLI가 아니라 `@manta/core`가 소유한다.
- `@manta/core`는 런타임 의존성을 추가하지 않는다.
- root SQLite 구현은 `@manta/core`의 무의존 원칙을 깨지 않도록 별도 모듈이나 adapter 경계에서 다룬다.

---

## Root SQLite 운영 계층

Markdown 파일은 source of truth다.
root SQLite는 원본 DB가 아니라 사용자 홈에 있는 로컬 작업 운영 엔진이다.
여러 Manta 프로젝트를 빠르게 찾고, 검색하고, GUI와 context 기능을 빠르게 만들기 위한 계층이다.

목적:

- `manta list`를 빠르게 만든다.
- `manta search`를 빠르게 만든다.
- GUI 목록/필터/검색을 빠르게 만든다.
- `manta context`가 관련 task와 섹션을 빠르게 찾게 한다.
- 여러 프로젝트의 최근 작업과 검색 결과를 한 곳에서 다룰 수 있게 한다.
- 단순 Markdown 폴더보다 Manta 프로그램을 쓸 이유를 만든다.

root DB 위치:

```text
~/.manta/manta.sqlite
```

프로젝트 anchor 위치:

```text
<project>/.manta/project.json
```

`project.json`은 root DB와 프로젝트 폴더를 다시 연결하기 위한 최소 정보만 가진다.

```json
{
  "projectId": "manta_proj_abc123",
  "schemaVersion": 1,
  "createdAt": "2026-05-03"
}
```

원칙:

- root SQLite는 삭제해도 project anchor와 Markdown task 파일에서 다시 만들 수 있어야 한다.
- 작업 본문, 상태, 결정, 결과처럼 복구 불가능한 사용자 데이터는 SQLite에만 두지 않는다.
- Git으로 추적할 기본 자산은 Markdown task 파일이다.
- `.manta/project.json`은 프로젝트 식별 anchor이고, `~/.manta/manta.sqlite`는 기본적으로 Git 추적 대상이 아니다.
- root DB가 없거나 깨졌을 때는 `manta index rebuild`로 복구한다.
- root DB는 성능과 제품 경험을 위한 계층이지, 소유권의 원본이 아니다.

초기 root DB 대상:

| 필드 | 설명 |
|---|---|
| `project_id` | `.manta/project.json`의 프로젝트 영구 ID |
| `last_seen_path` | 마지막으로 확인한 프로젝트 경로 |
| `id` | task id |
| `title` | frontmatter title |
| `status` | 폴더 위치에서 계산한 상태 |
| `created` | frontmatter created |
| `path` | task 파일 경로 |
| `hash` | 파일 변경 감지용 해시 |
| `body_text` | 검색용 본문 텍스트 |
| `sections` | 선택적 섹션 위치 메타데이터 |
| `context_rank` | context 후보 추출용 ranking/cache |
| `ui_metadata` | 최근 조회/선택 상태 같은 로컬 UI 메타데이터 |

인덱스 명령 후보:

| 명령어 | 설명 |
|---|---|
| `manta index rebuild` | project anchor와 Markdown task 파일을 스캔해 root DB를 다시 만든다 |
| `manta index check` | root DB의 경로, 해시, 파일 상태가 맞는지 확인한다 |

프로젝트 폴더 이동 처리:

- Manta는 현재 경로에서 `.manta/project.json`을 찾는다.
- `projectId`로 root DB의 기존 프로젝트 record를 찾는다.
- `last_seen_path`가 현재 경로와 다르면 새 경로로 갱신한다.
- 필요하면 Markdown task 파일을 다시 스캔해 인덱스를 갱신한다.

---

## 디렉토리 구조

Manta는 사용자의 git repo 안에 일반 폴더로 존재한다.
작업 데이터는 숨기지 않는다. 사용자가 직접 열고, AI가 직접 읽고, Git이 추적할 수 있어야 한다.

```
my-project/
├── .manta/
│   └── project.json              # root DB와 연결되는 프로젝트 anchor
└── manta/                        # 사용자 선택 경로, 기본값: manta/
    └── tasks/
        ├── todo/
        │   ├── .gitkeep
        │   └── task-1.md
        ├── in-progress/
        │   ├── .gitkeep
        │   └── task-2.md
        └── done/
            ├── .gitkeep
            └── task-3.md
```

설계 근거:

- **숨기지 않는다**: 사용자가 직접 파일을 열고 편집할 수 있어야 한다.
- **위치 선택 가능**: `manta init [path]`로 경로를 지정한다.
- **1 repo = 1 Manta**: 별도 프로젝트 개념을 만들지 않는다.
- **상태는 폴더로 드러낸다**: 상태 질의는 파싱이 아니라 파일 시스템 탐색으로 끝나야 한다.
- **DB는 root에 둔다**: 무거운 SQLite는 `~/.manta/manta.sqlite`에서 관리하고, 프로젝트에는 작은 anchor만 둔다.

---

## 작업 상태 모델

상태는 frontmatter가 아니라 폴더 위치가 결정한다.

```
todo → in-progress → done
```

| 상태 | 경로 | 의미 |
|---|---|---|
| `todo` | `tasks/todo/` | 생성 직후 기본 상태 |
| `in-progress` | `tasks/in-progress/` | 진행 중 |
| `done` | `tasks/done/` | 완료 |

상태 전환:

- `manta start task-3` → `tasks/in-progress/task-3.md`로 이동
- `manta done task-3` → `tasks/done/task-3.md`로 이동

전환 규칙:

- 지정한 task가 없으면 실패한다.
- 이미 목표 상태에 있으면 no-op + 안내 메시지 + exit 0.
- 그 외 전환은 허용한다. 예: `done`에 있는 task도 `start`로 다시 `in-progress`가 될 수 있다.

---

## 작업 ID 체계

ID는 순번 기반이다.

```
task-1
task-2
task-3
```

파일명이 곧 기본 ID다.

```
tasks/todo/task-1.md
tasks/in-progress/task-2.md
tasks/done/task-3.md
```

채번 규칙:

- `todo/`, `in-progress/`, `done/` 전체에서 `task-(\d+).md`를 스캔한다.
- 다음 ID는 최대 숫자 + 1이다.
- 삭제된 ID는 재사용하지 않는다.

---

## 작업 파일 포맷

Phase 1에서는 가벼운 포맷을 유지한다.

```markdown
---
id: task-1
title: CLI 프로토타입 만들기
created: 2026-04-28
---

자유 본문.
```

규칙:

- `status` 필드는 두지 않는다. 상태는 폴더 위치가 source of truth다.
- `id`, `title`, `created`는 생성 시 기록한다.
- 본문은 자유 Markdown이다.
- Phase 1 검색은 title과 body 전체 텍스트를 대상으로 한다.

Phase 2 이후 `manta context`는 아래 섹션이 있으면 활용할 수 있다. 단, 필수는 아니다.

```markdown
## Intent
## Notes
## Decisions
## Result
```

---

## 오류 분류

task 조회 공통 오류는 구분해야 한다.
AI가 직접 CLI를 쓰려면 실패 신호가 정확해야 한다.

| 코드 | 의미 |
|---|---|
| `INVALID_TASK_ID` | task id 형식이 잘못됨 |
| `TASK_NOT_FOUND` | 형식은 맞지만 task가 없음 |
| `DUPLICATE_TASK_ID` | 여러 상태 폴더에 같은 id가 존재함 |
| `TASK_FILE_UNREADABLE` | 파일 권한 등으로 읽을 수 없음 |
| `TASK_FILE_MALFORMED` | task 파일 구조가 깨짐 |

규칙:

- 모든 조회 실패를 `TASK_NOT_FOUND`나 `UNKNOWN`으로 뭉개지 않는다.
- 중복 ID는 데이터 무결성 오류다.
- CLI stderr는 사람이 읽을 수 있어야 한다.
- error code는 테스트, AI 분기, GUI 표시를 위한 안정 식별자다.

---

## 명령어

### Phase 1: Local Linear

| 명령어 | 설명 |
|---|---|
| `manta init [path]` | Manta 디렉토리 초기화 |
| `manta add "제목"` | `tasks/todo/task-N.md` 생성 |
| `manta list` | 상태별 작업 목록 보기 |
| `manta show <id>` | 작업 상세 보기 |
| `manta start <id>` | 작업을 `in-progress`로 이동 |
| `manta done <id>` | 작업을 `done`으로 이동 |
| `manta edit <id>` | 작업 파일 수정 |
| `manta search <query>` | 작업 title/body 텍스트 검색 |
| `manta help [command]` | 도움말 |

`manta search` v0:

```bash
manta search "auth"
manta search "migration"
manta search --status done "context"
```

규칙:

- v0는 단순 텍스트 검색이다.
- 검색 대상은 task title과 Markdown body다.
- `--status`는 폴더 기준으로 필터링한다.
- 결과가 없어도 exit code는 0이다.
- 결과 없음 메시지는 명확해야 한다.

예:

```text
No tasks matched "oauth".
No done tasks matched "oauth".
```

### Phase 2: Root SQLite

`manta index`는 project anchor와 로컬 Markdown task 파일에서 root SQLite를 만든다.
search, context, GUI는 이 root DB를 사용할 수 있다.

| 명령어 | 설명 |
|---|---|
| `manta index rebuild` | project anchor와 task 파일을 스캔해 root DB 재생성 |
| `manta index check` | root DB의 경로, 해시, 파일 상태 검증 |

### Phase 3: AI Context

`manta context`는 완료된 작업을 미래 AI 세션의 입력으로 다시 꺼내는 명령이다.

```bash
manta context task-1
manta context task-1 task-4 --max-chars 6000
manta context task-7 --for pr-review --max-chars 4000
```

v0 계약:

- 모델을 호출하지 않는다.
- 네트워크를 사용하지 않는다.
- 로컬 task 파일만 읽는다.
- deterministic extractive output을 만든다.
- `--max-chars`는 문자 수 기준이다.
- 하나라도 task 조회에 실패하면 전체 실패한다.
- partial context는 출력하지 않는다.

예:

```bash
manta context task-1 task-999 task-3
```

`task-999`가 없으면:

```text
exit code: 1
stderr: TASK_NOT_FOUND: task-999
stdout: 비어 있음
```

### Phase 4: Local GUI

GUI는 CLI/file contract 위에 올라가는 로컬 작업 공간이다.
GUI는 무료 제품의 일부다.

v0 방향:

```text
LEFT:   todo / in-progress / done task list
CENTER: selected task Markdown editor
GLOBAL: command palette for add / start / done / context
```

AI context는 상시 오른쪽 패널이 아니라 on-demand action이다.

- `Copy AI Context`
- modal
- drawer
- command palette action

규칙:

- preview는 read-only다.
- 명시적 save action 없이는 파일을 변경하지 않는다.
- GUI는 Jira식 관리자 대시보드가 아니다.

---

## help 명령어의 역할

`manta help`는 단순 도움말이 아니다.
AI가 Manta CLI를 처음 접했을 때 명령어 체계를 학습하는 진입점이다.

- `manta help`: 전체 명령어 목록 + 설명
- `manta help <command>`: 개별 명령어의 상세 사용법, 옵션, 예시

help 출력은 사람과 AI가 모두 읽을 수 있어야 한다.
