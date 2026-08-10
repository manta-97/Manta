# CLI 설계

## 개요

Manta CLI는 파일 기반 작업 시스템의 표준 인터페이스다.
사람, 스크립트, AI 에이전트가 모두 같은 명령어와 같은 파일 계약으로 작업을 조작한다.

CLI는 유일한 제품 표면이 아니라 `internal/core` 위에 올라가는 adapter다.
향후 GUI(Wails)도 같은 core와 같은 로컬 파일 계약을 사용해야 한다.

```
          ┌────────────────┐
CLI   ──▶ │                │
          │ internal/core  │ ──▶ local files
GUI   ──▶ │                │
(Wails)   └────────────────┘
                  │
                  ▼
          internal/engine (root SQLite)
```

원칙:

- CLI와 GUI는 `internal/core` 위의 동등한 adapter다.
- GUI는 CLI를 shell로 호출하거나 stdout/stderr를 파싱하지 않는다 (Wails binding으로 core 호출).
- 로컬 파일 계약은 CLI가 아니라 `internal/core`가 소유한다.
- `internal/core`는 가능한 한 stdlib 위주이며 무거운 런타임 의존성을 추가하지 않는다.
- root SQLite 구현은 `internal/engine` adapter 경계에 둔다.

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

`~/.manta/`에는 root DB와 프로젝트 레지스트리(`projects.json`)가 산다.
`MANTA_HOME` 환경변수로 위치를 바꿀 수 있다 — 테스트 격리와 샌드박스 환경용 탈출구다.

프로젝트 anchor 위치:

```text
<project>/.manta/project.json
```

`project.json`은 root DB와 프로젝트 폴더를 다시 연결하기 위한 최소 정보만 가진다.

```json
{
  "projectId": "manta_proj_abc123",
  "schemaVersion": 1,
  "createdAt": "2026-05-03",
  "taskDir": "manta"
}
```

`taskDir`는 프로젝트 루트 기준 task 디렉토리 상대 경로다. `manta init docs`처럼
비기본 경로로 초기화한 프로젝트에서도 하류 명령이 tasks 루트를 결정적으로 찾기 위해
anchor에 둔다. anchor가 깨졌을 때는 기본값 `manta`로 동작한다 — anchor 손상이
작업 파일 접근을 막아서는 안 된다.

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

## CLI 출력·종료 계약 (정석 정렬)

Unix/GNU 관례와 12-factor CLI에 맞춘다.
공개 계약은 **exit code + 사람용 메시지 + stdout/stderr 분리**다.
기계용 `[CODE]` 접두 프로토콜이나 sysexits(64+)는 쓰지 않는다.

### Exit code

| exit | 의미 | 예 |
|---:|---|---|
| `0` | 성공 | 생성·조회·상태 변경 성공, 검색 0건, idempotent no-op |
| `1` | 실행 실패 | task 없음, id 형식 오류, 파일 I/O, 데이터 깨짐, index 불일치 |
| `2` | 사용법 오류 | unknown command, 잘못된 옵션, 인자 개수 부족/과다 |

규칙:

- 스크립트·AI의 1차 분기는 exit code다.
- **CLI 사용법 문제만 2**, 그 외 실패는 1로 단순화한다.
- 이미 목표 상태인 `start`/`done`, 이미 초기화된 `init` 등은 **idempotent 성공(exit 0)** + 짧은 안내.
- 검색·목록이 비어 있는 것은 실패가 아니다 (exit 0).

### stdout / stderr

| 스트림 | 용도 |
|---|---|
| stdout | 성공 시의 결과 데이터 (목록, 본문, export JSON, context 문서 등) |
| stderr | 오류, 경고, (선택) 짧은 상태·안내 메시지 |

규칙:

- 실패 시 **결과 데이터를 stdout에 쓰지 않는다** (파이프·리다이렉트 오염 방지).
- 경고는 stderr. 성공 데이터와 섞지 않는다.
- TTY가 아닐 때 색/스피너 등은 내지 않는다 (해당 기능을 넣을 때).

### 오류 메시지 (사람 우선)

stderr 예:

```text
Error: unknown command "xyz"
Error: accepts 1 arg(s), received 0
Error: task not found: task-999
Error: invalid task id: abc

Run 'manta --help' or 'manta <command> --help' for usage.
```

규칙:

