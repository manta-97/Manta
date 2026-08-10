# Phase 1 개발 결과 보고 — Local Linear + Root SQLite v0 + GUI v0

> **ARCHIVED (TS/Electron era).** 2026-08부터 스택은 Go + Wails로 재구현한다.
> 이 문서는 이전 구현의 의사결정·교훈 참고용이다. 현재 로드맵 완료 상태가 아니다.
> CLI 오류의 `[CODE]` 접두·세분 코드 등 일부 계약은 폐기되었고, 현재 기준은
> [cli-design.md](cli-design.md)의 정석 exit 0/1/2·사람용 stderr다.
>
> 2026-06-10 · manta-repo 브랜치 `phase1-local-linear`

## 1. 어디서 출발했나

이 작업 전의 manta-repo는 이런 상태였다.

- `@manta/core`: `init`과 프로젝트 레지스트리만 존재. task를 다루는 코드는 없음
- `@manta/cli`: `init`, `help` 두 명령. 경로별로 제각각인 에러 처리
- `@manta/desktop`: Electron 스캐폴딩만. preload가 비어 있어 core와 미연결
- 글로벌 데이터가 `~/Library/Application Support/manta`에 — macOS 하드코딩
- Manifesto v2가 말하는 `.manta/project.json` anchor와 `~/.manta/manta.sqlite` 엔진은 미구현

즉, "작업을 파일로 정의하고 AI와 사람이 같은 규칙으로 조작한다"는 선언은 있었지만
정작 **작업(task)을 만들 수도, 옮길 수도 없는** 시스템이었다.

이번 작업은 Manifesto v2와 cli-design.md를 계약으로 삼아 그 간극을 닫았다.
이미 기획돼 있던 task-11(오류 정책)과 task-13(폴더 상태 모델)을 먼저 적용하고,
그 위에 task-14~19와 GUI v0를 쌓았다.

## 2. 무엇이 만들어졌나

### 패키지 구조

```
packages/
├── core/      @manta/core    — 파일 계약의 소유자. 런타임 의존성 0
├── engine/    @manta/engine  — root SQLite 운영 엔진 (better-sqlite3) [신규]
├── cli/       @manta/cli     — commander 기반 CLI. core+engine의 adapter
└── desktop/   @manta/desktop — Electron Local Workspace. core의 adapter
```

의존 방향은 한쪽으로만 흐른다: `cli → engine → core`, `desktop → core`.
core는 누구도 모른다. 파일 계약을 core가 소유하므로 CLI와 GUI의 동작이 갈라질 수 없다.

### 파일 계약 (core)

```
my-project/
├── .manta/
│   └── project.json          # anchor: projectId, schemaVersion, createdAt, taskDir
└── manta/                    # 사용자 선택 경로 (anchor의 taskDir)
    └── tasks/
        ├── todo/             # 폴더 = 상태. frontmatter에 status 없음
        ├── in-progress/
        └── done/
```

- task 파일: `task-N.md`, frontmatter는 `id`/`title`/`created` 3필드 (수제 파서 — YAML 라이브러리 없음)
- 상태 전환 = `mv`, 상태 질의 = `ls`. AI가 CLI 없이도 시스템을 일관되게 다룰 수 있다
- 채번: 세 폴더 전체 스캔 max+1, 갭 재사용 없음
- 글로벌 데이터는 `~/.manta/`(크로스 플랫폼)로 이동. `MANTA_HOME`으로 오버라이드 가능

### CLI — 10개 명령

```
init  add  list  show  start  done  edit  search  index  help
```

전 명령이 통일된 오류 정책 위에 있다: exit `0`(성공/no-op) / `1`(runtime) / `2`(usage),
stderr는 `[UNKNOWN_COMMAND]` / `[USAGE_ERROR]` / `[RUNTIME_FAILURE]` 한 줄 형식.
help registry가 source of truth이므로 구현된 명령만 help에 등장한다.

### Root SQLite 엔진 (engine)

`~/.manta/manta.sqlite`에 projects/tasks 인덱스. `manta index rebuild`는 전체 재생성,
`manta index check`는 해시 기반 불일치 진단. 프로젝트 폴더를 이동해도 anchor의
projectId로 같은 프로젝트를 재식별하고 `last_seen_path`를 갱신한다.
DB를 지워도 파일과 anchor에서 언제든 다시 만들어진다 — 원본은 항상 파일이다.

### Desktop — Local Workspace v0