- 문장으로 원인을 말한다. `[USAGE_ERROR]` 같은 대괄호 코드 접두는 쓰지 않는다.
- 가능하면 다음에 할 일(help 안내 등)을 한 줄 덧붙인다.
- 내부 Go 에러 타입으로 원인을 구분해도 된다. 다만 그 이름을 **외부 파싱 프로토콜로 고정하지 않는다**.
- 머신 친화 확장이 필요하면 나중에 `--json` 등으로 선택 제공한다.

### 내부 실패 구분 (구현·테스트용, 공개 프로토콜 아님)

core/cli 구현에서는 원인을 나눠 다루는 것이 좋다. 예:

- task id 형식 오류
- task 없음
- 동일 id가 여러 상태 폴더에 존재
- 파일 읽기/쓰기 실패
- task 파일 구조 깨짐

이 구분은 **더 나은 메시지와 테스트**를 위한 것이다.
CLI 사용자 계약은 여전히 exit `0|1|2`와 위 stderr 문장이다.

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
| `manta edit <id>` | `$VISUAL`/`$EDITOR`로 작업 파일 열기 (없으면 파일 경로 안내 후 실패) |
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
| `manta index check` | root DB의 경로, 해시, 파일 상태 검증. 불일치 시 issue 목록 + exit 1 |

구현 노트: SQLite는 `internal/core`의 가벼운 경계를 지키기 위해 `internal/engine`에 산다.
순수 Go 드라이버(`modernc.org/sqlite`)를 우선한다. rebuild는 증분이 아니라 전체 재생성이다 —
파일이 원본이므로 DB가 의심스러우면 다시 만드는 것이 가장 단순한 복구다.
`manta index`는 실행 전 현재 프로젝트를 projectId 기준으로 재등록하므로
폴더 이동 후에도 같은 프로젝트로 재연결된다.

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
stderr: Error: task not found: task-999
stdout: (결과 데이터 없음)
```

v0 계약 노트:

- stderr는 사람용 문장이다 (위 오류 계약 참고).
- `--for` 목적 프리셋은 v0에서 제외 (실사용 증거가 생기면 재검토).
- 절단은 헤더(제목/메타) 우선 보존 + task별 예산 균등 배분 + 섹션 우선순위
  선별(Result > Decisions > Intent > 기타 > Notes, 원문 순서 유지)로 동작한다.
- 출력 길이는 어떤 경우에도 `--max-chars`를 넘지 않는다.

### Phase 4: Local GUI (Wails)

GUI는 CLI/file contract 위에 올라가는 로컬 작업 공간이다.
스택은 **Wails**다 (Electron 폐기). GUI는 무료 제품의 일부다.

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
- Wails Go binding으로 `internal/core`를 직접 호출한다. CLI shell 호출·stdout 파싱 금지.
- Copy AI Context는 CLI와 같은 core 함수를 공유한다.

### Phase 6: Import / Export

| 명령어 | 설명 |
|---|---|
| `manta export` | 모든 task를 `manta-tasks` JSON 번들(v1)로 stdout에 출력 |
| `manta import <file>` | 번들의 task들을 새 id로 재채번하여 가져온다 (`task-3 → task-12` 매핑 출력) |

규칙:

- export stdout은 순수 JSON이다. 깨진 task는 stderr 경고로만 보고하고 exit 0.
- import는 쓰기 전에 번들 전체를 검증한다 — 부분 import는 없다. 실패 시 exit 1 + 사람용 메시지.
- 원본 id는 보존하지 않는다. 대상 프로젝트의 id 단조 증가 규칙과 충돌 방지가 우선이다.
- Jira/Notion/GitHub 커넥터는 이 번들 포맷의 변환기로 후속 구현한다.

---

## help · version

정석 CLI처럼 help와 version을 바로 찾을 수 있게 한다.

```bash
manta --help
manta -h
manta <command> --help
manta help              # 선택 별칭 (git 스타일)
manta help <command>
manta --version
```

규칙:

- 구현된 명령만 노출한다 (만들지 않은 명령을 help에 올리지 않는다).
- `Use` / `Short` / `Long` / `Example`을 명령마다 채운다 (cobra 기준).
- help는 사람·AI 모두가 읽고 올바른 호출을 만들 수 있어야 한다.
- “바이트 단위 이중 경로 SoT”를 1차부터 강제하지 않는다.
  `--help`와 `help`가 같은 내용을 가리키면 충분하다.
- 나중에 AI용 구조화 help가 필요하면 `manta help --json` 같은 선택 경로로 확장한다.