main process가 core를 직접 호출하고(CLI shell 호출 없음), preload가
`window.manta` 타입드 API 5개만 노출한다. 화면은 프로젝트 사이드바 + 상태별
task 목록(+ 새 task 입력) + read-only Markdown 미리보기(+ Start/Done 버튼).
core의 Result가 그대로 renderer까지 흘러가 GUI도 CLI와 같은 오류 코드로 분기한다.

## 3. 핵심 설계 결정 다섯 가지

자세한 결정 기록은 각 태스크 문서(tasks/done/task-11~21)에 있다. 여기서는 방향을 정한 다섯 개만.

1. **anchor에 `taskDir`를 추가했다.** cli-design의 anchor 예시는 3필드였지만,
   `manta init docs`로 만든 프로젝트에서 하류 명령이 tasks 루트를 결정적으로
   찾으려면 anchor가 그 경로를 알아야 했다. "추측보다 명시"를 택했다.

2. **SQLite는 별도 패키지로 격리했다.** core의 런타임 무의존 원칙과 SQLite의
   네이티브 의존성은 양립할 수 없다. `@manta/engine`이 그 경계다. `node:sqlite`
   (의존성 0)는 Node 22.5+ 전용이라 보류 — Node 최소 버전을 올릴 때 재검토.

3. **인덱스는 증분이 아니라 전체 재생성.** 파일이 원본인 시스템에서 증분 동기화는
   "어느 쪽이 맞는가"라는 질문을 만든다. v0는 그 질문 자체를 없앴다.

4. **list는 깨진 파일에 관대하고, show는 엄격하다.** 목록은 한 파일 때문에 죽지 않고
   `(malformed task file)`로 표시한다. 단일 조회는 `TASK_FILE_MALFORMED`로 정확히
   실패한다. 같은 데이터라도 명령의 책임에 따라 실패 정책이 다르다.

5. **GUI 상태는 파일을 재조회한다.** add/move 성공 후 renderer는 낙관적 갱신 대신
   목록을 다시 읽는다. "GUI는 별도 source of truth를 만들지 않는다"를 코드로 지켰다.

## 4. Manifesto v2 체크리스트

| 신념 | 상태 |
|---|---|
| 3.1 작업은 파일이다 | ✅ task = Markdown 파일, Git 추적 가능 |
| 3.2 SQLite는 로컬 운영 엔진이다 | ✅ `~/.manta/manta.sqlite` + anchor 재연결, 삭제 후 재생성 가능 |
| 3.3 단순함 | ✅ 상태 3개, 전환은 mv, 설정 파일 없음 |
| 3.4 CLI는 canonical API | ✅ 10개 명령, GUI는 그 위가 아니라 같은 core 위의 동등한 adapter |
| 3.5 AI는 사용자다 | ✅ 결정적 exit code/stable error code, help --json, 파일 직접 편집 경로 안내 |
| 3.6 데이터는 로컬에 | ✅ 클라우드/서버 의존 0, `MANTA_HOME`까지 사용자 제어 |
| 3.7 작업은 규약이다 | ✅ 생성/전환/채번/오류 규칙이 core 한 곳에, 깨진 파일도 복구 단서 유지 |

## 5. 검증

- 자동 테스트 **177개** 통과 (core 단위 / cli 계약·통합 / engine)
  - CLI 계약 테스트는 stdout·stderr·exit code를 함께 검증한다
- task-12 수동 QA 12개 시나리오 전부 통과 (byte-level help 동등성, NO_COLOR, JSON 계약 포함)
  — 실행 기록은 tasks/done/task-12/qa.md
- CLI 전체 흐름 스모크: init → add → start → list → show → done → search →
  index rebuild/check → 폴더 이동 후 재연결까지 임시 디렉토리에서 실행 확인
- desktop: typecheck + electron-forge 기동 + renderer 번들 빌드 확인
  (화면 조작 QA는 미수행 — 남은 일 참고)

## 6. 남은 일

- **task-20 `manta context` v0** — 다음 태스크. 계약은 todo/task-20에 정리해 둠
- desktop 화면 조작 수동 QA (앱 기동까지만 검증됨)
- desktop task 편집 + command palette (Phase 4 후속)
- 오류 JSON 포맷 (task-11에서 의도적으로 제외)
- 기존 `~/Library/Application Support/manta/projects.json` → `~/.manta` 수동 정리
  (출시 전 / 사용자 0명 전제로 자동 마이그레이션은 만들지 않았다 — task-13과 같은 논리)
